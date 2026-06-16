# Concetti Fondamentali — Spring Boot & Angular

> **Categoria:** `Spring Boot / Java / Angular / TypeScript`
> **Data:** `15/06/2026`
> **Contesto:** Concetti incontrati durante il fix della race condition nel salvataggio risposte del questionario STEP-33+
> **Collegato a:** `fix_RaceCondition_SalvataggioRisposteQuestionario.md`

---

## Indice

**Backend — Spring Boot / Java**

1. [@Transactional](#1-transactional)
2. [DTO — Data Transfer Object](#2-dto--data-transfer-object)
3. [Getter e Setter](#3-getter-e-setter)
4. [Repository Java](#4-repository-java)
5. [Spring Data JPA](#5-spring-data-jpa)
6. [Optional\<T\>](#6-optionalt)
7. [Metodi Service](#7-metodi-service)
8. [Endpoint Controller](#8-endpoint-controller)
9. [Tenant e Multi-Tenancy](#9-tenant-e-multi-tenancy)

**Frontend — Angular / TypeScript**

10. [Model Angular (.ts)](#10-model-angular-ts)
11. [Service Angular](#11-service-angular)
12. [Helpers Angular](#12-helpers-angular)

---

## 1. @Transactional

### Teoria

Una **transazione di database** è un'unità di lavoro composta da più operazioni che devono essere eseguite tutte insieme o nessuna. Il concetto si riassume con l'acronimo **ACID**:

- **Atomicity** (Atomicità): la transazione è un tutto — o tutte le operazioni vanno a buon fine, o nessuna viene applicata al database
- **Consistency** (Consistenza): il database passa da uno stato valido a un altro stato valido
- **Isolation** (Isolamento): due transazioni concorrenti non si vedono a metà esecuzione
- **Durability** (Durabilità): una volta confermata (commit), la transazione è permanente anche in caso di crash

Senza transazioni, potresti trovarti con un database in uno stato inconsistente: il `DELETE` della vecchia risposta è andato a buon fine, ma poi un errore nel `INSERT` della nuova ha fatto crashare tutto — lasciandoti senza nessuna risposta salvata.

### Come funziona in Spring Boot

Spring Boot gestisce le transazioni tramite un **proxy**: quando chiami un metodo annotato con `@Transactional`, Spring non esegue il metodo direttamente. Crea invece un wrapper invisibile attorno al tuo oggetto che:

1. Prima del metodo: **apre** la transazione (`BEGIN TRANSACTION`)
2. Esegue il metodo normalmente
3. Se il metodo termina senza eccezioni: **conferma** (`COMMIT`) — tutte le modifiche diventano permanenti nel DB
4. Se il metodo lancia un'eccezione non controllata (RuntimeException): **annulla tutto** (`ROLLBACK`) — il DB torna allo stato precedente come se non fosse successo nulla

Questo meccanismo si chiama **AOP (Aspect-Oriented Programming)**: la logica di gestione della transazione viene "iniettata" attorno al tuo metodo senza che tu la scriva esplicitamente.

### Esempio pratico dal codice

```java
// AnswerService.java
@Transactional
public void setPickAnswer(PickAnswerDTO dto) {
    // ...
    
    // OPERAZIONE 1: cerca risposta esistente
    Optional<Answer> existingAnswer = answerRepo.findByAttemptQuizIdAndQuestionId(
        attemptQuizInProgress.getId(), dto.getQuestionId()
    );
    
    // OPERAZIONE 2: se esiste, cancellala
    existingAnswer.ifPresent(answerRepo::delete);
    
    // OPERAZIONE 3: se c'è una nuova scelta, inseriscila
    if (dto.getNewChoiceId() != null) {
        // ...
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

### Nota importante

`@Transactional` funziona **solo su chiamate che arrivano dall'esterno della classe**. Se un metodo `@Transactional` chiama un altro metodo `@Transactional` della **stessa classe**, il secondo non apre una nuova transazione (il proxy non intercetta le chiamate interne). Per evitare questo problema, i metodi transazionali vengono di solito messi in classi service separate.

---

## 2. DTO — Data Transfer Object

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
    // ... altri campi JPA
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

Il frontend invia solo i tre campi del DTO. Il service prende quel DTO, fa le sue query, costruisce l'Entity completa con tutti i campi (compreso `companyId` preso dal contesto di sicurezza), e la salva. Il DB non espone mai la sua struttura completa all'esterno.

### Il flusso dati

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

La separazione in livelli (Controller → Service → Repository) si chiama architettura **a strati** (layered architecture). Ogni livello conosce solo quello sotto di lui, mai quelli sopra. Il Controller non tocca mai il Repository direttamente.

---

## 3. Getter e Setter

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

Lombok legge le annotazioni durante la compilazione e **genera il bytecode** dei getter e setter senza che tu li scriva. Il risultato compilato è esattamente identico a quello che avresti scritto a mano — il file `.class` contiene tutti i metodi, li vedi semplicemente in meno righe di codice sorgente.

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

Jackson cerca nell'oggetto `PickAnswerDTO` un metodo `setQuizId()`, un metodo `setQuestionId()`, un metodo `setNewChoiceId()` e li chiama con i valori dal JSON. Se i setter non esistono, il campo rimane `null` e non sai perché.

---

## 4. Repository Java

### Teoria

Il **Repository** è il livello di accesso ai dati. È il punto dell'applicazione dove si eseguono le query al database. La separazione della logica di accesso ai dati dalla logica di business (che sta nel Service) è uno dei principi cardine della buona architettura software: se un giorno cambi database, cambi solo il Repository, non il Service.

In Spring Data JPA, il Repository è tipicamente un'**interfaccia** — non una classe. Non scrivi l'implementazione: è Spring che la genera automaticamente al momento dell'avvio dell'applicazione.

### JpaRepository — l'interfaccia base

```java
// Repository per l'entità Answer
public interface AnswerRepository extends JpaRepository<Answer, String> {
    // JpaRepository<Entità, TipoDellaChiavePrimaria>
    //   ↑ Answer = tipo dell'entità
    //              ↑ String = tipo dell'@Id (la primary key)
}
```

Estendendo `JpaRepository` ottieni **gratuitamente** metodi pronti:
- `save(entity)` — inserisce o aggiorna (se ha già un ID, aggiorna; altrimenti inserisce)
- `findById(id)` — trova per chiave primaria, restituisce `Optional<Answer>`
- `findAll()` — restituisce tutte le righe
- `delete(entity)` — cancella l'entità
- `deleteById(id)` — cancella per chiave primaria
- `count()` — conta le righe
- e molti altri...

### Query derivate dal nome del metodo

La cosa più potente di Spring Data JPA è la possibilità di creare query semplicemente nominando il metodo in un certo modo. Spring legge il nome e costruisce automaticamente il SQL corrispondente:

```java
public interface AnswerRepository extends JpaRepository<Answer, String> {

    // Metodo aggiunto per il fix della race condition
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

Il `LIMIT 1` è implicito perché il tipo di ritorno è `Optional<Answer>` (al massimo uno), non `List<Answer>` (zero o più).

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

---

## 5. Spring Data JPA

### JPA — la specifica

**JPA (Java Persistence API)** è una specifica standard di Java (non una libreria concreta) che definisce come mappare oggetti Java su tabelle di database relazionale. Questo approccio si chiama **ORM (Object-Relational Mapping)**.

L'idea è semplice: invece di scrivere SQL a mano, scrivi classi Java annotate, e il framework si occupa di generare e gestire le query SQL per te. Un oggetto Java diventa una riga di tabella.

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

Il rapporto è analogo a quello tra JDBC (specifica per connettersi a DB Java) e il driver specifico del database (l'implementazione concreta per SQL Server, PostgreSQL, ecc.).

### Spring Data JPA — il livello sopra

**Spring Data JPA** aggiunge sopra JPA+Hibernate il sistema dei Repository che hai visto nella sezione precedente. Il suo valore principale è eliminare il codice boilerplate delle query CRUD comuni. Senza Spring Data JPA dovresti scrivere:

```java
// Senza Spring Data JPA — verboso
public class AnswerRepositoryImpl {
    @PersistenceContext
    private EntityManager em;

    public Optional<Answer> findByAttemptQuizIdAndQuestionId(String qId, String ques) {
        String jpql = "SELECT a FROM Answer a WHERE a.attemptQuizId = :qId AND a.questionId = :ques";
        TypedQuery<Answer> query = em.createQuery(jpql, Answer.class);
        query.setParameter("qId", qId);
        query.setParameter("ques", ques);
        List<Answer> results = query.getResultList();
        return results.isEmpty() ? Optional.empty() : Optional.of(results.get(0));
    }
}

// Con Spring Data JPA — una riga
Optional<Answer> findByAttemptQuizIdAndQuestionId(String attemptQuizId, String questionId);
```

### Lo stack completo

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

## 6. Optional\<T\>

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

Se dimentichi il controllo `null`, ottieni la famigerata **NullPointerException** a runtime. `Optional<T>` è un contenitore introdotto in Java 8 che rende esplicito il fatto che un valore potrebbe non esserci — il tipo di ritorno stesso comunica questa possibilità, e il compilatore ti forza a gestirla.

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
// Se opt è vuoto → lancia EntityNotFoundException
// Se opt è pieno → restituisce il valore direttamente (già "spacchettato" dall'Optional)
```

### Method reference — la sintassi `answerRepo::delete`

Nel codice del fix vedi questa sintassi:

```java
existingAnswer.ifPresent(answerRepo::delete);
```

Il `::` è il **method reference** di Java: una sintassi compatta per riferirsi a un metodo senza chiamarlo. È equivalente a scrivere:

```java
existingAnswer.ifPresent(answer -> answerRepo.delete(answer));
```

`ifPresent` vuole una funzione (tecnicamente un `Consumer<Answer>`) — qualcosa che prende un `Answer` e fa qualcosa. Puoi dargli una lambda `answer -> answerRepo.delete(answer)`, oppure, siccome `answerRepo.delete` prende già un `Answer`, puoi usare il method reference diretto `answerRepo::delete`. Stessa cosa, codice più corto.

---

## 7. Metodi Service

### Teoria — il livello Service

Nell'architettura a strati di Spring Boot, il **Service** è il livello dove vive la **logica di business**: le regole dell'applicazione, i calcoli, le operazioni sulle entità.

```
Controller  ← gestisce HTTP (parsing richiesta, validazione input, costruzione risposta)
    ↓ delega a
Service     ← logica di business (cosa fare, come farlo, in quale ordine)
    ↓ usa
Repository  ← accesso ai dati (come leggere/scrivere nel database)
```

Il Service non sa niente di HTTP. Non conosce `HttpRequest`, `ResponseEntity`, status code. Sa solo fare cose con i dati. Questo separazione è fondamentale perché rende la logica di business **testabile in isolamento**: puoi scrivere test unitari del Service senza avviare un server HTTP.

### @Service — l'annotazione

```java
@Service    // ← dichiara questa classe come un bean Service di Spring
public class AnswerService {
    // ...
}
```

`@Service` è un'annotazione speciale che fa tre cose:
1. Registra la classe nel **container IoC (Inversion of Control)** di Spring, rendendola disponibile per la **Dependency Injection**
2. Segnala a Spring che questo bean vive per tutta la durata dell'applicazione (è un **singleton** — ne esiste una sola istanza)
3. Semanticamente documenta il ruolo della classe (è un servizio, non un controller o un repository)

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

Spring vede che `AnswerService` ha bisogno di un `AnswerRepository` → cerca nel suo container se ne ha uno già creato → lo trova (perché `AnswerRepository` è annotata con `@Repository` o estende `JpaRepository`) → lo fornisce automaticamente. Non devi mai scrivere `new AnswerRepository()`.

---

## 8. Endpoint Controller

### Teoria

Il **Controller** è il livello di interfaccia HTTP dell'applicazione. Si occupa di:
- Ricevere le richieste HTTP
- Estrarre i dati dalla richiesta (header, parametri URL, corpo JSON)
- Delegare al Service la logica
- Costruire e restituire la risposta HTTP

In Spring Boot, un Controller si crea con l'annotazione `@RestController`:

```java
@RestController           // ← combina @Controller e @ResponseBody
@RequestMapping("/api/survey/answer")  // ← URL base per tutti i metodi di questa classe
public class AnswerAPI {
    // tutti i metodi qui gestiranno richieste a /api/survey/answer
}
```

### Le annotazioni di mapping

Ogni metodo del Controller gestisce un tipo specifico di richiesta HTTP:

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
- `@PathVariable` (non usato qui) — estrae un segmento dall'URL, es: `/api/answers/{id}` → `@PathVariable String id`
- `@RequestBody` — deserializza il corpo JSON in un oggetto Java (usa Jackson sotto il cofano)

### ResponseEntity

`ResponseEntity<T>` è il tipo di ritorno che ti dà controllo completo sulla risposta HTTP: status code, header, corpo.

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

Quando il Controller usa `@RestController`, Spring serializza automaticamente in JSON qualunque oggetto restituisci. `ResponseEntity<Void>` significa che non c'è corpo nella risposta (solo lo status code e gli header).

### PUT vs POST — perché importa la semantica

```java
// PRIMA del fix: POST era sbagliato semanticamente
@PostMapping  // POST = "crea una nuova risorsa"
public ResponseEntity<Void> submitAnswer(...) { ... }

// DOPO il fix: PUT è corretto
@PutMapping   // PUT = "sostituisci la risorsa corrente con questa"
public ResponseEntity<Void> setPickAnswer(...) { ... }
```

HTTP PUT è **idempotente**: chiamarlo più volte con gli stessi dati produce lo stesso risultato. "La risposta alla domanda X è ora la scelta Y" — se lo mandi tre volte, il risultato è lo stesso. HTTP POST non è idempotente: "crea una nuova risposta" chiamato tre volte crea tre righe nel database.

---

## 9. Tenant e Multi-Tenancy

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

## 10. Model Angular (.ts)

### Cos'è TypeScript

Prima di parlare dei model Angular, è fondamentale capire **TypeScript**. Angular è scritto in TypeScript, un linguaggio che **estende JavaScript** aggiungendo la tipizzazione statica. Quando scrivi codice TypeScript, lo **compili** in JavaScript puro (che il browser capisce). TypeScript non esiste a runtime — serve solo a te durante lo sviluppo.

Il vantaggio principale: il compilatore TypeScript ti dice in anticipo (mentre scrivi, nell'IDE) se stai usando un tipo sbagliato. In JavaScript puro, molti errori si scoprono solo a runtime (quando il programma è già in esecuzione e magari in mano all'utente).

```typescript
// JavaScript — nessun errore a compile time
function somma(a, b) { return a + b; }
somma("5", 3); // → "53" (concatena string e number!) — errore silenzioso

// TypeScript — errore a compile time
function somma(a: number, b: number): number { return a + b; }
somma("5", 3); // ← ERRORE: Argument of type 'string' is not assignable to parameter of type 'number'
```

### Interface vs Class in TypeScript

In Angular, i modelli di dati si definiscono quasi sempre come **interface** TypeScript, non come classi:

```typescript
// Interface (preferita per DTO e modelli di dati)
export interface PickAnswerDTO {
    quizId?: string;
    questionId?: string;
    newChoiceId?: string | null;
}

// Class (usata quando servono metodi o costruttori)
export class User {
    constructor(
        public id: string,
        public name: string
    ) {}
    
    getFullName(): string {
        return `User: ${this.name}`;
    }
}
```

La differenza pratica:
- Un'**interface** è solo un contratto: "questo oggetto deve avere questi campi con questi tipi". Non genera codice JavaScript — esiste solo a livello TypeScript per il type checking.
- Una **classe** genera codice JavaScript reale. Può avere costruttori, metodi, ereditarietà.

Per oggetti che sono semplici contenitori di dati (come i DTO), usa interface. Per oggetti con comportamento, usa class.

### Il simbolo `?` — proprietà opzionali

```typescript
export interface PickAnswerDTO {
    quizId?: string;      // ← il ? significa: questa proprietà è OPZIONALE
    questionId?: string;  // ← può essere presente o assente
    newChoiceId?: string | null; // ← può essere string, null, oppure assente
}
```

Senza `?`, la proprietà è **obbligatoria** — TypeScript ti darà errore se cerchi di creare un oggetto senza quel campo. Con `?`, puoi creare l'oggetto senza quel campo (sarà `undefined`).

### Il tipo union — `string | null`

```typescript
newChoiceId?: string | null;
// ↑ questo campo può essere:
//   - una stringa (es: "abc123")
//   - null (esplicitamente "nessun valore")
//   - assente (non incluso nell'oggetto)
```

In TypeScript `null` e `undefined` sono tipi distinti. `null` significa "valore esplicitamente assente", `undefined` significa "campo non inizializzato / non presente". Nel DTO del fix, `null` ha un significato semantico preciso: "deseleziona la risposta senza inserirne una nuova".

### Export e barrel file

```typescript
// pickAnswerDTO.ts — definizione del modello
export interface PickAnswerDTO {
    quizId?: string;
    questionId?: string;
    newChoiceId?: string | null;
}
```

La parola `export` rende l'interface disponibile per essere importata da altri file. Per usarla:

```typescript
// In un altro file
import { PickAnswerDTO } from './pickAnswerDTO';
```

Per non dover ricordare il percorso esatto di ogni modello, si usa un **barrel file** (tipicamente `models.ts` o `index.ts`): un file che ri-esporta tutto in un unico posto.

```typescript
// models.ts — barrel file
export * from './pickAnswerDTO';  // ← re-esporta tutto ciò che è in pickAnswerDTO.ts
export * from './phaseAnswersDTO';
export * from './quiz';
// ...
```

Così invece di:
```typescript
import { PickAnswerDTO } from '../../api/v1/model/pickAnswerDTO';
import { PhaseAnswersDTO } from '../../api/v1/model/phaseAnswersDTO';
```

Scrivi:
```typescript
import { PickAnswerDTO, PhaseAnswersDTO } from '../../api/v1/model/models';
```

---

## 11. Service Angular

### Teoria

In Angular, un **Service** è una classe con il decoratore `@Injectable` che contiene logica condivisa tra componenti. I Service sono il meccanismo standard per:

- Fare chiamate HTTP al backend
- Condividere stato tra componenti
- Contenere logica di business che non appartiene a un singolo componente

A differenza dei **componenti** (che hanno un template HTML e riguardano la visualizzazione), i service non hanno interfaccia grafica. Sono singletons: Angular ne crea una sola istanza e la condivide tra tutti i componenti che la richiedono.

### @Injectable — il decoratore

```typescript
@Injectable({
    providedIn: 'root'  // ← singleton per tutta l'applicazione
})
export class AnswerApiService {
    // ...
}
```

`@Injectable({ providedIn: 'root' })` registra il service nell'**Injector** di Angular (equivalente al container IoC di Spring). Quando un componente dichiara di aver bisogno di questo service nel costruttore, Angular lo fornisce automaticamente — non devi fare `new AnswerApiService()`.

### HttpClient — chiamate al backend

```typescript
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AnswerApiService {

    constructor(private httpClient: HttpClient) {}
    // ↑ Angular inietta HttpClient automaticamente

    public setPickAnswer(xTenant: string, pickAnswerDTO: PickAnswerDTO): Observable<any> {
        return this.httpClient.request<any>('put', `${this.configuration.basePath}/api/survey/answer`, {
            body: pickAnswerDTO,
            headers: { 'X-Tenant': xTenant },
            // ...
        });
    }
}
```

### Observable — il cuore di Angular per le chiamate asincrone

`Observable<T>` è il tipo restituito da quasi tutte le operazioni asincrone in Angular. È il meccanismo di **RxJS (Reactive Extensions for JavaScript)** — una libreria per la programmazione reattiva.

Pensa a un `Observable` come a un "flusso di valori nel tempo". Una chiamata HTTP è un flusso con un solo valore (la risposta) che arriva dopo un certo tempo.

**La differenza chiave con le Promise:**

| | Promise | Observable |
|---|---|---|
| Valori | Uno solo | Potenzialmente infiniti |
| Avvio | Si esegue subito | Si esegue solo quando qualcuno si "iscrive" (`subscribe`) |
| Annullamento | Non cancellabile | Cancellabile (`unsubscribe`) |
| Operatori | `.then()`, `.catch()` | `map`, `filter`, `mergeMap`, ecc. (potentissimi) |

Un `Observable` che non viene mai "sottoscritto" non fa mai la chiamata HTTP. Questo è diverso da JavaScript puro dove una `Promise` parte subito.

### .subscribe() — come ricevere i valori

```typescript
this.answerApiService.setPickAnswer(tenant, dto).subscribe({
    next: (result) => {
        // chiamata riuscita — result è la risposta del server
        console.log('Risposta salvata');
    },
    error: (err) => {
        // chiamata fallita — err contiene i dettagli dell'errore
        this.somethingWentWrong = true;
    },
    complete: () => {
        // il flusso è terminato (per le chiamate HTTP, arriva dopo next)
    }
});
```

`subscribe` accetta un oggetto con tre callback opzionali:
- `next`: chiamato ogni volta che arriva un valore (per le chiamate HTTP, una volta sola)
- `error`: chiamato se arriva un errore (l'HTTP call fallisce, timeout, 4xx/5xx)
- `complete`: chiamato quando il flusso termina normalmente (dopo `next` nelle chiamate HTTP)

### Observable → Promise con async/await

Nel componente del fix, si converte l'Observable in Promise per poter usare `await`:

```typescript
async setPickAnswer(questionId: string, newChoiceId: string | null) {
    const dto: PickAnswerDTO = { quizId: this.quizId, questionId, newChoiceId };
    
    // Creo una Promise che wrappa l'Observable
    const promise = new Promise<void>((resolve, reject) => {
        this.answerApiService.setPickAnswer(this.tenant, dto).subscribe({
            next: () => resolve(),   // l'Observable ha emesso il valore → la Promise si risolve
            error: (err) => reject(err) // errore → la Promise viene rifiutata
        });
    });
    
    await promise; // aspetta che la chiamata HTTP finisca prima di continuare
}
```

Perché questa conversione? Perché `handleRadioButton` usa `async/await` per sequenziare operazioni asincrone in modo leggibile. Senza `await`, le chiamate HTTP partirebbero tutte in parallelo senza attendere la risposta della precedente.

---

## 12. Helpers Angular

### Cosa si intende per "helper" in Angular

In Angular non esiste un concetto ufficiale chiamato "helper" — è un termine generico che si usa per indicare **metodi di supporto** che semplificano la logica all'interno di un componente o di un service. Nel contesto del fix, `setPickAnswer` viene chiamato helper perché:

- Non è chiamato direttamente dall'utente (non è legato a un click diretto nel template HTML)
- Incapsula un'operazione ripetuta (costruire il DTO + chiamare il service + gestire errori)
- È riusato da più punti del componente (`handleRadioButton`, `applyAvoidDuplicatesRule`, `isValueDuplicatedInGroup`)

Puoi pensarlo come un metodo privato di supporto (in Java si chiamerebbe `private`).

### Pattern — costruisci DTO, chiama service, gestisci errore

Il pattern ricorrente in Angular per fare chiamate al backend è sempre questo:

```typescript
// 1. Metodo async (può usare await)
async setPickAnswer(questionId: string, newChoiceId: string | null) {

    // 2. Costruisci il DTO con i dati correnti del componente
    const pickAnswerDto: PickAnswerDTO = {
        quizId: this.quizId,       // ← this = il componente stesso
        questionId: questionId,
        newChoiceId: newChoiceId
    };

    // 3. Mostra uno stato di caricamento (opzionale)
    this.disableAnimationService.setDisableAnimation(true);

    // 4. Crea una Promise che wrappa l'Observable del service
    const promise = new Promise<void>((resolve, reject) => {
        if (this.tenant) {  // ← guardia: assicurati che il tenant sia disponibile
            this.answerApiService
                .setPickAnswer(this.tenant, pickAnswerDto)
                .subscribe({
                    next: () => resolve(),
                    error: (error) => {
                        this.somethingWentWrong = true; // ← mostra errore all'utente
                        reject(error);
                    }
                });
        }
    });

    // 5. Aspetta che la chiamata finisca (blocca l'esecuzione di questo metodo)
    await promise;

    // 6. Il codice qui viene eseguito DOPO che il server ha risposto
}
```

### this — il componente e i suoi campi

In TypeScript (e JavaScript), `this` all'interno di un metodo di classe si riferisce all'istanza della classe — nel contesto di un componente Angular, è il componente stesso. Puoi accedere a qualsiasi campo o metodo del componente con `this.nomeCampo`.

```typescript
export class GruppoDomandeMotoreRenderingComponent {
    quizId: string = '';           // ← field del componente
    tenant: string | null = null;  // ← field del componente
    somethingWentWrong = false;    // ← field del componente

    constructor(
        private answerApiService: AnswerApiService  // ← iniettato da Angular
    ) {}

    async setPickAnswer(questionId: string, newChoiceId: string | null) {
        // Accede ai campi del componente tramite this
        const dto: PickAnswerDTO = {
            quizId: this.quizId,  // ← legge quizId del componente
            // ...
        };
        
        if (this.tenant) {  // ← legge tenant del componente
            this.answerApiService.setPickAnswer(this.tenant, dto).subscribe({
                error: () => {
                    this.somethingWentWrong = true;  // ← scrive sul campo del componente
                }
            });
        }
    }
}
```

### async/await in Angular — perché è importante

Senza `async/await`, due operazioni asincrone sequenziali andrebbero in parallelo:

```typescript
// PROBLEMA — senza await (le due chiamate partono in parallelo)
async handleRadioButton(questionId: string, value: string) {
    // Ricerca del duplicato + deseleziona
    this.isValueDuplicatedInGroup(questionId, value); // ← parte ma non aspetta
    
    // Imposta la nuova risposta
    this.setPickAnswer(questionId, choiceId); // ← parte subito, senza aspettare il risultato sopra
}

// SOLUZIONE — con await (sequenziale, ognuna aspetta la precedente)
async handleRadioButton(questionId: string, value: string) {
    await this.isValueDuplicatedInGroup(questionId, value); // ← aspetta che finisca
    
    await this.setPickAnswer(questionId, choiceId); // ← poi parte questa
}
```

`await` può essere usato solo all'interno di una funzione dichiarata `async`. Una funzione `async` restituisce sempre una `Promise` (automaticamente), anche se non hai scritto `return new Promise(...)`.

### Observer pattern — subscribe() spiegato in profondità

`subscribe()` implementa il **pattern Observer**: hai un "produttore di valori" (l'Observable — la chiamata HTTP) e uno o più "consumatori" (gli observer — le callback che hai passato a subscribe).

```typescript
// Equivalente concettuale in Java/Spring (pseudocodice)
Future<Risposta> future = httpClient.put("/api/survey/answer", dto);
future.thenAccept(risposta -> {
    // chiamata riuscita
}).exceptionally(err -> {
    // errore
    return null;
});
```

La differenza fondamentale con una Promise JavaScript è che puoi **annullare** un Observable:

```typescript
// Salva il "subscription" quando ti iscrivi
const subscription = this.answerApiService.setPickAnswer(tenant, dto).subscribe({
    next: () => { ... }
});

// In seguito, se vuoi cancellare la chiamata (es: l'utente ha navigato via):
subscription.unsubscribe();
```

In Angular è buona pratica disiscriversì dagli Observables a lunga vita quando il componente viene distrutto (nel metodo `ngOnDestroy()`), per evitare memory leak. Le chiamate HTTP si completano da sole e non richiedono unsubscribe manuale, ma altri tipi di Observable (timer, WebSocket, store) sì.

### Template del componente — come gli helpers si collegano all'HTML

Un componente Angular ha tre parti:

```typescript
// componente.ts — logica
@Component({
    selector: 'app-gruppo-domande',
    templateUrl: './gruppo-domande.component.html',  // ← punta al template
    styleUrls: ['./gruppo-domande.component.scss']
})
export class GruppoDomandeMotoreRenderingComponent {
    somethingWentWrong = false;
    
    // Questo metodo viene chiamato dall'HTML quando l'utente clicca una risposta
    async handleRadioButton(questionId: string, value: string) {
        // ...
        await this.setPickAnswer(questionId, choiceId);
    }
    
    // Helper — non chiamato direttamente dall'HTML
    async setPickAnswer(questionId: string, newChoiceId: string | null) {
        // ...
    }
}
```

```html
<!-- componente.html — template -->
<div *ngIf="somethingWentWrong">
    <!-- Mostra il messaggio di errore se somethingWentWrong è true -->
    <p>Si è verificato un errore</p>
</div>

<input type="radio" (change)="handleRadioButton(question.id, $event.target.value)">
<!-- ↑ (change) è un event binding: quando l'utente cambia la radio,
       chiama handleRadioButton nel componente -->
```

Il legame tra template e logica TypeScript è il **data binding** di Angular:
- `{{ espressione }}` — interpolazione: mostra il valore di una variabile
- `[proprietà]="espressione"` — property binding: imposta una proprietà HTML
- `(evento)="metodo()"` — event binding: chiama un metodo quando scatta l'evento
- `*ngIf="condizione"` — direttiva strutturale: mostra/nasconde un elemento
- `*ngFor="let item of lista"` — direttiva strutturale: ripete un elemento per ogni item

---

## Riepilogo visivo — Come tutto si collega

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

## Note personali

- **`@Transactional` su metodi privati non funziona** — Spring crea un proxy attorno all'oggetto, e i proxy non intercettano le chiamate a metodi privati. Metti sempre `@Transactional` su metodi `public`.

- **Optional non va usato come parametro di un metodo** — `Optional` è pensato per i valori di ritorno che "potrebbero non esserci". Come parametro di metodo è semanticamente sbagliato (passare `Optional.empty()` invece di `null` non migliora la chiarezza). Se un parametro può mancare, usa un normale `null` check o un overload del metodo.

- **Observable è "lazy", Promise è "eager"** — creare un `Observable` non fa partire la chiamata HTTP. Devi chiamare `subscribe()`. Questo è il motivo per cui puoi costruire catene di operatori (`.pipe(map(...), filter(...))`) prima della sottoscrizione senza fare nessuna chiamata di rete.

- **`this` nelle arrow function** — in JavaScript classico, il valore di `this` all'interno di una callback dipende da come la callback è chiamata. Le **arrow function** (`() => { ... }`) catturano il `this` del contesto circostante. Per questo le callback di `subscribe()` sono scritte come arrow function: `error: (err) => { this.somethingWentWrong = true }` — `this` si riferisce al componente Angular, non all'Observable.

- **Getter Lombok e JPA** — JPA usa i getter per accedere ai campi delle Entity durante la serializzazione e la gestione delle relazioni lazy. Se usi `@Getter` di Lombok sull'Entity, funziona perfettamente. Se rimuovi accidentalmente `@Getter`, Hibernate potrebbe avere comportamenti inattesi.

- **Il Service Angular è un singleton** — Angular crea una sola istanza del service per tutta l'applicazione (se usi `providedIn: 'root'`). Questo significa che puoi usarlo per condividere stato tra componenti diversi. Ma attenzione: se memorizzi dati nel service, rimangono in memoria anche quando navighi su un'altra pagina.
