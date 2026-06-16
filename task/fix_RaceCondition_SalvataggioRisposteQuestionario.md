# Fix Race Condition — Salvataggio Risposte Questionario STEP-33+

> **Categoria:** `Spring Boot / Angular / REST API / Concorrenza`
> **Data:** `15/06/2026`
> **Difficoltà percepita:** ⭐⭐⭐

---

## Problema

Quando un candidato svolgeva il questionario **STEP-33+** e cambiava risposta rapidamente sulla stessa domanda, l'applicazione andava in crash mostrando la modal di errore (`somethingWentWrong`).

### Causa radice — Race Condition

Analizzando il tab **Network** dei DevTools, ogni cambio di risposta generava **due chiamate HTTP POST** sequenziali:

1. `POST /api/survey/answer` con `pickAction: "UNSELECT"` — cancellava la risposta precedente
2. `POST /api/survey/answer` con `pickAction: "SELECT"` — inseriva la nuova risposta

Il problema: `handleRadioButton` è una funzione `async` non protetta da accesso concorrente. Se l'utente cliccava rapidamente su due risposte diverse, **due invocazioni della funzione giravano in parallelo** condividendo la stessa mappa locale delle risposte:

```
Invocazione 1 (click su B)   |   Invocazione 2 (click su C)
legge currentSelected = A    |   legge currentSelected = A  ← mappa non ancora aggiornata!
UNSELECT(A) → in volo        |   UNSELECT(A) → in volo      ← A già cancellata dal primo!
SELECT(B)                    |   SELECT(C)
```

Il secondo `UNSELECT(A)` arrivava al backend dopo che il primo aveva già cancellato la riga → il backend lanciava `IllegalStateException("Answer not submitted")` → crash.

---

## Soluzione — API Atomica (PUT)

Invece di due chiamate separate (UNSELECT + SELECT), è stata introdotta **una singola chiamata `PUT`** che dice al backend: *"per questa domanda, la risposta è ora questa choice"*. Il backend gestisce la transizione atomicamente in un'unica transazione `@Transactional`.

```
PRIMA (2 POST, race condition possibile):
Frontend → POST UNSELECT → Backend (cancella vecchia risposta)
Frontend → POST SELECT   → Backend (inserisce nuova risposta)

DOPO (1 PUT, atomico):
Frontend → PUT { questionId, newChoiceId } → Backend (cancella vecchia E inserisce nuova in una transazione sola)
```

---

## Implementazione

### BACKEND — Spring Boot

#### 1. Nuovo DTO — `PickAnswerDTO.java`

Creato in `survey/src/main/java/it/limontainformatica/lisasurvey/dto/`:

```java
@Getter
@Setter
public class PickAnswerDTO {
    private String quizId;
    private String questionId;
    private String newChoiceId; // null = solo deselect, senza nuova selezione
}
```

A differenza del vecchio `PhaseAnswersDTO`, non contiene più `pickAction` (SELECT/UNSELECT) — la logica di cosa cancellare e cosa inserire è delegata interamente al backend.

#### 2. Nuovo metodo Repository — `AnswerRepository.java`

Aggiunto in `survey/src/main/java/it/limontainformatica/lisasurvey/repository/`:

```java
Optional<Answer> findByAttemptQuizIdAndQuestionId(String attemptQuizId, String questionId);
```

Spring Data JPA genera automaticamente la query SQL dal nome del metodo. Restituisce `Optional<Answer>` perché la risposta potrebbe non esistere ancora (domanda non ancora risposta dal candidato).

#### 3. Nuovo metodo Service — `AnswerService.java`

Aggiunto in `survey/src/main/java/it/limontainformatica/lisasurvey/service/`:

```java
@Transactional
public void setPickAnswer(PickAnswerDTO dto) {
    Quiz quiz = quizRepo.findById(dto.getQuizId())
        .orElseThrow(() -> new EntityNotFoundException("Quiz [" + dto.getQuizId() + "]"));
    Question question = questionRepo.findById(dto.getQuestionId())
        .orElseThrow(() -> new EntityNotFoundException("Question [" + dto.getQuestionId() + "]"));

    AttemptQuiz attemptQuizInProgress = attemptService.getAttemptQuizInProgress(quiz);
    AttemptPhase attemptPhaseInProgress = attemptService.getAttemptPhaseInProgress(quiz);

    // 1. Cerca risposta esistente per questa domanda in questo tentativo
    Optional<Answer> existingAnswer = answerRepo.findByAttemptQuizIdAndQuestionId(
        attemptQuizInProgress.getId(), dto.getQuestionId()
    );
    // 2. Se esiste, cancellala
    existingAnswer.ifPresent(answerRepo::delete);

    // 3. Se c'è una nuova scelta, inseriscila
    if (dto.getNewChoiceId() != null) {
        Choice choice = choiceRepo.findById(dto.getNewChoiceId())
            .orElseThrow(() -> new EntityNotFoundException("Choice [" + dto.getNewChoiceId() + "]"));
        Answer answer = new Answer();
        answer.setAttemptQuizId(attemptQuizInProgress.getId());
        answer.setAttemptPhaseId(attemptPhaseInProgress.getId());
        answer.setCompanyId(TenantContext.currentTenant());
        answer.setQuestionId(dto.getQuestionId());
        answer.setChoiceId(choice.getId());
        answerRepo.save(answer);
    }
}
```

L'annotazione `@Transactional` garantisce che delete + save avvengano come un'unica operazione atomica: se qualcosa va storto a metà, tutto viene annullato.

#### 4. Nuovo endpoint Controller — `AnswerAPI.java`

Aggiunto in `survey/src/main/java/it/limontainformatica/lisasurvey/api/`:

```java
@PutMapping
public ResponseEntity<Void> setPickAnswer(
        @RequestHeader("X-Tenant") String tenant,
        @RequestParam(name = "lang", required = false, defaultValue = "IT") Language lang,
        @RequestBody PickAnswerDTO dto
) {
    log.info("PUT /api/survey/answer - Set pick answer invoked");
    tenantDataFilter.setFilter();
    answerService.setPickAnswer(dto);
    return ResponseEntity.ok().build();
}
```

Usa `@PutMapping` invece di `@PostMapping` perché la semantica HTTP di PUT è "sostituisci con questo" — coerente con l'operazione di sostituzione della risposta.

---

### FRONTEND — Angular

#### 5. Nuovo modello TypeScript — `pickAnswerDTO.ts`

Creato in `lisa-survey/src/app/api/v1/model/`:

```typescript
export interface PickAnswerDTO {
    quizId?: string;
    questionId?: string;
    newChoiceId?: string | null; // string | null: può essere esplicitamente null per il deselect
}
```

Aggiunto il re-export in `models.ts`:
```typescript
export * from './pickAnswerDTO';
```

#### 6. Nuovo metodo nel Service Angular — `answerApi.service.ts`

Aggiunto il metodo `setPickAnswer` che chiama `PUT /api/survey/answer`:

```typescript
public setPickAnswer(xTenant: string, pickAnswerDTO: PickAnswerDTO, ...): Observable<any> {
    // ... setup headers identico al metodo submitPhaseAnswer esistente ...
    return this.httpClient.request<any>('put', `${this.configuration.basePath}/api/survey/answer`, {
        body: pickAnswerDTO,
        // ...
    });
}
```

#### 7. Nuovo helper nel componente — `gruppo-domande-motore-rendering.component.ts`

Aggiunto `setPickAnswer(questionId, newChoiceId)` che costruisce il DTO e gestisce la Promise:

```typescript
async setPickAnswer(questionId: string, newChoiceId: string | null) {
    const pickAnswerDto: PickAnswerDTO = {
        quizId: this.quizId,
        questionId: questionId,
        newChoiceId: newChoiceId
    };
    this.disableAnimationService.setDisableAnimation(true);
    const promise = new Promise<void>((resolve, reject) => {
        if (this.tenant) {
            this.answerApiService.setPickAnswer(this.tenant, pickAnswerDto).subscribe({
                next: () => resolve(),
                error: (error) => {
                    this.somethingWentWrong = true;
                    reject(error);
                }
            });
        }
    });
    await promise;
}
```

#### 8. Modifica `handleRadioButton`

Il metodo è stato semplificato: al posto della coppia UNSELECT+SELECT, una singola chiamata `setPickAnswer`.

```typescript
// PRIMA
if (currentSelected) {
    const previousChoiceId = this.getChoiceIdForSelectedValue(...);
    await this.updateAnswerApi(previousChoiceId, 'UNSELECT');
}
groupMap.set(questionId, value);
await this.updateAnswerApi(choiseId, 'SELECT');

// DOPO
groupMap.set(questionId, value);
await this.setPickAnswer(questionId, choiseId);
```

Caso deselect (click su risposta già selezionata):
```typescript
// PRIMA
await this.updateAnswerApi(choiseId, 'UNSELECT');

// DOPO
await this.setPickAnswer(questionId, null); // null = nessuna nuova selezione
```

#### 9. Modifica `applyAvoidDuplicatesRule` e `isValueDuplicatedInGroup`

STEP-33+ usa la regola `AVOID_DUPLICATES` (logica "terzine"): ogni valore può essere assegnato a una sola domanda per gruppo. Anche questi metodi usavano il vecchio pattern SELECT/UNSELECT.

In `isValueDuplicatedInGroup`, la rimozione del duplicato ora usa:
```typescript
// PRIMA
await this.updateAnswerApi(choiceId, 'UNSELECT');

// DOPO
await this.setPickAnswer(question.id, null);
```

In `applyAvoidDuplicatesRule`, entrambi i rami sono stati semplificati eliminando il blocco `if (currentSelected)` con la ricerca del `previousChoiceId`:
```typescript
// entrambi i rami ora terminano con:
groupMap.set(questionId, value);
await this.setPickAnswer(questionId, choiseId);
```

---

## Chiamate HTTP durante l'operatività

### Selezione normale (nessuna risposta precedente)

```
PUT /api/survey/answer
Headers: X-Tenant: <tenant>
Body: {
  "quizId": "<id>",
  "questionId": "<id>",
  "newChoiceId": "<id>"
}
→ 200 OK
```

### Cambio risposta (sostituzione)

```
PUT /api/survey/answer
Body: { "quizId": "...", "questionId": "...", "newChoiceId": "<nuova_choice>" }
→ Backend: DELETE vecchia risposta + INSERT nuova (1 transazione)
→ 200 OK
```

### Deseleziona (click su risposta già selezionata)

```
PUT /api/survey/answer
Body: { "quizId": "...", "questionId": "...", "newChoiceId": null }
→ Backend: DELETE risposta esistente, nessun INSERT
→ 200 OK
```

### Caso AVOID_DUPLICATES — logica terzine (2 PUT sequenziali)

Scenario: valore "2" già selezionato sulla domanda B, utente seleziona "2" sulla domanda A.

```
1ª PUT /api/survey/answer
Body: { "quizId": "...", "questionId": "<B.id>", "newChoiceId": null }
→ Rimuove il "2" dalla domanda B
→ 200 OK

2ª PUT /api/survey/answer
Body: { "quizId": "...", "questionId": "<A.id>", "newChoiceId": "<choiceId_del_2_in_A>" }
→ Imposta il "2" sulla domanda A (cancella eventuale risposta precedente di A)
→ 200 OK
```

---

## Errori che ho fatto

- **Errore:** Ho creato una seconda `public interface AnswerOptionRepository` dentro il file `AnswerRepository.java`.
  **Soluzione:** In Java ogni `public` type deve stare nel suo file con lo stesso nome. Il metodo andava aggiunto dentro l'interfaccia già esistente.

- **Errore:** Ho usato `findAllByAttemptQuizIdAndAttemptPhaseId` invece del nuovo metodo `findByAttemptQuizIdAndQuestionId` nella query del service.
  **Soluzione:** Ho usato il metodo corretto puntando al `questionId` invece dell'`attemptPhaseId`.

- **Errore:** Ho usato `isPresent(answer -> ...)` invece di `ifPresent(answer -> ...)`.
  **Soluzione:** `isPresent()` non accetta argomenti e restituisce solo `boolean`. `ifPresent()` accetta un Consumer ed esegue l'azione solo se il valore è presente.

- **Errore:** Nel service Angular ho usato `PhaseAnswersDTO` come tipo del parametro nelle overload di `setPickAnswer` invece di `PickAnswerDTO`.
  **Soluzione:** Aggiornate tutte e quattro le firme del metodo con il tipo corretto.

- **Errore:** Ho modificato solo `handleRadioButton` ma STEP-33+ usa la regola `AVOID_DUPLICATES`, che passa per `applyAvoidDuplicatesRule` e `isValueDuplicatedInGroup` — ancora sul vecchio pattern SELECT/UNSELECT.
  **Soluzione:** Aggiornati anche questi due metodi.

---

## Concetti collegati

- `@Transactional` in Spring Boot: garantisce atomicità — se il metodo fallisce a metà, tutte le operazioni DB vengono annullate (rollback).
- `Optional<T>` in Java: contenitore per valori che possono essere assenti. `.ifPresent(fn)` esegue la funzione solo se il valore c'è. `.orElseThrow()` lancia un'eccezione se il valore è assente.
- Method reference Java (`answerRepo::delete`): sintassi compatta equivalente a `answer -> answerRepo.delete(answer)`.
- Race condition: bug che si verifica quando due operazioni concorrenti modificano lo stesso stato condiviso in un ordine non deterministico.
- Semantica HTTP PUT vs POST: POST = "crea una nuova risorsa", PUT = "sostituisci la risorsa con questo" — PUT è naturalmente idempotente (chiamarlo più volte produce lo stesso risultato).
- Collegato a: `..\concetti\concetti_stack_LISA_Survey.md`
