# Concetti — Piattaforma LISA

> **Categoria:** `LISA Survey / LISA2Next / Architettura / Infrastruttura`
> **Aggiornato:** `30/06/2026`

---

## Indice

**LISA Survey**

1. [Architettura a Microservizi](#1-architettura-a-microservizi)
2. [NGP Stack — Infrastruttura base](#2-ngp-stack--infrastruttura-base)
3. [Multi-Tenancy](#3-multi-tenancy)
4. [OAuth2 — Configurazione specifica LISA](#4-oauth2--configurazione-specifica-lisa)
5. [Riepilogo visivo — Stack LISA Survey](#5-riepilogo-visivo--stack-lisa-survey)
6. [Riepilogo visivo — Flusso richiesta Survey (Angular → Spring Boot)](#6-riepilogo-visivo--flusso-richiesta-survey-angular--spring-boot)

**LISA2Next**

7. [Architettura del progetto](#7-architettura-del-progetto)
8. [Il processo di build e deploy](#8-il-processo-di-build-e-deploy)
9. [Il Data Object (DO)](#9-il-data-object-do)
10. [L'annotazione @DOMetaData](#10-lannotazione-dometadata)
11. [Il meccanismo di persistenza — solo scrittura](#11-il-meccanismo-di-persistenza--solo-scrittura)
12. [La SELECT — completamente separata dalla persistenza](#12-la-select--completamente-separata-dalla-persistenza)
13. [Il GestioneContabilitaBean — il bean di business](#13-il-gestionecontabilitabean--il-bean-di-business)

---

# PARTE I — LISA Survey

## 1. Architettura a Microservizi

Per capire cosa significa "architettura a microservizi" è utile partire dal suo opposto: l'architettura **monolitica**. In un'applicazione monolitica classica, tutto il codice vive in un unico grande progetto. Il login, la gestione degli utenti, le logiche di business, il database, l'interfaccia — tutto è nello stesso posto, compilato insieme e deployato come un'unica unità. Questa soluzione funziona bene per applicazioni piccole, ma man mano che il sistema cresce, inizia a mostrare i suoi limiti: se vuoi aggiornare solo una piccola parte, devi ridistribuire tutto; se quella parte crasha, può portare giù l'intera applicazione; se un team vuole lavorare su una funzionalità, rischia di entrare in conflitto con il lavoro di un altro team.

L'architettura a microservizi nasce per risolvere esattamente questi problemi. L'idea centrale è semplice: invece di avere un'unica applicazione gigante, si divide il sistema in tanti piccoli servizi indipendenti, ciascuno responsabile di una cosa sola. Ogni servizio ha il proprio codice, il proprio database (se necessario), e comunica con gli altri tramite la rete — di solito attraverso chiamate HTTP o messaggi asincroni.

Nel contesto di LISA Survey, il sistema è suddiviso così:

- Un servizio gestisce solo l'**autenticazione** (il Modulo Base) — chi si può loggare, come vengono rilasciati i token, chi è autorizzato a fare cosa
- Un servizio gestisce il **registro dei microservizi** (Eureka) — tiene traccia di dove ogni servizio è in esecuzione
- Un servizio fa da **punto di ingresso unico** per tutte le richieste (API Gateway) — le instrada al servizio giusto
- Un servizio gestisce la **logica del Survey** — questionari, domande, risposte

Ogni servizio può essere avviato, fermato, aggiornato e scalato in modo completamente indipendente. Se c'è un bug nel servizio Survey, non tocchi il servizio di autenticazione. Se vuoi aggiornare il gateway, non devi toccare nient'altro.

Il rovescio della medaglia è la complessità operativa: per far funzionare anche una sola funzionalità devi avere **tutti i servizi dipendenti** in esecuzione. È questo il motivo per cui, prima di poter lavorare al Survey, hai dovuto avviare la Trinità: il Survey dipende dall'autenticazione (Modulo Base) per sapere chi è loggato, e dall'Eureka per sapere dove sono gli altri servizi.

---

## 2. NGP Stack — Infrastruttura base

L'infrastruttura base per sviluppare qualsiasi modulo LISA è il **NGP Stack**: un insieme di container Docker che devono essere sempre in esecuzione. A differenza della vecchia "Trinità" (un unico `trinity.yml`), ogni servizio ha ora il proprio file Docker Compose dedicato, con immagini aggiornate e allineate. I file si trovano in `C:\dev\NGP\ngp-stack\`.

---

### discoveryservice — Eureka

È il **registro centrale** dove tutti i microservizi si iscrivono all'avvio e comunicano la propria posizione sulla rete. Gira sulla porta **9201**. Tecnicamente è un'istanza di **Eureka Server**. Senza questo container, i microservizi non riescono a trovarsi tra loro. Definito in `discovery-service.yml`.

---

### apigateway — API Gateway

È il **punto di ingresso unico** per tutte le richieste esterne. Gira sulla porta **9102**. Quando il frontend fa una chiamata HTTP a un microservizio, non lo chiama direttamente — passa sempre attraverso il gateway, che sa dove instradare la richiesta. Questo rende il sistema più sicuro (un unico punto dove applicare autenticazione e controllo degli accessi) e più flessibile (si può spostare un microservizio su un'altra porta senza che il frontend lo sappia). Definito in `api-gateway.yml`.

---

### lisamodulobase — Modulo Base

È il **cuore dell'autenticazione e dell'autorizzazione** dell'intera piattaforma LISA. Gira sulla porta **9000**. Implementa un server OAuth2 completo: gestisce gli utenti, rilascia token JWT, verifica le credenziali. A differenza della vecchia trinity, questo modulobase usa **PostgreSQL locale** come database — non dipende da nessun server remoto. Definito in `modulobase-backend.yml`.

---

### nginx — Reverse proxy

Espone il modulobase all'URL `https://authorization-dev.lisasuite.it` gestendo SSL. Il browser e le applicazioni non parlano mai direttamente con la porta 9000 — passano sempre attraverso nginx, che smista le richieste al container giusto. Definito in `nginx.yml`.

---

### postgresql — Database del Modulo Base

Contiene tutti i dati del modulobase: utenti, client OAuth2 registrati, redirect URI, profili applicativi. Gira sulla porta **5432**. Definito in `postgresql.yml`.

---

### Struttura dei file e rete condivisa

Ogni file è un compose separato, ma tutti condividono la **stessa rete Docker esterna** chiamata `modulobase`. Questo è il meccanismo che permette ai container di vedersi tra loro anche se sono definiti in file diversi.

```
C:\dev\NGP\ngp-stack\
├── .env                    ← variabili lette da tutti i file
├── discovery-service.yml   ← discoveryservice  (porta 9201)
├── api-gateway.yml         ← apigateway        (porta 9102)
├── modulobase-backend.yml  ← lisamodulobase    (porta 9000)
├── nginx.yml               ← nginx             (porta 80/443)
└── postgresql.yml          ← postgresql        (porta 5432)
```

Il file `.env` contiene le variabili condivise:

```env
MODULOBASE_NETWORK=modulobase              # nome della rete Docker condivisa
VOLUMES_ROOT=C:/Dev/NGP/volumes            # root dei volumi persistenti
POSTGRES_PASSWORD=localpassword
MODULOBASE_ISSUER_URI_HOST=https://authorization-dev.lisasuite.it
```

**La chiave da capire sulle reti Docker**: i container sulla stessa rete si "vedono" usando il loro `container_name` come hostname. Per questo `api-gateway.yml` ha `EUREKA-HOST-NAME=discoveryservice` — `discoveryservice` è il nome del container Eureka sulla rete `modulobase`. Se mettessi `localhost` dentro un container, si riferirebbe al container stesso, non all'host fisico.

---

## 3. Multi-Tenancy

### Teoria

La **multi-tenancy** (multi-inquilinato) è un pattern architetturale dove una singola istanza dell'applicazione serve **più clienti distinti** (i "tenant"), mantenendo i loro dati completamente separati e isolati.

Nella piattaforma LISA, ogni azienda cliente è un tenant. Più aziende usano la stessa applicazione Survey, ma ogni azienda vede solo i propri questionari, i propri candidati, le proprie risposte.

I tre approcci principali per implementare la multi-tenancy:

| Approccio | Come funziona | Pro/Contro |
|---|---|---|
| **Database separato** | Ogni tenant ha il suo DB | Massimo isolamento, costoso |
| **Schema separato** | Stesso DB, schemi diversi | Buon isolamento, medio |
| **Tabella condivisa + discriminatore** | Stesso schema, colonna `company_id` | Economico, richiede disciplina nei filtri |

LISA Survey usa il terzo approccio: ogni tabella ha una colonna `company_id` (o `companyId`), e ogni query filtra per questo valore.

### TenantContext nel codice

```java
// Nel service, quando si crea una nuova risposta:
answer.setCompanyId(TenantContext.currentTenant());
// ↑ recupera il tenant dell'utente corrente dal contesto della richiesta
```

`TenantContext` è una classe che usa un `ThreadLocal` — una variabile che è locale al **thread** corrente (ogni richiesta HTTP è gestita da un thread separato). All'inizio di ogni richiesta, un filtro (`tenantDataFilter.setFilter()`) legge il header `X-Tenant` dalla richiesta HTTP e lo salva nel `ThreadLocal`. Da quel momento, qualsiasi codice eseguito nel contesto di quella richiesta può chiamare `TenantContext.currentTenant()` e ottenere il tenant corretto.

```java
// Nel Controller:
@PutMapping
public ResponseEntity<Void> setPickAnswer(
        @RequestHeader("X-Tenant") String tenant,   // ← letto dall'header HTTP
        ...) {
    tenantDataFilter.setFilter(); // ← salva il tenant nel ThreadLocal
    answerService.setPickAnswer(dto);
    return ResponseEntity.ok().build();
}
```

### X-Tenant — il header HTTP

Il frontend Angular include in ogni richiesta al backend l'header `X-Tenant` con l'identificatore dell'azienda dell'utente loggato. Questo header non fa parte dello standard HTTP (gli header che iniziano con `X-` sono custom), ma è la convenzione usata in LISA.

---

## 4. OAuth2 — Configurazione specifica LISA

### Due istanze di Modulo Base

Sul PC di sviluppo coesistono **due istanze diverse** del Modulo Base, per due contesti diversi:

In locale si usa esclusivamente il **Modulo Base NGP**:

| Container | Porta esposta | Database | URL pubblico |
|---|---|---|---|
| `lisamodulobase` | 9000 (dietro nginx) | PostgreSQL locale | `https://authorization-dev.lisasuite.it` |

Il profilo `local` del Survey punta a `https://authorization-dev.lisasuite.it` — nginx riceve la richiesta e la smista al container `lisamodulobase` sulla rete Docker `modulobase`. Gli utenti e i client OAuth2 (tra cui `angular-client`) vivono nel database PostgreSQL locale.

> La vecchia "trinity-modulobase" (che si connetteva a un SQL Server remoto su `10.204.204.114`) non viene più usata in locale — il team ha migrato tutto al NGP stack con immagini aggiornate e database locale.

### Tabella redirect_uri in LISA

Nel Modulo Base NGP, i redirect URI non si trovano nella tabella standard Spring `oauth2_registered_client`. Si trovano nella tabella personalizzata:

```
mb_registered_application_redirect_uri_set
```

Quando il login fallisce con errore 400 (redirect_uri non riconosciuto), è qui che si va ad aggiungere la URI mancante. Spring Authorization Server fa un confronto **esatto** sulla stringa — porta 4200 e porta 4201 sono diverse, `/callback` e assenza di `/callback` sono diversi.

### Configurazione Spring Security OAuth2 del Survey backend

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://authorization-dev.lisasuite.it/oauth2/jwks
          issuer-uri: https://authorization-dev.lisasuite.it
```

`jwks` (JSON Web Key Set) è l'endpoint del Modulo Base che espone le chiavi pubbliche usate per verificare la firma dei token. Il backend Survey scarica queste chiavi all'avvio e le usa per validare ogni token JWT ricevuto senza dover fare una chiamata di rete ad ogni richiesta.

---

## 5. Riepilogo visivo — Stack LISA Survey

```
BROWSER (localhost:4200)
    │ usa
    ▼
ANGULAR (framework frontend)
    │ costruito con
    ├── Angular CLI (strumenti sviluppo)
    └── DevExtreme UI (componenti grafici)
    │
    │ redirect per login → https://authorization-dev.lisasuite.it/login
    │ callback con codice → http://localhost:4200/callback
    │ poi HTTP + JWT token
    ▼
API GATEWAY — apigateway / NGP Stack (porta 9102)
    │ instrada le richieste consultando
    ▼
EUREKA — discoveryservice / NGP Stack (porta 9201)
    │ sa dove si trova
    ▼
SURVEY BACKEND — Spring Boot (porta 9090)
    │ usa
    ├── Spring Security OAuth2 (verifica il token JWT)
    ├── Spring Cloud Eureka Client (si registra su Eureka)
    ├── Swagger UI (documentazione API esposta su /swagger-ui)
    └── JPA/Hibernate (accesso al database)
    │
    ├── verifica token con ──► NGP MODULO BASE — authorization-dev.lisasuite.it
    │                               │ implementa
    │                               ├── OAuth2 Authorization Server
    │                               ├── Database: PostgreSQL locale (container postgresql)
    │                               └── Redirect URI registrati per angular-client
    │                                     incluso: http://localhost:4200/callback
    │
    └── legge/scrive su
    ▼
SQL SERVER — container Docker (porta 1433)
    │ dati persistiti su
    └── Volume Docker → C:\dev\NGP\volumes\mssql2019
```

---

## 6. Riepilogo visivo — Flusso richiesta Survey (Angular → Spring Boot)

```
FRONTEND Angular
┌─────────────────────────────────────────────────────┐
│  Model (.ts)                                        │
│  export interface PickAnswerDTO {                   │
│      quizId?: string; questionId?: string; ...      │
│  }                                                  │
│         ↓ usato da                                  │
│  Service Angular (answerApi.service.ts)             │
│  @Injectable — singleton per l'app                  │
│  public setPickAnswer(...): Observable<any> {       │
│      return this.httpClient.request('put', ...);    │
│  }                                                  │
│         ↓ usato da                                  │
│  Helper nel Componente                              │
│  async setPickAnswer(questionId, newChoiceId) {     │
│      const dto = { quizId: this.quizId, ... };      │
│      await new Promise((resolve, reject) => {       │
│          service.setPickAnswer(tenant, dto)         │
│              .subscribe({ next: resolve,            │
│                           error: reject });         │
│      });                                            │
│  }                                                  │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP PUT /api/survey/answer
                       │ Header: X-Tenant: <azienda>
                       │ Body: { quizId, questionId, newChoiceId }
                       ▼
BACKEND Spring Boot
┌─────────────────────────────────────────────────────┐
│  Controller (AnswerAPI.java)                        │
│  @PutMapping                                        │
│  public ResponseEntity<Void> setPickAnswer(         │
│      @RequestHeader("X-Tenant") String tenant,      │
│      @RequestBody PickAnswerDTO dto) {              │
│      tenantDataFilter.setFilter(); ← salva tenant  │
│      answerService.setPickAnswer(dto); ← delega    │
│      return ResponseEntity.ok().build();            │
│  }                                                  │
│         ↓ delega a                                  │
│  Service (AnswerService.java)                       │
│  @Transactional ← atomicità garantita              │
│  public void setPickAnswer(PickAnswerDTO dto) {     │
│      // cerca risposta esistente                    │
│      Optional<Answer> existing = answerRepo         │
│          .findByAttemptQuizIdAndQuestionId(...);    │
│      existing.ifPresent(answerRepo::delete); ← cancella
│      if (dto.getNewChoiceId() != null) {            │
│          Answer a = new Answer();                   │
│          a.setCompanyId(TenantContext.currentTenant());
│          answerRepo.save(a); ← inserisce           │
│      }                                             │
│  }                                                  │
│         ↓ usa                                       │
│  Repository (AnswerRepository.java)                 │
│  interface AnswerRepository                         │
│      extends JpaRepository<Answer, String> {       │
│      Optional<Answer> findByAttemptQuizId           │
│          AndQuestionId(String, String);            │
│  }         ↑ Spring genera il SQL automaticamente   │
│         ↓ Hibernate traduce in SQL                  │
│  Database SQL Server                                │
│  SELECT/DELETE/INSERT su tabella answers            │
└─────────────────────────────────────────────────────┘
```

---

# PARTE II — LISA2Next

## 7. Architettura del progetto

LISA2Next è strutturato come applicazione **Java EE** (Java Enterprise Edition) con un'architettura ibrida client-server. Il progetto è composto da due macro-aree con tecnologie e processi di build distinti.

```
LISA2Next/
├── SERVER (Maven)
│   ├── JavaCore            → libreria core condivisa (JAR).
│   ├── LISAServicesClient  → interfacce EJB (Enterprise JavaBeans) client (JAR)
│   ├── POJOEJBServices     → implementazioni bean di business (JAR)
│   ├── LISAServices        → modulo EJB (JAR)
│   ├── LISA2API            → API REST (WAR)
│   ├── LISA2Web            → frontend web / servlet (WAR)
│   ├── NETBridge           → bridge verso sistemi esterni (WAR)
│   └── LISA2               → aggregatore EAR (contiene tutto)
│
└── CLIENT (Ant / NetBeans)
    ├── GUIFramework        → libreria componenti UI custom
    ├── ScrollableDesktop   → libreria UI desktop
    └── LISAClient          → applicazione Swing desktop
```

### Composizione dettagliata

**JavaCore — Libreria core condivisa (JAR)**

È la fondamenta dell'intero sistema. Contiene tutto ciò che è trasversale e riutilizzato da tutti gli altri moduli.

Cosa contiene:
- I **Data Object** (DOProvvigioni, DOBudgetVendita, ecc.) — le classi POJO (Plain Old Java Object) che mappano le tabelle AS/400
- L'annotazione @DOMetaData e il sistema di persistenza (DataObject, DataWriter, SQLWriter, Query, Cursor)
- L'**ApplicationContext** — il contenitore che viaggia tra client e server in ogni chiamata
- Le classi di utilità comuni (Data per le date, MathUtility, ApplicationException, ecc.)
- Il client ActiveMQ per la messaggistica asincrona
- I client HTTP per chiamate verso sistemi esterni

Dipendenze esterne principali:
- jt400 — driver JDBC per AS/400
- activemq-broker — messaggistica
- httpclient — chiamate HTTP
- jackson / gson — JSON
- postgresql — driver alternativo per PostgreSQL

Dipende da: nessun altro modulo interno. È il modulo più "basso" della catena.

Deve essere buildato per primo perché tutti gli altri moduli lo importano.

---
**LISAServicesClient — Interfacce dei servizi EJB (JAR)**

Contiene solo interfacce Java — nessuna implementazione. Definisce il contratto che ogni servizio di business deve rispettare.

Cosa contiene:
- Oltre 100 interfacce come IGestioneContabilita, IGestioneClienti, IGestioneProduzione, IPersistor, IUserManagement, ecc.
- Ogni interfaccia dichiara i metodi che il servizio espone, ad esempio:

```java
public interface IGestioneContabilita {
    ApplicationContext elencoProvvigioni(ApplicationContext ctx);
    ApplicationContext salvaProvvigione(ApplicationContext ctx);
    ...
}
```

- Le interfacce sono annotate con @EndpointInterface(isEJB=true, lookup="java:global/LISA2/...") che indica il nome JNDI con cui il bean è registrato su WildFly

Perché esiste come modulo separato: il client Swing deve poter chiamare i servizi sul server senza avere in classpath l'implementazione completa. Importa solo questo JAR leggero con le interfacce, non i bean veri e propri.

Dipende da: JavaCore

---

**POJOEJBServices — Implementazioni POJO dei bean di business (JAR)**

Contiene le implementazioni concrete dei bean di business. È qui che vive la **logica applicativa vera**: accesso al database, costruzione delle query, orchestrazione delle operazioni.

Cosa contiene:
- Le classi *Bean* che implementano le interfacce di LISAServicesClient, organizzate per dominio:
  - GestioneContabilitaBean — provvigioni, budget, piano conti
  - GestioneClientiBean — anagrafica clienti
  - GestioneOrdiniBean — gestione ordini
  - ecc.
- I bean POJO non sono EJB diretti — sono classi Java normali che contengono la logica e vengono poi "avvolte" da LISAServices

Pattern di ogni bean: riceve un ApplicationContext, apre la connessione al DB, esegue le operazioni, restituisce il context popolato.

Dipende da: JavaCore + LISAServicesClient

---

**LISAServices — Modulo EJB (JAR, packaging ejb)**

È il punto di ingresso EJB dell'applicazione. Espone i servizi di business verso WildFly come Stateless Session Bean.

Cosa contiene:
- Bean EJB annotati con @Stateless e @Local che avvolgono i bean di POJOEJBServices:
```java
@Stateless
@Local(IGestioneContabilita.class)
public class GestioneContabilita implements IGestioneContabilita {
    // delega a GestioneContabilitaBean
}
```
- Il Persistor — implementazione ORM completa (~1300 righe) con operazioni CRUD, relazioni 1-N, N-N
- UserManagement, SessionRegistry — gestione sessioni e utenti
- BaseService — classe base con connessione al DB e utility condivise

Perché esiste separato da POJOEJBServices: WildFly richiede che i bean EJB siano nel modulo con packaging=ejb. POJOEJBServices contiene la logica pura (POJO, testabile), LISAServices aggiunge la "colla" EJB (transazioni, lifecycle gestito dal container, JNDI lookup).

Dipende da: JavaCore + LISAServicesClient + dipendenze DB: db2jcc (AS/400), mysql-connector

---

**LISA2Web — Servlet layer — canale client Swing (WAR)**

È il canale di comunicazione tra il client Swing e il server. Non è una web application con pagine HTML — è essenzialmente un router HTTP che riceve richieste dal client desktop, le smista ai bean EJB, e risponde.

Cosa contiene:

- FrontendControllerServlet — il controller centrale. Ogni operazione del client Swing passa da qui. Verifica l'autorizzazione, identifica l'operazione richiesta e delega alla servlet specifica
- RemoteRPCRouter — endpoint RPC generico: deserializza l'oggetto Request in arrivo, usa la reflection per trovare il bean EJB giusto tramite JNDI, invoca il metodo, serializza la Response
- LoginServlet — autenticazione utente
- PreloaderServlet — caricamento dati iniziali (combo, lookup) all'avvio del modulo
- DecoderServlet — decodifica dei codici (es. codice cliente → ragione sociale)
- Oltre 50 servlet specifiche per dominio (GestioneContabilitaServlet, GestioneClientiServlet, ecc.)

Protocollo di comunicazione: il client e il server si scambiano oggetti ApplicationContext serializzati, trasportati via HTTP. Il client manda il DO con i parametri dentro il context, il server risponde con i risultati nello stesso context.

Dipende da: JavaCore + POJOEJBServices + LISAServicesClient (tutti provided — già presenti nell'EAR)
Librerie aggiuntive: hessian (serializzazione binaria), axis (SOAP legacy)

---

**LISA2API — REST API (WAR)**

Espone endpoint REST/JSON per sistemi esterni e moderni. È completamente separato dal canale del client Swing.

Cosa contiene:

- RestBase — classe base di tutti gli endpoint: gestisce autenticazione token, serializzazione JSON, risposte di errore standard
- RestAuthenticator — valida i token di accesso tramite ISessionRegistry
- Oltre 50 resource class annotate con JAX-RS (@Path, @POST, @GET, @Stateless):
  - AuthResource — login e health check
  - ClientiResources — ricerca e lettura clienti
  - ProdottiResource — catalogo prodotti
  - ProduzioneResource — dati produzione
  - Controller di integrazione con sistemi esterni:
    - ShopifyEndpoint — e-commerce Shopify
    - WooSitipEndpoint — WooCommerce
    - IFMulesoftController, IF001Controller...IF012Controller — integrazione supply chain con GFCOpernico via Mulesoft

Differenza rispetto a LISA2Web: LISA2Web serve il client Swing proprietario con oggetti serializzati Java. LISA2API serve sistemi terzi con JSON standard via REST.

Dipende da: JavaCore + POJOEJBServices + LISAServicesClient (tutti provided)

---

**NETBridge — Bridge verso sistemi .NET (WAR)**

È un adattatore che permette a client e sistemi scritti in .NET (tipicamente i palmari di magazzino e sistemi legacy) di comunicare con il backend Java.

Cosa contiene:

- NETBridge.java — punto di ingresso SOAP: riceve un oggetto DataStructure, identifica l'operazione (login o altra), e instrada la richiesta
- DataStructure / DataStructureRecord — envelope di trasporto equivalente all'ApplicationContext ma pensata per .NET (campi: sessionID, operazione, sottocodice, callerIP, errorCode)
- ServiceRouter — mappa i codici operazione alle classi di servizio via reflection
- ServizioBase — classe base con accesso al DB e utility di conversione
- Oltre 20 servizi di dominio specifici per operazioni da palmare:
  - GestionePalmari, PalmareWRL5, PalmareRFID — gestione dispositivi mobili di magazzino
  - PrelievoWRL2, CaricoFornitore, ResoPartita — operazioni di picking, carico, reso
  - GestioneInventario, UbicazioneArticoloPartita — inventario e ubicazioni
  - GestioneMagazzinoAutomatico — magazzino automatizzato
  - SpedizioneCliente — tracking spedizioni

Dipende da: JavaCore + POJOEJBServices (entrambi provided)

---

**LISA2 — EAR aggregatore (EAR)**

Non contiene codice applicativo. Il suo unico scopo è assemblare tutti i moduli in un unico archivio deployabile su WildFly.

Cosa fa:
- Il pom.xml dichiara come dipendenze tutti i moduli precedenti
- Il plugin maven-ear-plugin li raccoglie e produce LISA2.ear con questa struttura interna:

```
LISA2.ear
├── LISAServices.jar        ← modulo EJB
├── LISA2Web.war            ← servlet layer
├── LISA2API.war            ← REST API
├── NETBridge.war           ← bridge .NET
└── lib/
    ├── javacore-0.0.1.jar       ← libreria core
    ├── POJOEJBServices-0.0.1.jar ← bean di business
    └── lisaservicesclient-0.0.1.jar ← interfacce EJB
```

- application.xml definisce i context-root dei WAR e la cartella lib/

WildFly carica l'EAR, registra i bean EJB nel container, e mette in ascolto i WAR sui rispettivi context path.

---

### Grafico delle dipendenze

```
JavaCore  ←──────────────────────────── tutto dipende da qui
    ↑
LISAServicesClient  ←─────────────────── contratti EJB
    ↑               ↑
POJOEJBServices     LISAServices  ←───── implementazione EJB
    ↑               ↑
    └───────┬────────┘
            ↓
       LISA2Web      ←────────────────── canale client Swing
       LISA2API      ←────────────────── REST per sistemi esterni
       NETBridge     ←────────────────── bridge .NET / palmari
            ↓
          LISA2 (EAR) ←─────────────── pacchetto finale
```

### Server

Il backend gira su **WildFly 26.1.3** (application server Java EE). Il modulo principale è `LISA2.ear`, un archivio che contiene tutti i WAR, il JAR EJB e le librerie condivise nella cartella `C:\dev\sharedlibs\java`.

Il database è un **AS/400 (IBM iSeries)**, accessibile via JDBC attraverso il driver `jt400`.

### Client

Il client è un'applicazione **Java Swing** desktop, costruita con **NetBeans/Ant**. Comunica con il server tramite HTTP chiamando la servlet `FrontendControllerServlet`, serializzando e deserializzando oggetti `ApplicationContext`. Non accede mai direttamente al database.

---

## 8. Il processo di build e deploy

### Server — Maven

Il file `build.bat` esegue in sequenza `mvn clean install` su ogni modulo, rispettando le dipendenze:

```bat
mvn -f JavaCore\pom.xml        clean install   ← 1°
mvn -f LISAServicesClient\pom.xml clean install ← 2°
mvn -f POJOEJBServices\pom.xml clean install   ← 3°
mvn -f LISAServices\pom.xml    clean install   ← 4°
mvn -f LISA2Web\pom.xml        clean install   ← 5°
mvn -f LISA2API\pom.xml        clean install   ← 6°
mvn -f NETBridge\pom.xml       clean install   ← 7°
mvn -f LISA2\pom.xml           clean install   ← 8° produce LISA2.ear
```

L'ordine è obbligatorio: ogni modulo dipende dal precedente. `clean install` compila, impacchetta il JAR e lo installa nel repository Maven locale (`~/.m2/repository`) perché i moduli successivi lo trovino come dipendenza.

Il `LISA2.ear` risultante viene copiato in:
```
C:\dev\WildFly\26.1.3.Final\standalone\deployments\LISA2.ear
```

WildFly monitora questa cartella e rideploya automaticamente quando il file cambia.

### Client — Ant + rilascio_client.bat

Il client Swing usa Ant (NetBeans). Il processo di rilascio è in tre fasi:

1. **Ant compila** i sorgenti e produce `LISAClient/dist/LISA.jar` (unico JAR monolitico)
2. **`rilascio_client.bat` esplode e risuddivide** il JAR in ~30 JAR tematici:
   ```bat
   jar.exe -xf LISA.jar                          ← estrae le .class
   jar.exe -cf gui/contabilita.jar ./lisa/gui/contabilita/  ← ricompatta per area
   jar.exe -cf gui/clienti.jar     ./lisa/gui/clienti/
   ...
   ```
3. I JAR vengono copiati nella cartella di baseline per la distribuzione.

La suddivisione in JAR tematici serve per **Java Web Start (JNLP)**: il client scarica solo i JAR delle aree che apre, non il monolite intero.

---

## 9. Il Data Object (DO)

I Data Object sono classi POJO che mappano le tabelle AS/400. Estendono `DataObject` e usano l'annotazione `@DOMetaData` su ogni campo pubblico.

```java
public class DOProvvigioni extends DataObject {
    @DOMetaData(description = "Data documento", tables = { "PROVV00F" }, size = 8, separator = true)
    public Data PRDDOC;
}
```

I nomi dei campi corrispondono esattamente ai nomi delle colonne sulla tabella AS/400.

---

## 10. L'annotazione `@DOMetaData`

Ogni campo pubblico del DO è annotato per descrivere come deve essere trattato dal framework.

### Attributi DB

| Attributo | Tipo | Default | Significato |
|---|---|---|---|
| `tables` | `String[]` | `{}` | Tabella/e AS/400 a cui appartiene la colonna. Se vuoto, il campo non viene mai scritto nel DB |
| `persistent` | `boolean` | `true` | Se `false`, il campo è escluso da INSERT/UPDATE |
| `identity` | `boolean` | `false` | PK auto-generata: il DataWriter la gestisce separatamente |
| `size` | `int` | `-1` | Lunghezza massima colonna (cifre o caratteri) |
| `decimal` | `int` | `-1` | Numero di decimali per i `Double` |

### Attributi stringa

| Attributo | Default | Significato |
|---|---|---|
| `upperCase` | `true` | Forza il valore in maiuscolo prima del salvataggio |
| `mustTrimSpaces` | `true` | Elimina spazi iniziali/finali (l'AS/400 riempie i CHAR con spazi) |

### Attributi serializzazione

| Attributo | Default | Significato |
|---|---|---|
| `separator` | `true` | Il campo è incluso nella serializzazione client-server |
| `transiente` | `false` | Il campo è escluso da tutto: né DB né protocollo. Solo memoria durante l'elaborazione |

### Attributi UI

| Attributo | Significato |
|---|---|
| `description` | Etichetta leggibile, usata in griglie e messaggi di errore |
| `decodificable` | Il campo è un codice con decodifica associata |
| `searchCode` | Codice del lookup da aprire per questo campo |
| `preloadCode` | Codice per pre-caricare i valori possibili |
| `filter` | Il campo è un parametro di filtro, non un dato reale |
| `flagAnnullo` | Questo campo è il flag di soft-delete del record |
| `renderClass` | Classe `RenderValue` per formattare la visualizzazione |
| `DDSStartPosition` / `DDSContentLen` | Posizione/lunghezza nel formato fisso DDS AS/400 (legacy) |

---

## 11. Il meccanismo di persistenza — solo scrittura

Il flag `persistent` governa **esclusivamente le operazioni di scrittura** (INSERT/UPDATE). Non ha nessun effetto sulla SELECT.

### Come funziona la scrittura (`DataWriter` / `SQLWriter`)

Il `DataWriter` usa la reflection per iterare tutti i campi del DO e costruire dinamicamente la query SQL:

```
Per ogni campo del DO:
  1. Ha @DOMetaData? → no: salta
  2. persistent == false? → salta
  3. tables[] contiene la tabella corrente? → no: salta
  4. È identity? → gestisci separatamente (RETURN_GENERATED_KEYS)
  5. → aggiungi alla INSERT o UPDATE
```

Esempio in `salvaProvvigione`:
```java
SQLWriter dw = new SQLWriter(filePROVV00F, conn);
dw.setWritePolicy(insertingNew ? WritePolicy.INSERT : WritePolicy.UPDATE);
dw.writeData(obj);   // ← scansiona tutti i campi persistent=true con tables=PROVV00F
dw.addKey("PRCCLI", obj.PRCCLI);
dw.addKey("RRN("+filePROVV00F+")", "=", obj.RECNBR);
dw.write();
```

### La scelta INSERT vs UPDATE

In `PROVV00F` si usa il **RRN (Relative Record Number)** dell'AS/400 come discriminante:
- `RECNBR == 0` → record nuovo → INSERT
- `RECNBR != 0` → record esistente → UPDATE identificato dal RRN fisico

### Validazione prima del salvataggio

`DataObject.checkDataTruncation()` verifica che ogni valore non ecceda `size` e `decimal` definiti nell'annotazione, prevenendo troncamenti silenziosi sull'AS/400.

### Campi `persistent = false`

Esistono solo in memoria. Esempi in `DOProvvigioni`:

| Campo | Motivo |
|---|---|
| `decPRCCLI` | Descrizione decodificata del cliente, non è una colonna |
| `origPRCAGE` | Copia temporanea del codice agente per la chiave UPDATE |
| `RECNBR` | RRN letto con `RRN(T) AS RECNBR` — colonna virtuale, non fisica |
| `fromPRDDOC` | Limite inferiore per il filtro data, non è una colonna |
| `toPRDDOC` | Limite superiore per il filtro data, non è una colonna |

---

## 12. La SELECT — completamente separata dalla persistenza

La lettura dei dati non usa reflection né `@DOMetaData`. Il bean costruisce la query manualmente usando la classe `Query`.

### Pattern della query

```java
Query query = new Query("SELECT RRN(T) AS RECNBR, T.* FROM tabella AS T", conn);

// Vincolo sempre applicato (3 argomenti):
query.addConstraint("PRFANN", "<>", "A");

// Vincolo opzionale (4 argomenti): aggiunto solo se valore != default
query.addConstraint("PRCCLI", "=", filtro.PRCCLI, "");   // "" = ignora se vuoto
query.addConstraint("PRNDOC", "=", filtro.PRNDOC, 0);    // 0  = ignora se zero

query.addOrderBy("ORDER BY PRDMOV, PRNMOV, PRRMOV");
query.setFirstRecordIndex(offset);   // paginazione
query.setMaxNumResult(limit + 1);    // +1 per sapere se esiste la pagina successiva
query.prepare();

Cursor c = query.getResultSet();
while (c.next()) {
    DOProvvigioni o = new DOProvvigioni();
    c.bind(o);   // popola il DO via reflection: nome colonna → nome campo Java
}
```

### Come funziona `c.bind(o)`

`ResultSetDefaultImpl.bind()` fa il percorso inverso della scrittura: legge ogni colonna dal `ResultSet` e la assegna al campo Java con lo stesso nome, via reflection. I campi `persistent = false` non vengono toccati perché non esistono come colonne nel risultato.

### Paginazione

Il metodo richiede `MAX_RESULT + 1` record (51 invece di 50):
- Se arrivano 51 → esistono altri record → abilita il pulsante "Avanti" (`W0001`)
- Se arrivano ≤ 50 → ultima pagina → disabilita "Avanti" (`W0002`)

---

## 13. Il GestioneContabilitaBean — il bean di business

Ogni metodo pubblico del bean segue questo pattern:

```java
public ApplicationContext metodo(ApplicationContext ctx) {
    Connection conn = null;
    try {
        conn = ctx.openDataBaseConnection();
        DOXxx obj = (DOXxx) ctx.getAttribute(CostantsDataObject.DO_XXX);
        // ... logica ...
        ctx.setAttribute(CostantsDataObject.DO_XXX, obj);
    } catch (Exception e) {
        ctx.markAsInvalid(Costants.ERROR_INVALID_OPERATION);
    } finally {
        ctx.closeDatabaseConnection(conn);
    }
    return ctx;
}
```

`ApplicationContext` è il contenitore che viaggia tra client e server, portando: connessione DB, attributi in ingresso, risultati in uscita, stato dell'operazione e dati utente.

---

## Note personali

- **redirect_uri: ogni carattere conta** — Spring Authorization Server fa un confronto esatto della stringa. `localhost` e `127.0.0.1` sono diversi. `/callback` e l'assenza di `/callback` sono diversi. Porta 4200 e porta 4201 sono diverse. Se il login fallisce con errore 400, il primo posto dove guardare è la tabella `mb_registered_application_redirect_uri_set` nel database PostgreSQL del Modulo Base NGP.

- **Due Modulo Base in locale** — La trinity usa il modulobase con SQL Server remoto (utenti di sviluppo condivisi), il progetto NGP usa il modulobase con PostgreSQL locale. Per il Survey in locale si usa sempre `authorization-dev.lisasuite.it` (NGP), non `localhost:9000` (trinity). La vecchia trinity-modulobase non viene più usata in locale.

- **La rete interna Docker è trasparente per i container** — I container si vedono tramite hostname senza che tu faccia nulla, ma dall'host (il tuo PC) devi usare `localhost:<porta mappata>`. Quando una variabile d'ambiente dentro un container dice `http://discoveryservice:9201`, funziona perché entrambi i container sono sulla stessa rete Docker. Se metti quell'URL nel browser, non funzionerà.
