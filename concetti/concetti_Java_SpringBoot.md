# Concetti — Java e Spring Boot

> **Categoria:** `Java / Spring Boot / Spring Data JPA / Spring Cloud / OAuth2`
> **Aggiornato:** `30/06/2026`

---

## Indice

**Java**

1. [Getter e Setter / Lombok](#1-getter-e-setter--lombok)
2. [Optional\<T\>](#2-optionalt)

**Spring Boot — Framework e architettura**

3. [Spring Boot](#3-spring-boot)
4. [Architettura a strati](#4-architettura-a-strati)
5. [@Transactional](#5-transactional)
6. [DTO — Data Transfer Object](#6-dto--data-transfer-object)
7. [Repository e Spring Data JPA](#7-repository-e-spring-data-jpa)
8. [Metodi Service — @Service e Dependency Injection](#8-metodi-service--service-e-dependency-injection)
9. [Endpoint Controller — REST](#9-endpoint-controller--rest)

**Spring — Ecosistema**

10. [OAuth2 — Protocollo e Spring Security OAuth2](#10-oauth2--protocollo-e-spring-security-oauth2)
11. [Service Discovery ed Eureka](#11-service-discovery-ed-eureka)
12. [Spring Cloud Eureka](#12-spring-cloud-eureka)
13. [API Gateway](#13-api-gateway)
14. [Swagger UI](#14-swagger-ui)

---

## 1. Getter e Setter / Lombok

### Teoria

In Java, il principio di **incapsulamento** dice che i campi di una classe devono essere privati (non accessibili direttamente dall'esterno) e accessibili solo tramite metodi pubblici. Questi metodi si chiamano **getter** (per leggere) e **setter** (per scrivere).

```java
// Senza getter/setter — MALE (il campo è pubblico, chiunque può modificarlo)
public class PickAnswerDTO {
    public String quizId;  // ← accessibile e modificabile da chiunque
}

// Con getter/setter — BENE (incapsulamento corretto)
public class PickAnswerDTO {
    private String quizId;  // ← nessuno può accedervi direttamente

    // Getter — metodo per LEGGERE il valore
    public String getQuizId() {
        return quizId;
    }

    // Setter — metodo per SCRIVERE il valore
    public void setQuizId(String quizId) {
        this.quizId = quizId;
    }
}
```

Dalla parte di chi usa il DTO:
```java
PickAnswerDTO dto = new PickAnswerDTO();
dto.setQuizId("abc123");   // setter — imposta il valore
String id = dto.getQuizId(); // getter — legge il valore
```

La convenzione **JavaBeans** (lo standard) impone che i getter si chiamino `get` + NomeCampoConPrimaMaiuscola e i setter `set` + NomeCampoConPrimaMaiuscola. Per i booleani il getter può chiamarsi `is` + NomeCampo (es: `isActive()`).

### Lombok — @Getter e @Setter

Scrivere getter e setter per ogni campo di ogni classe è estremamente ripetitivo. La libreria **Lombok** permette di generarli automaticamente con due annotazioni:

```java
@Getter
@Setter
public class PickAnswerDTO {
    private String quizId;
    private String questionId;
    private String newChoiceId;
}
```

Lombok legge le annotazioni durante la compilazione e **genera il bytecode** dei getter e setter senza che tu li scriva. Il risultato compilato è esattamente identico a quello che avresti scritto a mano.

Puoi applicare `@Getter`/`@Setter` anche sul singolo campo per controllare la visibilità:
```java
@Getter
@Setter(AccessLevel.PRIVATE) // il setter è privato — solo la classe può modificarlo
private String internaleId;
```

### Perché getter/setter sono essenziali in Spring

Spring (e in particolare Jackson, la libreria usata per convertire JSON in oggetti Java e viceversa) usa la **reflection** per trovare i campi di un oggetto tramite i loro getter/setter. Quando il Controller riceve la richiesta HTTP con il corpo JSON:

```json
{ "quizId": "abc", "questionId": "xyz", "newChoiceId": "123" }
```

Jackson cerca nell'oggetto `PickAnswerDTO` un metodo `setQuizId()`, un metodo `setQuestionId()`, un metodo `setNewChoiceId()` e li chiama con i valori dal JSON. Se i setter non esistono, il campo rimane `null`.

---

## 2. Optional\<T\>

### Il problema che risolve

Prima di `Optional`, il codice Java era pieno di controlli `null`:

```java
Answer answer = answerRepo.findSomething(id);
if (answer != null) {
    // usalo
} else {
    throw new NotFoundException(...);
}
```

Se dimentichi il controllo `null`, ottieni la famigerata **NullPointerException** a runtime. `Optional<T>` è un contenitore introdotto in Java 8 che rende esplicito il fatto che un valore potrebbe non esserci.

### Teoria

`Optional<T>` è un wrapper attorno a un valore di tipo `T` che può essere **presente** (`Optional` non vuoto) o **assente** (`Optional` vuoto). Invece di `null`, restituisci `Optional.empty()`. Invece di un valore, restituisci `Optional.of(valore)`.

### I metodi principali

```java
Optional<Answer> opt = answerRepo.findByAttemptQuizIdAndQuestionId(quizId, questionId);

// 1. isPresent() — controlla se il valore è presente, restituisce boolean
if (opt.isPresent()) {
    Answer a = opt.get(); // .get() estrae il valore (pericoloso se vuoto)
}

// 2. isEmpty() — controlla se è vuoto (Java 11+)
if (opt.isEmpty()) {
    // nessuna risposta trovata
}

// 3. ifPresent(consumer) — esegue l'azione SOLO se il valore è presente
// (NON confondere con isPresent()!)
opt.ifPresent(answer -> answerRepo.delete(answer));
// equivalente a:
// if (opt.isPresent()) { answerRepo.delete(opt.get()); }

// 4. orElse(default) — restituisce il valore se presente, altrimenti il default
Answer answer = opt.orElse(new Answer());

// 5. orElseGet(supplier) — come orElse ma il default è calcolato "pigramente"
Answer answer = opt.orElseGet(() -> new Answer());

// 6. orElseThrow(exception) — restituisce il valore se presente, altrimenti lancia eccezione
Answer answer = opt.orElseThrow(() -> new EntityNotFoundException("Answer not found"));
```

### Method reference — la sintassi `answerRepo::delete`

```java
existingAnswer.ifPresent(answerRepo::delete);
```

Il `::` è il **method reference** di Java: una sintassi compatta per riferirsi a un metodo senza chiamarlo. È equivalente a scrivere:

```java
existingAnswer.ifPresent(answer -> answerRepo.delete(answer));
```

`ifPresent` vuole una funzione (tecnicamente un `Consumer<Answer>`) — qualcosa che prende un `Answer` e fa qualcosa. `answerRepo::delete` prende già un `Answer`, quindi si usa come method reference diretto.

---

## 3. Spring Boot

**Spring Boot** è un framework Java che rende semplice creare applicazioni web e microservizi. Fa parte dell'ecosistema Spring, che esiste da oltre vent'anni ed è lo standard de facto per lo sviluppo enterprise in Java.

Il problema che Spring Boot ha risolto storicamente è quello della **configurazione**. Il vecchio Spring richiedeva file XML interminabili per configurare ogni componente. Spring Boot introduce il principio di **convention over configuration**: invece di configurare tutto manualmente, Spring Boot assume una serie di configurazioni predefinite ragionevoli, che puoi sovrascrivere solo quando necessario.

In pratica, con Spring Boot puoi creare un'applicazione web funzionante con pochissimo codice. Includi le dipendenze che ti servono (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, ecc.) e Spring Boot configura automaticamente tutto il necessario: un server web embedded (Tomcat), il connection pool al database, la serializzazione JSON, e così via.

Il Survey backend è un'applicazione Spring Boot che espone una REST API sulla porta 9090. All'avvio legge il file `application.yaml` (o `application-local.yaml` se usi il profilo `local`) per la configurazione, stabilisce la connessione al database, si registra su Eureka, e inizia ad accettare richieste HTTP.

I **profili** di Spring Boot sono un meccanismo elegante per avere configurazioni diverse per ambienti diversi. Il profilo `local` sovrascrive l'URL del database (da server remoto a localhost), l'URL dell'auth server, e altri parametri — senza toccare il codice sorgente.

---

## 4. Architettura a strati

Nell'architettura a strati di Spring Boot, ogni livello ha una responsabilità distinta:

```
Controller  ← gestisce HTTP (parsing richiesta, validazione input, costruzione risposta)
    ↓ delega a
Service     ← logica di business (cosa fare, come farlo, in quale ordine)
    ↓ usa
Repository  ← accesso ai dati (come leggere/scrivere nel database)
    ↓ tradotto da
JPA/Hibernate ← ORM (genera SQL, gestisce cache, mapping oggetti-tabelle)
    ↓ si connette tramite
Database
```

Ogni livello conosce solo quello sotto di lui, mai quelli sopra. Il Controller non tocca mai il Repository direttamente.

Il flusso dati per una richiesta di salvataggio:
```
Frontend Angular
  │ invia PickAnswerDTO { quizId, questionId, newChoiceId }
  ▼
Controller (deserializza il JSON nel DTO)
  │ passa il DTO al Service
  ▼
Service (usa il DTO per fare le query, costruisce l'Entity interna)
  │ salva l'Entity via Repository
  ▼
Database
```

---

## 5. @Transactional

### Teoria

Una **transazione di database** è un'unità di lavoro composta da più operazioni che devono essere eseguite tutte insieme o nessuna. Il concetto si riassume con l'acronimo **ACID**:

- **Atomicity** (Atomicità): la transazione è un tutto — o tutte le operazioni vanno a buon fine, o nessuna viene applicata al database
- **Consistency** (Consistenza): il database passa da uno stato valido a un altro stato valido
- **Isolation** (Isolamento): due transazioni concorrenti non si vedono a metà esecuzione
- **Durability** (Durabilità): una volta confermata (commit), la transazione è permanente anche in caso di crash

### Come funziona in Spring Boot

Spring Boot gestisce le transazioni tramite un **proxy**: quando chiami un metodo annotato con `@Transactional`, Spring non esegue il metodo direttamente. Crea invece un wrapper invisibile attorno al tuo oggetto che:

1. Prima del metodo: **apre** la transazione (`BEGIN TRANSACTION`)
2. Esegue il metodo normalmente
3. Se il metodo termina senza eccezioni: **conferma** (`COMMIT`) — tutte le modifiche diventano permanenti nel DB
4. Se il metodo lancia un'eccezione non controllata (RuntimeException): **annulla tutto** (`ROLLBACK`) — il DB torna allo stato precedente come se non fosse successo nulla

Questo meccanismo si chiama **AOP (Aspect-Oriented Programming)**: la logica di gestione della transazione viene "iniettata" attorno al tuo metodo senza che tu la scriva esplicitamente.

### Esempio pratico

```java
// AnswerService.java
@Transactional
public void setPickAnswer(PickAnswerDTO dto) {
    // OPERAZIONE 1: cerca risposta esistente
    Optional<Answer> existingAnswer = answerRepo.findByAttemptQuizIdAndQuestionId(
        attemptQuizInProgress.getId(), dto.getQuestionId()
    );
    
    // OPERAZIONE 2: se esiste, cancellala
    existingAnswer.ifPresent(answerRepo::delete);
    
    // OPERAZIONE 3: se c'è una nuova scelta, inseriscila
    if (dto.getNewChoiceId() != null) {
        answerRepo.save(answer);
    }
}
```

Il `@Transactional` garantisce che le operazioni 2 e 3 siano atomiche:

```
Scenario senza @Transactional (rischioso):
  DELETE vecchia risposta → OK ✓
  ... qualcosa va storto ...
  INSERT nuova risposta   → FALLISCE ✗
  Risultato: il candidato non ha nessuna risposta salvata — STATO INCONSISTENTE

Scenario con @Transactional:
  BEGIN TRANSACTION
  DELETE vecchia risposta → OK ✓
  ... qualcosa va storto ...
  INSERT nuova risposta   → FALLISCE ✗
  ROLLBACK automatico
  Risultato: il DB torna allo stato prima della chiamata — risposta precedente ancora presente
```

---

## 6. DTO — Data Transfer Object

### Teoria

Un **DTO (Data Transfer Object)** è un oggetto il cui unico scopo è trasportare dati tra livelli diversi dell'applicazione. Non contiene logica di business, solo campi e i loro getter/setter.

Il concetto nasce da un'esigenza precisa: l'oggetto che usi internamente (l'**Entity** JPA, che mappa una riga del database) spesso non è adatto per essere esposto all'esterno via API. I motivi sono diversi:

- L'Entity potrebbe avere campi sensibili (password, dati interni) che non devono uscire nell'API
- Il frontend potrebbe aver bisogno di una struttura dati diversa rispetto a come sono organizzate le tabelle
- Esporre l'Entity direttamente accoppia il contratto dell'API alla struttura del database — se rinomini una colonna, rompi il contratto con tutti i client

### DTO vs Entity — la differenza visiva

```java
// Entity (mappa la tabella del DB — uso interno)
@Entity
@Table(name = "answers")
public class Answer {
    @Id
    private String id;
    private String attemptQuizId;
    private String attemptPhaseId;
    private String companyId;      // ← campo interno, non ha senso inviarlo al frontend
    private String questionId;
    private String choiceId;
}

// DTO (contratto con il frontend — uso nell'API)
@Getter
@Setter
public class PickAnswerDTO {
    private String quizId;
    private String questionId;
    private String newChoiceId; // null = solo deselect
}
```

Il frontend invia solo i tre campi del DTO. Il service prende quel DTO, fa le sue query, costruisce l'Entity completa con tutti i campi (compreso `companyId` preso dal contesto di sicurezza), e la salva.

---

## 7. Repository e Spring Data JPA

### JPA — la specifica

**JPA (Java Persistence API)** è una specifica standard di Java (non una libreria concreta) che definisce come mappare oggetti Java su tabelle di database relazionale. Questo approccio si chiama **ORM (Object-Relational Mapping)**.

```java
@Entity                          // ← questa classe mappa una tabella
@Table(name = "answers")         // ← mappa la tabella "answers"
public class Answer {

    @Id                          // ← questo campo è la primary key
    private String id;

    @Column(name = "attempt_quiz_id") // ← mappa la colonna "attempt_quiz_id"
    private String attemptQuizId;

    private String questionId;   // ← mappa automaticamente la colonna "question_id"
                                 //   (Spring converte camelCase → snake_case di default)
}
```

### Hibernate — l'implementazione

JPA è solo la specifica (le regole). **Hibernate** è l'implementazione più usata: il motore concreto che prende le annotazioni JPA e genera SQL, gestisce la cache, esegue le query. Spring Boot usa Hibernate di default quando aggiungi `spring-boot-starter-data-jpa` alle dipendenze.

### Repository — il pattern

Il **Repository** è il livello di accesso ai dati. È il punto dell'applicazione dove si eseguono le query al database. La separazione della logica di accesso ai dati dalla logica di business (che sta nel Service) è uno dei principi cardine della buona architettura software.

In Spring Data JPA, il Repository è tipicamente un'**interfaccia** — non una classe. Non scrivi l'implementazione: è Spring che la genera automaticamente al momento dell'avvio dell'applicazione.

```java
// Repository per l'entità Answer
public interface AnswerRepository extends JpaRepository<Answer, String> {
    // JpaRepository<Entità, TipoDellaChiavePrimaria>
}
```

Estendendo `JpaRepository` ottieni **gratuitamente** metodi pronti:
- `save(entity)` — inserisce o aggiorna
- `findById(id)` — trova per chiave primaria, restituisce `Optional<Answer>`
- `findAll()` — restituisce tutte le righe
- `delete(entity)` — cancella l'entità
- `deleteById(id)` — cancella per chiave primaria
- `count()` — conta le righe

### Query derivate dal nome del metodo

La cosa più potente di Spring Data JPA è la possibilità di creare query semplicemente nominando il metodo in un certo modo:

```java
public interface AnswerRepository extends JpaRepository<Answer, String> {

    Optional<Answer> findByAttemptQuizIdAndQuestionId(String attemptQuizId, String questionId);
    //  ↑ "find" = SELECT
    //        ↑ "By" = WHERE
    //          ↑ "AttemptQuizId" = campo `attemptQuizId` dell'entità
    //                         ↑ "And" = AND SQL
    //                             ↑ "QuestionId" = campo `questionId` dell'entità
}
```

Spring traduce questo nome nel SQL:
```sql
SELECT * FROM answers
WHERE attempt_quiz_id = ?
  AND question_id = ?
LIMIT 1
```

### Convenzioni per i nomi

| Prefisso | Significato |
|---|---|
| `findBy...` | SELECT con WHERE |
| `findAllBy...` | SELECT con WHERE, ritorna Lista |
| `countBy...` | SELECT COUNT con WHERE |
| `deleteBy...` | DELETE con WHERE |
| `existsBy...` | SELECT EXISTS |

| Parola chiave nel nome | SQL generato |
|---|---|
| `And` | `AND` |
| `Or` | `OR` |
| `OrderBy...Asc` | `ORDER BY ... ASC` |
| `GreaterThan` | `>` |
| `LessThan` | `<` |
| `Like` | `LIKE` |
| `IsNull` | `IS NULL` |
| `Not` | `<>` o `NOT` |

### Lo stack completo JPA

```
La tua applicazione
      ↓ usa
Spring Data JPA (Repository, query derivate dai nomi)
      ↓ usa
JPA / EntityManager (spec standard Java)
      ↓ implementata da
Hibernate (ORM engine — genera SQL, gestisce cache, mapping)
      ↓ si connette tramite
JDBC (driver specifico del database)
      ↓ parla con
SQL Server / PostgreSQL / MySQL (database)
```

---

## 8. Metodi Service — @Service e Dependency Injection

### Il livello Service

Il **Service** è il livello dove vive la **logica di business**: le regole dell'applicazione, i calcoli, le operazioni sulle entità.

Il Service non sa niente di HTTP. Non conosce `HttpRequest`, `ResponseEntity`, status code. Sa solo fare cose con i dati. Questa separazione è fondamentale perché rende la logica di business **testabile in isolamento**: puoi scrivere test unitari del Service senza avviare un server HTTP.

### @Service — l'annotazione

```java
@Service    // ← dichiara questa classe come un bean Service di Spring
public class AnswerService {
    // ...
}
```

`@Service` fa tre cose:
1. Registra la classe nel **container IoC (Inversion of Control)** di Spring, rendendola disponibile per la **Dependency Injection**
2. Segnala a Spring che questo bean vive per tutta la durata dell'applicazione (è un **singleton** — ne esiste una sola istanza)
3. Semanticamente documenta il ruolo della classe

### Dependency Injection

Invece di creare gli oggetti con `new`, Spring li "inietta" automaticamente. Il Service ha bisogno del Repository? Dichiaralo nel costruttore e Spring lo fornisce:

```java
@Service
public class AnswerService {

    // Le dipendenze vengono "iniettate" da Spring
    private final AnswerRepository answerRepo;
    private final QuizRepository quizRepo;
    private final AttemptService attemptService;

    // Constructor injection (il modo preferito in Spring moderno)
    public AnswerService(
            AnswerRepository answerRepo,
            QuizRepository quizRepo,
            AttemptService attemptService) {
        this.answerRepo = answerRepo;
        this.quizRepo = quizRepo;
        this.attemptService = attemptService;
    }

    @Transactional
    public void setPickAnswer(PickAnswerDTO dto) {
        // usa answerRepo, quizRepo, attemptService...
    }
}
```

Spring vede che `AnswerService` ha bisogno di un `AnswerRepository` → cerca nel suo container se ne ha uno già creato → lo trova → lo fornisce automaticamente. Non devi mai scrivere `new AnswerRepository()`.

---

## 9. Endpoint Controller — REST

### Teoria

Il **Controller** è il livello di interfaccia HTTP dell'applicazione. Si occupa di:
- Ricevere le richieste HTTP
- Estrarre i dati dalla richiesta (header, parametri URL, corpo JSON)
- Delegare al Service la logica
- Costruire e restituire la risposta HTTP

```java
@RestController           // ← combina @Controller e @ResponseBody
@RequestMapping("/api/survey/answer")  // ← URL base per tutti i metodi di questa classe
public class AnswerAPI {
    // tutti i metodi qui gestiranno richieste a /api/survey/answer
}
```

### Le annotazioni di mapping

| Annotazione | Metodo HTTP | Semantica tipica |
|---|---|---|
| `@GetMapping` | GET | Leggi una risorsa |
| `@PostMapping` | POST | Crea una nuova risorsa |
| `@PutMapping` | PUT | Sostituisci completamente una risorsa |
| `@PatchMapping` | PATCH | Modifica parzialmente una risorsa |
| `@DeleteMapping` | DELETE | Cancella una risorsa |

### Come estrarre dati dalla richiesta

```java
@PutMapping
public ResponseEntity<Void> setPickAnswer(
        @RequestHeader("X-Tenant") String tenant,
        // ↑ legge l'header HTTP "X-Tenant" dalla richiesta
        
        @RequestParam(name = "lang", required = false, defaultValue = "IT") Language lang,
        // ↑ legge il parametro URL ?lang=EN (opzionale, default IT)
        
        @RequestBody PickAnswerDTO dto
        // ↑ deserializza il corpo JSON della richiesta nell'oggetto PickAnswerDTO
) {
    // ...
}
```

- `@RequestHeader("nome")` — estrae un header HTTP dalla richiesta
- `@RequestParam("nome")` — estrae un parametro dalla query string (`?nome=valore`)
- `@PathVariable` — estrae un segmento dall'URL, es: `/api/answers/{id}` → `@PathVariable String id`
- `@RequestBody` — deserializza il corpo JSON in un oggetto Java (usa Jackson sotto il cofano)

### ResponseEntity

`ResponseEntity<T>` è il tipo di ritorno che ti dà controllo completo sulla risposta HTTP:

```java
// Risposta 200 OK senza corpo
return ResponseEntity.ok().build();

// Risposta 200 OK con un oggetto nel corpo
return ResponseEntity.ok(myObject);

// Risposta 201 Created con la risorsa nel corpo
return ResponseEntity.status(HttpStatus.CREATED).body(createdObject);

// Risposta 404 Not Found senza corpo
return ResponseEntity.notFound().build();

// Risposta 400 Bad Request con messaggio di errore
return ResponseEntity.badRequest().body("Parametro mancante");
```

### PUT vs POST — perché importa la semantica

HTTP PUT è **idempotente**: chiamarlo più volte con gli stessi dati produce lo stesso risultato. "La risposta alla domanda X è ora la scelta Y" — se lo mandi tre volte, il risultato è lo stesso. HTTP POST non è idempotente: "crea una nuova risposta" chiamato tre volte crea tre righe nel database.

---

## 10. OAuth2 — Protocollo e Spring Security OAuth2

### OAuth2 — Il concetto

**OAuth2** è un protocollo di autorizzazione standard. Il suo scopo è permettere a un'applicazione di accedere a risorse protette per conto di un utente, senza che l'applicazione conosca le credenziali dell'utente.

Nel contesto del Survey, il flusso è questo:

1. Apri il browser su `http://localhost:4200` (frontend Angular)
2. Il frontend rileva che non sei autenticato
3. Il frontend ti reindirizza a `https://authorization-dev.lisasuite.it/login` (l'**Authorization Server**)
4. Inserisci username e password sul Modulo Base
5. Il Modulo Base verifica le credenziali e, se corrette, ti rilascia un **token JWT**
6. Vieni reindirizzato al frontend all'URL `http://localhost:4200/callback` con il codice di autorizzazione
7. Il frontend scambia il codice per il token JWT
8. Il frontend usa il token per tutte le chiamate successive al backend Survey
9. Il backend Survey, quando riceve una richiesta, verifica il token chiedendo al Modulo Base se è valido

Il token **JWT (JSON Web Token)** è una stringa codificata che contiene informazioni sull'utente (chi è, che ruoli ha, quando scade la sessione) e una firma digitale che permette al backend di verificarne l'autenticità senza dover interrogare il Modulo Base ad ogni richiesta.

### Il flusso completo — Authorization Code con PKCE

Il flusso usato dall'applicazione Survey si chiama **Authorization Code Flow con PKCE** (Proof Key for Code Exchange). È lo standard raccomandato per le applicazioni frontend che girano nel browser, dove non è possibile conservare un segreto in modo sicuro.

---

#### Fase 1 — Preparazione (Angular genera le chiavi PKCE)

Prima ancora di reindirizzare l'utente, Angular genera due valori casuali legati tra loro:

- **`code_verifier`**: una stringa casuale lunga generata localmente e tenuta in memoria da Angular (non viene mai inviata all'auth server in questa fase)
- **`code_challenge`**: l'hash SHA-256 del `code_verifier`, codificato in Base64. Questo viene inviato al Modulo Base.

L'idea è: se qualcuno intercetta la comunicazione in questa fase, ottiene solo l'hash — inutile senza il `code_verifier` originale.

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

| Parametro | Scopo |
|---|---|
| `client_id` | Identifica l'applicazione che chiede il login — deve corrispondere a un client registrato nel DB |
| `redirect_uri` | URL dove il Modulo Base manderà l'utente dopo il login — deve corrispondere esattamente a uno registrato nel DB |
| `response_type=code` | Dice che vogliamo il flusso "Authorization Code": prima un codice temporaneo, poi lo scambiamo per il token |
| `scope=openid` | Dice che vogliamo usare OpenID Connect (OIDC) — lo strato di identità sopra OAuth2 |
| `state` | Stringa casuale generata da Angular. Il Modulo Base la rimanderà indietro invariata. Angular verifica che corrisponda: protezione CSRF |
| `nonce` | Simile allo `state` ma per proteggere dall'uso ripetuto dello stesso token (replay attack) |
| `code_challenge` | L'hash del `code_verifier`. Il Modulo Base lo memorizza per usarlo nello step 4 |

---

#### Fase 3 — Login sul Modulo Base

Il browser è ora sulla pagina di login. L'utente inserisce username e password. Il Modulo Base verifica le credenziali nel database PostgreSQL. Se le credenziali sono corrette, il Modulo Base genera un **codice di autorizzazione** (authorization code): una stringa casuale, monouso, valida per pochi secondi.

---

#### Fase 4 — Redirect verso Angular con il codice

Il Modulo Base reindirizza il browser verso la `redirect_uri` dichiarata nello step 2:

```
GET http://localhost:4200/callback
  ?code=XXXXXXXXXXXXX    ← il codice di autorizzazione temporaneo
  &state=a41a4250afd2ceed...  ← lo stesso state di prima, invariato
```

Angular verifica che lo `state` corrisponda, poi estrae il `code`.

> **Perché un codice e non direttamente il token?** Il `code` viaggia nella URL del browser — potenzialmente loggato nella cronologia, nei proxy, nei log del server. Il token JWT è molto più prezioso: se qualcuno lo ottiene, può impersonare l'utente per tutta la sua durata. Separando i due step, il token non passa mai nell'URL del browser.

---

#### Fase 5 — Scambio codice → token JWT

Angular fa una chiamata HTTP POST **diretta** al Modulo Base:

```
POST https://authorization-dev.lisasuite.it/oauth2/token
  grant_type=authorization_code
  code=XXXXXXXXXXXXX          ← il codice ricevuto al passo 4
  redirect_uri=http://localhost:4200/callback
  client_id=angular-client
  code_verifier=<stringa-originale>  ← il code_verifier generato al passo 1
```

Il Modulo Base:
1. Verifica che il `code` esista e non sia già stato usato
2. Calcola l'hash SHA-256 del `code_verifier` ricevuto
3. Confronta l'hash con il `code_challenge` salvato al passo 2 — devono corrispondere
4. Se tutto è ok, risponde con il token JWT

Il controllo PKCE garantisce che solo chi ha generato la richiesta originale possa completare il flusso.

---

#### Fase 6 — L'applicazione è autenticata

Angular riceve il token JWT e da questo momento ogni chiamata al backend Survey include il token nell'header HTTP:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiIs...
```

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

Un dettaglio fondamentale di OAuth2: il redirect URI **deve essere registrato esplicitamente** nel database dell'auth server. Questo è un meccanismo di sicurezza: impedisce che un'applicazione malevola si spacci per un client legittimo e riceva il token di un utente su un sito diverso.

Spring Authorization Server fa un confronto **esatto** — non basta che il dominio sia giusto, ogni carattere deve corrispondere:

| URI inviata dal frontend | Registrata nel DB | Risultato |
|---|---|---|
| `http://localhost:4200/callback` | `http://localhost:4200/callback` | ✅ Login OK |
| `http://localhost:4200/callback` | `http://localhost:4200` | ❌ Errore 400 |
| `http://localhost:4200/callback` | `http://127.0.0.1:4200/callback` | ❌ Errore 400 |
| `http://localhost:4201/callback` | `http://localhost:4200/callback` | ❌ Errore 400 |

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

## 11. Service Discovery ed Eureka

In un'architettura a microservizi, i servizi devono trovarsi tra loro per comunicare. Nella soluzione più semplice, ogni servizio conosce l'indirizzo IP e la porta degli altri. Ma questo approccio ha un problema: in ambienti dinamici (cloud, container che si avviano e fermano, scaling automatico), gli indirizzi cambiano continuamente.

Il **Service Discovery** è il meccanismo che risolve questo problema. Funziona come una rubrica telefonica automatica: ogni servizio, quando parte, si "registra" con il proprio nome e il proprio indirizzo. Ogni servizio, quando ha bisogno di parlare con un altro, consulta la rubrica e ottiene l'indirizzo aggiornato.

**Eureka** è l'implementazione di Service Discovery sviluppata da Netflix e adottata da Spring Cloud. Esiste in due ruoli:

- **Eureka Server** (la rubrica): un servizio centralizzato che mantiene il registro di tutti i microservizi attivi. Nel progetto LISA è `discoveryservice`.
- **Eureka Client** (chi si registra e chi consulta): ogni microservizio che vuole essere trovato o vuole trovare altri include il client Eureka. Il Survey backend si registra come `lisa-survey` su Eureka all'avvio.

Il client Eureka manda periodicamente un "heartbeat" al server (ogni 30 secondi circa) per segnalare che è ancora vivo. Se un servizio non manda heartbeat per troppo tempo, Eureka lo rimuove dal registro.

---

## 12. Spring Cloud Eureka

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

> Nel profilo `local` del Survey, il `defaultZone` punta alla porta **9201** (il discovery service del progetto NGP), non alla porta 9101 (trinity-discovery). Le due istanze di Eureka sono indipendenti.

---

## 13. API Gateway

L'**API Gateway** è un servizio che fa da intermediario tra il mondo esterno e i microservizi interni. È l'unico punto di accesso alla piattaforma dall'esterno.

Senza gateway, il frontend dovrebbe conoscere l'indirizzo di ogni microservizio e chiamarli direttamente. Con il gateway:

- Il frontend fa **tutte le chiamate a un unico indirizzo** (il gateway)
- Il gateway consulta Eureka per sapere dove si trova il microservizio richiesto
- Il gateway instrada la richiesta al microservizio corretto
- Il gateway può applicare **politiche trasversali**: autenticazione, rate limiting, logging, CORS

Nel progetto LISA, `apigateway` (porta 9102) usa Spring Cloud Gateway e si appoggia a Eureka per sapere dove si trovano i vari microservizi. Quando una richiesta arriva al gateway con il prefisso `/survey/...`, il gateway sa che deve instradarla al servizio `lisa-survey` trovato nel registro Eureka.

---

## 14. Swagger UI

**Swagger UI** è un'interfaccia web che genera automaticamente la documentazione interattiva di una REST API a partire dal codice. Invece di leggere un documento statico, puoi vedere tutti gli endpoint disponibili, i parametri che accettano, le risposte che restituiscono, e puoi **provarli direttamente dal browser** senza scrivere una riga di codice.

Spring Boot integra Swagger tramite la libreria **SpringDoc OpenAPI**. Basta aggiungere la dipendenza e all'avvio dell'applicazione Swagger UI è disponibile automaticamente all'indirizzo:

```
http://localhost:9090/swagger-ui/index.html
```

Per il Survey backend, Swagger UI mostra tutti gli endpoint della REST API: i controller per i questionari, per le risposte, per i candidati, ecc. Swagger è uno strumento prezioso perché permette di testare le chiamate API direttamente, vedere cosa restituisce il backend e verificare che un fix funzioni, senza dover passare dal frontend.

---

## Note personali

- **OAuth2 è un protocollo di autorizzazione, non di autenticazione** — La distinzione è sottile ma importante. OAuth2 risponde a "questa app può accedere a queste risorse?", non a "chi è questo utente?". Per l'autenticazione si usa OpenID Connect (OIDC), che è uno strato sopra OAuth2. In pratica nella maggior parte delle implementazioni si usano insieme e la distinzione è teorica.

- **JWT è autosufficiente** — Il backend Survey non deve chiamare il Modulo Base ad ogni richiesta per verificare l'autenticità. Il token JWT contiene la firma digitale: il backend la verifica con la chiave pubblica scaricata una volta sola. Questo rende il sistema scalabile perché non c'è un collo di bottiglia sul server di autenticazione.

- **Spring Boot e il "magic"** — Spring Boot fa moltissime cose automaticamente "per magia" grazie all'autoconfiguration. Quando aggiungi `spring-boot-starter-data-jpa` alle dipendenze, Spring Boot configura automaticamente Hibernate, il DataSource, il connection pool. Capire cosa è configurato automaticamente e cosa devi configurare manualmente richiede esperienza con il framework.

- **`@Transactional` su metodi privati non funziona** — Spring crea un proxy attorno all'oggetto, e i proxy non intercettano le chiamate a metodi privati. Metti sempre `@Transactional` su metodi `public`. Stesso motivo per cui `@Transactional` su un metodo che chiama un altro `@Transactional` della **stessa classe** non apre una nuova transazione.

- **Optional non va usato come parametro di un metodo** — `Optional` è pensato per i valori di ritorno che "potrebbero non esserci". Come parametro di metodo è semanticamente sbagliato. Se un parametro può mancare, usa un normale `null` check o un overload del metodo.

- **Getter Lombok e JPA** — JPA usa i getter per accedere ai campi delle Entity durante la serializzazione e la gestione delle relazioni lazy. Se usi `@Getter` di Lombok sull'Entity, funziona perfettamente. Se rimuovi accidentalmente `@Getter`, Hibernate potrebbe avere comportamenti inattesi.
