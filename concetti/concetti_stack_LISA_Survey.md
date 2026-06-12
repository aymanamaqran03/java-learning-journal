# Concetti Tecnici — Stack LISA Survey

> **Categoria:** `Architettura / Docker / Spring Boot / Angular`
> **Data:** `12/06/2026`
> **Contesto:** Concetti incontrati durante il setup dell'ambiente di sviluppo locale del modulo LISA Survey

---

## Indice

1. [Architettura a Microservizi](#1-architettura-a-microservizi)
2. [Docker — Lezione Completa](#2-docker--lezione-completa)
   - Container
   - Image
   - Volumi
   - Docker Compose
   - File YAML
3. [La Trinità — L'infrastruttura base di LISA](#3-la-trinità--linfrastruttura-base-di-lisa)
   - trinity-discovery
   - trinity-apigateway
   - trinity-modulobase
   - Lettura guidata di trinity.yml
4. [Service Discovery ed Eureka](#4-service-discovery-ed-eureka)
5. [API Gateway](#5-api-gateway)
6. [Spring Boot](#6-spring-boot)
7. [Auth OAuth2 e Spring Security OAuth2](#7-auth-oauth2-e-spring-security-oauth2)
8. [Spring Cloud Eureka](#8-spring-cloud-eureka)
9. [Swagger UI](#9-swagger-ui)
10. [Angular e Angular CLI](#10-angular-e-angular-cli)
11. [DevExtreme UI](#11-devextreme-ui)

---

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

## 2. Docker — Lezione Completa

Docker è uno strumento che permette di eseguire applicazioni in ambienti isolati e portabili chiamati **container**. Per capire davvero cosa fa Docker, è necessario capire il problema che risolve.

### Il problema che Docker risolve

Immagina di dover eseguire un'applicazione Java che richiede esattamente JDK 17, una specifica versione di SQL Server, e alcune variabili d'ambiente configurate in un certo modo. Sul tuo PC potresti avere JDK 21 installato di default, una versione diversa di SQL Server, e configurazioni diverse. Il risultato classico è: "funziona sulla mia macchina ma non sulla tua". Docker elimina questo problema mettendo l'applicazione e tutto il suo ambiente in una scatola autonoma, che funzionerà in modo identico ovunque Docker sia installato.

---

### Container

Un **container** è un processo in esecuzione isolato dal resto del sistema operativo. Non è una macchina virtuale — non simula un hardware diverso, non ha un kernel separato. Usa il kernel del sistema operativo host ma è separato in termini di filesystem, rete e processi. Questo lo rende molto più leggero di una VM: si avvia in secondi, consuma poca memoria, e puoi averne decine in esecuzione contemporaneamente.

Pensa a un container come a un appartamento in un condominio: ogni appartamento ha il suo interno, i suoi oggetti, la sua porta chiusa. Gli inquilini non vedono cosa succede negli altri appartamenti. Ma tutti condividono le fondamenta, il tetto e l'impianto idraulico (il kernel del sistema operativo host). È molto diverso dall'avere case separate su terreni separati (le macchine virtuali), che richiedono ognuna la propria fondamenta e il proprio tetto.

Ogni container ha:
- Il suo **filesystem** isolato (non vede i file dell'host né degli altri container, a meno che non si configurino esplicitamente dei "ponti")
- La sua **rete** interna (ha il suo indirizzo IP, le sue porte)
- I suoi **processi** (il sistema operativo host non vede direttamente cosa gira dentro)
- Le sue **variabili d'ambiente**

Quando un container viene fermato (`docker stop`), i processi al suo interno si fermano, ma il container rimane disponibile. Quando viene rimosso (`docker rm`), scompare. Quando viene avviato di nuovo (`docker start`), riparte dallo stesso stato in cui era stato creato.

---

### Image

Un'**image** (immagine Docker) è il "progetto" da cui nasce un container. Se il container è una casa in esecuzione con persone che ci vivono dentro, l'image è il progetto architettonico: descrive come deve essere costruita la casa, cosa deve contenere, come è configurata. Da un unico progetto puoi costruire quante case identiche vuoi.

Un'image è composta da **layer** (strati) impilati uno sopra l'altro. Per esempio, l'image di SQL Server usata nel progetto (`mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04`) è costruita così:

1. **Layer base**: Ubuntu 16.04 (il sistema operativo)
2. **Layer SQL Server**: i binari di SQL Server 2019 installati sopra Ubuntu
3. **Layer configurazione**: script di avvio, variabili di default, ecc.

Questo sistema a layer ha un vantaggio enorme: se hai due immagini che partono dallo stesso Ubuntu, Docker scarica quel layer una volta sola e lo condivide. Risparmia spazio su disco e tempo di download.

Le immagini vengono scaricate da un **registry** — un archivio pubblico o privato di immagini. Il registry pubblico più famoso è **Docker Hub** (`hub.docker.com`). Quando hai eseguito `docker compose up` per la prima volta, Docker ha scaricato le immagini da Docker Hub perché non erano presenti in locale. Le immagini del progetto LISA (`javadevliminf/lisamodulobase:dev`, ecc.) vengono scaricate da un registry privato di Limonta Informatica.

Il tag dopo i due punti (`:dev`, `:2019-GA-ubuntu-16.04`) identifica la **versione** dell'immagine. `dev` significa che è la versione di sviluppo, aggiornata regolarmente dal team.

---

### Volumi

I container sono **efimeri per natura**: quando un container viene rimosso, tutto quello che era scritto nel suo filesystem sparisce. Per i database questo è un problema enorme — se fermi e rimuovi il container SQL Server, perdi tutti i dati.

I **volumi** sono il meccanismo di Docker per rendere i dati persistenti. Un volume è essenzialmente un collegamento tra una cartella nel filesystem del **container** e una cartella nel filesystem dell'**host** (il tuo PC). Ogni scrittura che il container fa in quella cartella finisce realmente sul tuo PC, e sopravvive alla rimozione del container.

Nel `mssql2019.yml` del Survey:

```yaml
volumes:
  - ${DOCKER_VOLUMES_ROOT}/mssql2019:/var/opt/mssql/data
```

Questa riga dice: "la cartella `/var/opt/mssql/data` dentro il container (dove SQL Server scrive i suoi file di database) deve essere mappata sulla cartella `C:\dev\NGP\volumes\mssql2019` sul tuo PC". Quando SQL Server crea il file del database `SURVEY-DEV`, lo scrive in `/var/opt/mssql/data`, che in realtà è `C:\dev\NGP\volumes\mssql2019` sul tuo disco. Se domani fermi e rimuovi il container e lo riavvii, SQL Server trova i file già lì e riparte con tutti i dati intatti.

---

### Docker Compose

Gestire un container alla volta con i comandi `docker run` diventa complicato quando hai più servizi che devono partire insieme, comunicare tra loro e condividere configurazioni. **Docker Compose** nasce per questo: ti permette di descrivere una **collezione di container** in un unico file di configurazione e gestirli tutti insieme con un solo comando.

Con Docker Compose puoi:
- Definire tutti i servizi che compongono la tua applicazione
- Specificare come si parlano tra loro (reti interne)
- Configurare variabili d'ambiente, porte, volumi per ciascuno
- Avviarli tutti con `docker compose up -d` e fermarli tutti con `docker compose down`

Il flag `-d` (detached) significa "avvia i container in background e restituisci il controllo del terminale" — senza di esso, i log di tutti i container scorrerebbero nel terminale finché non premi Ctrl+C.

---

### File YAML

I file `.yml` (o `.yaml`) usano il formato **YAML** (Yet Another Markup Language) per rappresentare dati strutturati in modo leggibile. È l'alternativa più moderna e leggibile al JSON o all'XML, ed è il formato scelto da Docker Compose, Spring Boot e moltissimi altri strumenti DevOps.

Le regole fondamentali di YAML:

**Indentazione = struttura gerarchica.** Non esistono parentesi o tag. La gerarchia è determinata dagli spazi (MAI tab). Ogni livello figlio è indentato di 2 spazi rispetto al padre.

**Coppia chiave: valore**
```yaml
nome: mario
eta: 30
```

**Lista** (il trattino `-` indica un elemento di lista)
```yaml
colori:
  - rosso
  - verde
  - blu
```

**Oggetto annidato**
```yaml
database:
  host: localhost
  porta: 1433
  nome: SURVEY-DEV
```

**Variabili d'ambiente** (sintassi `${VARIABILE}`)
```yaml
volumes:
  - ${DOCKER_VOLUMES_ROOT}/mssql2019:/var/opt/mssql/data
```
Il valore viene sostituito a runtime con il contenuto della variabile d'ambiente del sistema.

---

## 3. Il NGP Stack — L'infrastruttura base di LISA

L'infrastruttura base per sviluppare qualsiasi modulo LISA è il **NGP Stack**: un insieme di container Docker che devono essere sempre in esecuzione. A differenza della vecchia "Trinità" (un unico `trinity.yml`), ogni servizio ha ora il proprio file Docker Compose dedicato, con immagini aggiornate e allineate. I file si trovano in `C:\dev\NGP\ngp-stack\`.

---

### discoveryservice — Eureka

È il **registro centrale** dove tutti i microservizi si iscrivono all'avvio e comunicano la propria posizione sulla rete. Gira sulla porta **9201**. Tecnicamente è un'istanza di **Eureka Server** (vedi sezione dedicata più avanti). Senza questo container, i microservizi non riescono a trovarsi tra loro. Definito in `discovery-service.yml`.

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

## 4. Service Discovery ed Eureka

In un'architettura a microservizi, i servizi devono trovarsi tra loro per comunicare. Nella soluzione più semplice, ogni servizio conosce l'indirizzo IP e la porta degli altri. Ma questo approccio ha un problema: in ambienti dinamici (cloud, container che si avviano e fermano, scaling automatico), gli indirizzi cambiano continuamente.

Il **Service Discovery** è il meccanismo che risolve questo problema. Funziona come una rubrica telefonica automatica: ogni servizio, quando parte, si "registra" con il proprio nome e il proprio indirizzo. Ogni servizio, quando ha bisogno di parlare con un altro, consulta la rubrica e ottiene l'indirizzo aggiornato.

**Eureka** è l'implementazione di Service Discovery sviluppata da Netflix e adottata da Spring Cloud. Esiste in due ruoli:

- **Eureka Server** (la rubrica): un servizio centralizzato che mantiene il registro di tutti i microservizi attivi. Nel progetto LISA è `trinity-discovery`.
- **Eureka Client** (chi si registra e chi consulta): ogni microservizio che vuole essere trovato o vuole trovare altri include il client Eureka. Il Survey backend si registra come `lisa-survey` su Eureka all'avvio.

Il client Eureka manda periodicamente un "heartbeat" al server (ogni 30 secondi circa) per segnalare che è ancora vivo. Se un servizio non manda heartbeat per troppo tempo, Eureka lo rimuove dal registro.

---

## 5. API Gateway

L'**API Gateway** è un servizio che fa da intermediario tra il mondo esterno e i microservizi interni. È l'unico punto di accesso alla piattaforma dall'esterno.

Senza gateway, il frontend dovrebbe conoscere l'indirizzo di ogni microservizio e chiamarli direttamente. Con il gateway:

- Il frontend fa **tutte le chiamate a un unico indirizzo** (il gateway)
- Il gateway consulta Eureka per sapere dove si trova il microservizio richiesto
- Il gateway instrada la richiesta al microservizio corretto
- Il gateway può applicare **politiche trasversali**: autenticazione, rate limiting, logging, CORS

Nel progetto LISA, `trinity-apigateway` (porta 9102) usa Spring Cloud Gateway e si appoggia a Eureka per sapere dove si trovano i vari microservizi. Quando una richiesta arriva al gateway con il prefisso `/survey/...`, il gateway sa che deve instradarla al servizio `lisa-survey` che ha trovato nel registro Eureka.

---

## 6. Spring Boot

**Spring Boot** è un framework Java che rende semplice creare applicazioni web e microservizi. Fa parte dell'ecosistema Spring, che esiste da oltre vent'anni ed è lo standard de facto per lo sviluppo enterprise in Java.

Il problema che Spring Boot ha risolto storicamente è quello della **configurazione**. Il vecchio Spring richiedeva file XML interminabili per configurare ogni componente. Spring Boot introduce il principio di **convention over configuration**: invece di configurare tutto manualmente, Spring Boot assume una serie di configurazioni predefinite ragionevoli, che puoi sovrascrivere solo quando necessario.

In pratica, con Spring Boot puoi creare un'applicazione web funzionante con pochissimo codice. Includi le dipendenze che ti servono (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, ecc.) e Spring Boot configura automaticamente tutto il necessario: un server web embedded (Tomcat), il connection pool al database, la serializzazione JSON, e così via.

Il Survey backend è un'applicazione Spring Boot che espone una REST API sulla porta 9090. All'avvio legge il file `application.yaml` (o `application-local.yaml` se usi il profilo `local`) per la configurazione, stabilisce la connessione al database, si registra su Eureka, e inizia ad accettare richieste HTTP.

I **profili** di Spring Boot sono un meccanismo elegante per avere configurazioni diverse per ambienti diversi. Il profilo `local` sovrascrive l'URL del database (da server remoto a localhost), l'URL dell'auth server, e altri parametri — senza toccare il codice sorgente.

---

## 7. Auth OAuth2 e Spring Security OAuth2

### OAuth2 — Il concetto

**OAuth2** è un protocollo di autorizzazione standard. Il suo scopo è permettere a un'applicazione di accedere a risorse protette per conto di un utente, senza che l'applicazione conosca le credenziali dell'utente.

Nel contesto del Survey, il flusso è questo:

1. Apri il browser su `http://localhost:4200` (frontend Angular)
2. Il frontend rileva che non sei autenticato
3. Il frontend ti reindirizza a `https://authorization-dev.lisasuite.it/login` (il Modulo Base NGP, che è l'**Authorization Server**)
4. Inserisci username e password sul Modulo Base
5. Il Modulo Base verifica le credenziali e, se corrette, ti rilascia un **token JWT**
6. Vieni reindirizzato al frontend all'URL `http://localhost:4200/callback` con il codice di autorizzazione
7. Il frontend scambia il codice per il token JWT
8. Il frontend usa il token per tutte le chiamate successive al backend Survey
9. Il backend Survey, quando riceve una richiesta, verifica il token chiedendo al Modulo Base se è valido

Il token **JWT (JSON Web Token)** è una stringa codificata che contiene informazioni sull'utente (chi è, che ruoli ha, quando scade la sessione) e una firma digitale che permette al backend di verificarne l'autenticità senza dover interrogare il Modulo Base ad ogni richiesta.

### Il flusso completo — Authorization Code con PKCE

Il flusso usato dall'applicazione Survey si chiama **Authorization Code Flow con PKCE** (Proof Key for Code Exchange). È lo standard raccomandato per le applicazioni frontend che girano nel browser, dove non è possibile conservare un segreto in modo sicuro.

Ecco il flusso completo passo per passo, con i parametri reali che si vedono nelle URL del browser:

---

#### Fase 1 — Preparazione (Angular genera le chiavi PKCE)

Prima ancora di reindirizzare l'utente, Angular genera due valori casuali legati tra loro:

- **`code_verifier`**: una stringa casuale lunga generata localmente e tenuta in memoria da Angular (non viene mai inviata all'auth server in questa fase)
- **`code_challenge`**: l'hash SHA-256 del `code_verifier`, codificato in Base64. Questo viene inviato al Modulo Base.

L'idea è: se qualcuno intercetta la comunicazione in questa fase, ottiene solo l'hash — inutile senza il `code_verifier` originale. Il valore viene verificato poi nello step 4.

---

#### Fase 2 — Redirect verso il Modulo Base (Angular → Browser → Modulo Base)

Angular reindirizza il browser all'endpoint di autorizzazione del Modulo Base con questi parametri:

```
GET https://authorization-dev.lisasuite.it/oauth2/authorize
  ?client_id=angular-client          ← chi sta richiedendo il login
  &redirect_uri=http://localhost:4200/callback  ← dove mandare l'utente dopo
  &response_type=code                ← voglio un codice, non il token direttamente
  &scope=openid                      ← voglio le informazioni base sull'utente (OIDC)
  &state=a41a4250afd2ceed...         ← stringa casuale anti-CSRF generata da Angular
  &nonce=50fca9f59b962cde...         ← stringa casuale contro i replay attack
  &code_challenge=FOd3tw2RQLH3...    ← hash del code_verifier (PKCE)
  &code_challenge_method=S256        ← algoritmo usato per l'hash (SHA-256)
```

Cosa fa ogni parametro:

| Parametro | Scopo |
|---|---|
| `client_id` | Identifica l'applicazione che chiede il login — deve corrispondere a un client registrato nel DB |
| `redirect_uri` | URL dove il Modulo Base manderà l'utente dopo il login — deve corrispondere esattamente a uno registrato nel DB |
| `response_type=code` | Dice che vogliamo il flusso "Authorization Code": prima un codice temporaneo, poi lo scambiamo per il token |
| `scope=openid` | Dice che vogliamo usare OpenID Connect (OIDC) — lo strato di identità sopra OAuth2 |
| `state` | Stringa casuale generata da Angular. Il Modulo Base la rimanderà indietro invariata. Angular verifica che corrisponda: se qualcuno ha manipolato il redirect, lo `state` non tornerà uguale e Angular blocca tutto (protezione CSRF) |
| `nonce` | Simile allo `state` ma per proteggere dall'uso ripetuto dello stesso token (replay attack) |
| `code_challenge` | L'hash del `code_verifier`. Il Modulo Base lo memorizza per usarlo nello step 4 |
| `code_challenge_method=S256` | Dice che l'hash è SHA-256 |

---

#### Fase 3 — Login sul Modulo Base (Utente inserisce le credenziali)

Il browser è ora su `https://authorization-dev.lisasuite.it/login`. L'utente inserisce username e password. Il Modulo Base verifica le credenziali nel database PostgreSQL.

Se le credenziali sono corrette, il Modulo Base genera un **codice di autorizzazione** (authorization code): una stringa casuale, monouso, valida per pochi secondi.

---

#### Fase 4 — Redirect verso Angular con il codice (Modulo Base → Browser → Angular)

Il Modulo Base reindirizza il browser verso la `redirect_uri` dichiarata nello step 2, aggiungendo il codice come parametro:

```
GET http://localhost:4200/callback
  ?code=XXXXXXXXXXXXX    ← il codice di autorizzazione temporaneo
  &state=a41a4250afd2ceed...  ← lo stesso state di prima, invariato
```

Angular è in ascolto su questa rotta (`/callback`). Quando il browser arriva qui:
1. Verifica che lo `state` corrisponda a quello che aveva generato
2. Estrae il `code`
3. Passa alla fase successiva

> **Perché un codice e non direttamente il token?** Il `code` viaggia nella URL del browser — potenzialmente loggato nella cronologia, nei proxy, nei log del server. Il token JWT è molto più prezioso: se qualcuno lo ottiene, può impersonare l'utente per tutta la sua durata. Separando i due step, il token non passa mai nell'URL del browser.

---

#### Fase 5 — Scambio codice → token JWT (Angular → Modulo Base, chiamata diretta)

Angular fa una chiamata HTTP POST **diretta** al Modulo Base, non tramite il browser:

```
POST https://authorization-dev.lisasuite.it/oauth2/token
  grant_type=authorization_code
  code=XXXXXXXXXXXXX          ← il codice ricevuto al passo 4
  redirect_uri=http://localhost:4200/callback  ← deve coincidere con quello usato al passo 2
  client_id=angular-client
  code_verifier=<stringa-originale>  ← il code_verifier generato al passo 1 (mai inviato prima)
```

Il Modulo Base:
1. Verifica che il `code` esista e non sia già stato usato
2. Calcola l'hash SHA-256 del `code_verifier` ricevuto
3. Confronta l'hash con il `code_challenge` salvato al passo 2 — devono corrispondere
4. Se tutto è ok, risponde con il token JWT

Il controllo PKCE al punto 3 garantisce che solo chi ha generato la richiesta originale (Angular, che aveva il `code_verifier` in memoria) possa completare il flusso — anche se qualcuno avesse intercettato il `code` al passo 4.

---

#### Fase 6 — L'applicazione è autenticata

Angular riceve il token JWT, lo conserva in memoria e ti reindirizza su `/home` (configurato come `postLoginRoute`). Da questo momento ogni chiamata al backend Survey include il token nell'header HTTP:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiIs...
```

Il backend Survey riceve il token, verifica la firma con la chiave pubblica scaricata dal Modulo Base, e se tutto è valido processa la richiesta.

---

#### Sequenza visiva completa

```
Angular              Browser             Modulo Base          Survey Backend
  │                     │                     │                     │
  │──genera PKCE────────│                     │                     │
  │──redirect a /oauth2/authorize──────────►  │                     │
  │                     │  pagina di login    │                     │
  │                     │◄────────────────────│                     │
  │                     │  inserisci credenz. │                     │
  │                     │────────────────────►│                     │
  │                     │  redirect /callback │                     │
  │◄────────────────────│◄────────────────────│                     │
  │  verifica state     │                     │                     │
  │  POST /oauth2/token ─────────────────────►│                     │
  │                                           │ verifica PKCE       │
  │◄──────────── token JWT ───────────────────│                     │
  │                                           │                     │
  │──────────── GET /api/survey/... con Bearer token ──────────────►│
  │                                           │  verifica JWT       │
  │◄────────────────────────────────────────────── risposta ────────│
```

### redirect_uri — Perché deve essere registrata

Un dettaglio fondamentale di OAuth2: il redirect URI (l'URL a cui l'auth server manda l'utente dopo il login) **deve essere registrato esplicitamente** nel database dell'auth server. Questo è un meccanismo di sicurezza: impedisce che un'applicazione malevola si spacci per un client legittimo e riceva il token di un utente su un sito diverso.

Spring Authorization Server fa un confronto **esatto** — non basta che il dominio sia giusto, ogni carattere deve corrispondere:

| URI inviata dal frontend | Registrata nel DB | Risultato |
|---|---|---|
| `http://localhost:4200/callback` | `http://localhost:4200/callback` | ✅ Login OK |
| `http://localhost:4200/callback` | `http://localhost:4200` | ❌ Errore 400 |
| `http://localhost:4200/callback` | `http://127.0.0.1:4200/callback` | ❌ Errore 400 |
| `http://localhost:4201/callback` | `http://localhost:4200/callback` | ❌ Errore 400 |

Quando manca il redirect URI corretto, lo si aggiunge direttamente nel database del Modulo Base. In questo progetto il Modulo Base NGP usa PostgreSQL con una struttura dati personalizzata — i redirect URI sono nella tabella `mb_registered_application_redirect_uri_set`, non nella tabella standard Spring `oauth2_registered_client`.

### Due istanze di Modulo Base

Sul PC di sviluppo coesistono **due istanze diverse** del Modulo Base, per due contesti diversi:

In locale si usa esclusivamente il **Modulo Base NGP**:

| Container | Porta esposta | Database | URL pubblico |
|---|---|---|---|
| `lisamodulobase` | 9000 (dietro nginx) | PostgreSQL locale | `https://authorization-dev.lisasuite.it` |

Il profilo `local` del Survey punta a `https://authorization-dev.lisasuite.it` — nginx riceve la richiesta e la smista al container `lisamodulobase` sulla rete Docker `modulobase`. Gli utenti e i client OAuth2 (tra cui `angular-client`) vivono nel database PostgreSQL locale.

> La vecchia "trinity-modulobase" (che si connetteva a un SQL Server remoto su `10.204.204.114`) non viene più usata in locale — il team ha migrato tutto al NGP stack con immagini aggiornate e database locale.

### Spring Security OAuth2

**Spring Security** è il modulo di Spring che gestisce autenticazione e autorizzazione. **Spring Security OAuth2** estende Spring Security con il supporto al protocollo OAuth2.

Nel Survey backend, Spring Security OAuth2 è configurato come **Resource Server**: sa che le richieste in arrivo devono avere un token JWT valido nell'header HTTP (`Authorization: Bearer <token>`). Per verificare il token, il Survey backend contatta il Modulo Base all'URL configurato:

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

## 8. Spring Cloud Eureka

**Spring Cloud** è un insieme di librerie che aggiungono a Spring Boot le funzionalità tipiche dei sistemi distribuiti: service discovery, configuration management, circuit breaker, ecc.

**Spring Cloud Eureka** è il modulo specifico per integrare Spring Boot con Eureka. In ogni microservizio che vuole registrarsi su Eureka, si aggiunge la dipendenza `spring-cloud-starter-netflix-eureka-client` e si configurano poche righe in `application.yaml`:

```yaml
eureka:
  client:
    registerWithEureka: true      # "iscriviti al registro"
    fetchRegistry: true           # "scarica la lista degli altri servizi"
    serviceUrl:
      defaultZone: http://localhost:9201/eureka/   # indirizzo del server Eureka (NGP, profilo local)
```

Al momento dell'avvio, il microservizio si connette all'indirizzo `defaultZone` e annuncia: "Sono online, mi chiamo `lisa-survey`, sono raggiungibile all'IP X sulla porta 9090". Da quel momento, chiunque voglia parlare con `lisa-survey` può chiedere ad Eureka dove trovarlo.

> **Nota:** nel profilo `local` del Survey, il `defaultZone` punta alla porta **9201** (il discovery service del progetto NGP), non alla porta 9101 (trinity-discovery). Le due istanze di Eureka sono indipendenti.

---

## 9. Swagger UI

**Swagger UI** è un'interfaccia web che genera automaticamente la documentazione interattiva di una REST API a partire dal codice. Invece di leggere un documento statico, puoi vedere tutti gli endpoint disponibili, i parametri che accettano, le risposte che restituiscono, e — cosa molto utile — puoi **provarli direttamente dal browser** senza scrivere una riga di codice.

Spring Boot integra Swagger tramite la libreria **SpringDoc OpenAPI**. Basta aggiungere la dipendenza e all'avvio dell'applicazione Swagger UI è disponibile automaticamente all'indirizzo:

```
http://localhost:9090/swagger-ui/index.html
```

Per il Survey backend, Swagger UI mostra tutti gli endpoint della REST API: i controller per i questionari, per le risposte, per i candidati, ecc. Durante il bugfix, Swagger è uno strumento prezioso perché ti permette di testare le chiamate API direttamente, vedere cosa restituisce il backend e verificare che il fix funzioni, senza dover necessariamente passare dal frontend.

---

## 10. Angular e Angular CLI

### Angular

**Angular** è un framework per costruire applicazioni web lato client (frontend). È sviluppato da Google e scritto in **TypeScript** (un superset di JavaScript che aggiunge la tipizzazione statica). È il framework scelto per tutti i frontend della piattaforma LISA.

L'idea fondamentale di Angular è il modello **component-based**: l'interfaccia utente è suddivisa in componenti riutilizzabili. Ogni componente ha un template HTML (cosa mostrare), un file TypeScript (la logica), e un file CSS (lo stile). I componenti si compongono ad albero per formare l'intera applicazione.

Angular include anche:
- **Routing**: naviga tra "pagine" senza ricaricare il browser (Single Page Application)
- **Servizi e Dependency Injection**: logica condivisa (chiamate HTTP, gestione stato) separata dai componenti
- **Reactive Forms**: gestione avanzata dei form con validazione
- **HttpClient**: modulo per fare chiamate REST al backend

Il frontend del Survey (`lisa-survey`) è un'applicazione Angular che gira in locale su `http://localhost:4200`. Quando fai il login, Angular riceve il codice di autorizzazione da `authorization-dev.lisasuite.it` al callback `http://localhost:4200/callback` e poi scambia il codice per un token JWT, che include automaticamente in ogni chiamata HTTP al backend Survey tramite un **interceptor**.

### Angular CLI

**Angular CLI** (Command Line Interface) è lo strumento a riga di comando ufficiale per lavorare con Angular. Permette di:

- **Creare** nuovi progetti, componenti, servizi con struttura corretta: `ng generate component nome`
- **Compilare** il codice TypeScript in JavaScript ottimizzato: `ng build`
- **Avviare** il server di sviluppo locale con hot-reload: `ng serve`

Il comando che abbiamo usato, `npx ng serve --port 4200`, avvia il server di sviluppo:
- Compila il codice TypeScript
- Avvia un server HTTP locale
- Osserva i file sorgente: ogni volta che salvi una modifica, ricompila automaticamente e aggiorna il browser (questa funzione si chiama **hot-reload** o **live reload**)

`npx` è un comando di Node.js che permette di eseguire strumenti installati localmente nel progetto senza doverli installare globalmente. L'abbiamo usato al posto di `ng` direttamente perché PowerShell blocca l'esecuzione di script `.ps1` di terze parti per policy di sicurezza.

---

## 11. DevExtreme UI

**DevExtreme** è una libreria di componenti UI (User Interface) per Angular (e altri framework) sviluppata da DevExpress. Fornisce componenti pronti all'uso e altamente personalizzabili per costruire interfacce enterprise: griglie dati, grafici, form, calendar, scheduler, ecc.

Il componente più usato nei progetti LISA è probabilmente `dxDataGrid`: una griglia dati estremamente potente con ordinamento, filtraggio, paginazione, raggruppamento, export in Excel, editing inline, tutto incluso.

DevExtreme è una libreria **commerciale** (a pagamento), ma molto usata nel mondo enterprise proprio perché offre componenti già pronti per casi d'uso complessi che richiederebbero settimane di sviluppo custom. La vedi nel progetto come dipendenza `devextreme` e `devextreme-angular` nel `package.json`.

---

## Riepilogo visivo — Come i concetti si collegano

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

## Note personali

- **OAuth2 è un protocollo di autorizzazione, non di autenticazione** — la distinzione è sottile ma importante. OAuth2 risponde a "questa app può accedere a queste risorse?", non a "chi è questo utente?". Per l'autenticazione si usa OpenID Connect (OIDC), che è uno strato sopra OAuth2. In pratica nella maggior parte delle implementazioni si usano insieme e la distinzione è teorica.

- **La rete interna Docker è trasparente** — i container si vedono tramite hostname senza che tu faccia nulla. Ma dall'host (il tuo PC) devi usare `localhost:<porta mappata>`. Quando una variabile d'ambiente dentro un container dice `http://trinity-discovery:9101`, funziona perché entrambi i container sono sulla stessa rete Docker. Se metti quell'URL nel browser, non funzionerà.

- **Spring Boot e il "magic"** — Spring Boot fa moltissime cose automaticamente "per magia" grazie all'autoconfiguration. Quando aggiungi `spring-boot-starter-data-jpa` alle dipendenze, Spring Boot configura automaticamente Hibernate, il DataSource, il connection pool. Capire cosa è configurato automaticamente e cosa devi configurare manualmente richiede esperienza con il framework.

- **JWT è autosufficiente** — il backend Survey non deve chiamare il Modulo Base ad ogni richiesta per verificare l'autenticità. Il token JWT contiene la firma digitale: il backend la verifica con la chiave pubblica scaricata una volta sola. Questo rende il sistema scalabile perché non c'è un collo di bottiglia sul server di autenticazione.

- **redirect_uri: ogni carattere conta** — Spring Authorization Server fa un confronto esatto della stringa. `localhost` e `127.0.0.1` sono diversi. `/callback` e l'assenza di `/callback` sono diversi. Porta 4200 e porta 4201 sono diverse. Se il login fallisce con errore 400, il primo posto dove guardare è la tabella dei redirect URI registrati nel DB del modulobase.

- **Due Modulo Base in locale** — la trinity usa il modulobase con SQL Server remoto (utenti di sviluppo condivisi), il progetto NGP usa il modulobase con PostgreSQL locale. Per il Survey in locale si usa sempre `authorization-dev.lisasuite.it` (NGP), non `localhost:9000` (trinity).
