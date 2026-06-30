# Concetti — Angular e TypeScript

> **Categoria:** `Angular / TypeScript / Frontend / DevExtreme`
> **Aggiornato:** `30/06/2026`

---

## Indice

1. [Angular — Il framework](#1-angular--il-framework)
2. [Angular CLI](#2-angular-cli)
3. [DevExtreme UI](#3-devextreme-ui)
4. [TypeScript — Fondamentali](#4-typescript--fondamentali)
5. [Model Angular (.ts)](#5-model-angular-ts)
6. [Service Angular](#6-service-angular)
7. [Helpers e pattern dei Componenti](#7-helpers-e-pattern-dei-componenti)

---

## 1. Angular — Il framework

**Angular** è un framework per costruire applicazioni web lato client (frontend). È sviluppato da Google e scritto in **TypeScript** (un superset di JavaScript che aggiunge la tipizzazione statica). È il framework scelto per tutti i frontend della piattaforma LISA.

L'idea fondamentale di Angular è il modello **component-based**: l'interfaccia utente è suddivisa in componenti riutilizzabili. Ogni componente ha un template HTML (cosa mostrare), un file TypeScript (la logica), e un file CSS (lo stile). I componenti si compongono ad albero per formare l'intera applicazione.

Angular include anche:
- **Routing**: naviga tra "pagine" senza ricaricare il browser (Single Page Application)
- **Servizi e Dependency Injection**: logica condivisa (chiamate HTTP, gestione stato) separata dai componenti
- **Reactive Forms**: gestione avanzata dei form con validazione
- **HttpClient**: modulo per fare chiamate REST al backend

Il frontend del Survey (`lisa-survey`) è un'applicazione Angular che gira in locale su `http://localhost:4200`. Quando fai il login, Angular riceve il codice di autorizzazione da `authorization-dev.lisasuite.it` al callback `http://localhost:4200/callback` e poi scambia il codice per un token JWT, che include automaticamente in ogni chiamata HTTP al backend Survey tramite un **interceptor**.

---

## 2. Angular CLI

**Angular CLI** (Command Line Interface) è lo strumento a riga di comando ufficiale per lavorare con Angular. Permette di:

- **Creare** nuovi progetti, componenti, servizi con struttura corretta: `ng generate component nome`
- **Compilare** il codice TypeScript in JavaScript ottimizzato: `ng build`
- **Avviare** il server di sviluppo locale con hot-reload: `ng serve`

Il comando `npx ng serve --port 4200` avvia il server di sviluppo:
- Compila il codice TypeScript
- Avvia un server HTTP locale
- Osserva i file sorgente: ogni volta che salvi una modifica, ricompila automaticamente e aggiorna il browser (questa funzione si chiama **hot-reload** o **live reload**)

`npx` è un comando di Node.js che permette di eseguire strumenti installati localmente nel progetto senza doverli installare globalmente. Si usa al posto di `ng` direttamente perché PowerShell blocca l'esecuzione di script `.ps1` di terze parti per policy di sicurezza.

---

## 3. DevExtreme UI

**DevExtreme** è una libreria di componenti UI (User Interface) per Angular (e altri framework) sviluppata da DevExpress. Fornisce componenti pronti all'uso e altamente personalizzabili per costruire interfacce enterprise: griglie dati, grafici, form, calendar, scheduler, ecc.

Il componente più usato nei progetti LISA è probabilmente `dxDataGrid`: una griglia dati estremamente potente con ordinamento, filtraggio, paginazione, raggruppamento, export in Excel, editing inline, tutto incluso.

DevExtreme è una libreria **commerciale** (a pagamento), ma molto usata nel mondo enterprise proprio perché offre componenti già pronti per casi d'uso complessi che richiederebbero settimane di sviluppo custom. La vedi nel progetto come dipendenza `devextreme` e `devextreme-angular` nel `package.json`.

---

## 4. TypeScript — Fondamentali

### Cos'è TypeScript

**TypeScript** estende JavaScript aggiungendo la tipizzazione statica. Quando scrivi codice TypeScript, lo **compili** in JavaScript puro (che il browser capisce). TypeScript non esiste a runtime — serve solo a te durante lo sviluppo.

Il vantaggio principale: il compilatore TypeScript ti dice in anticipo (mentre scrivi, nell'IDE) se stai usando un tipo sbagliato. In JavaScript puro, molti errori si scoprono solo a runtime.

```typescript
// JavaScript — nessun errore a compile time
function somma(a, b) { return a + b; }
somma("5", 3); // → "53" (concatena string e number!) — errore silenzioso

// TypeScript — errore a compile time
function somma(a: number, b: number): number { return a + b; }
somma("5", 3); // ← ERRORE: Argument of type 'string' is not assignable to parameter of type 'number'
```

### Interface vs Class

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

In TypeScript `null` e `undefined` sono tipi distinti. `null` significa "valore esplicitamente assente", `undefined` significa "campo non inizializzato / non presente". Nel DTO del Survey, `null` ha un significato semantico preciso: "deseleziona la risposta senza inserirne una nuova".

---

## 5. Model Angular (.ts)

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

## 6. Service Angular

### Teoria

In Angular, un **Service** è una classe con il decoratore `@Injectable` che contiene logica condivisa tra componenti. I Service sono il meccanismo standard per:

- Fare chiamate HTTP al backend
- Condividere stato tra componenti
- Contenere logica di business che non appartiene a un singolo componente

A differenza dei **componenti** (che hanno un template HTML e riguardano la visualizzazione), i service non hanno interfaccia grafica. Sono singleton: Angular ne crea una sola istanza e la condivide tra tutti i componenti che la richiedono.

### @Injectable — il decoratore

```typescript
@Injectable({
    providedIn: 'root'  // ← singleton per tutta l'applicazione
})
export class AnswerApiService {
    // ...
}
```

`@Injectable({ providedIn: 'root' })` registra il service nell'**Injector** di Angular (equivalente al container IoC di Spring). Quando un componente dichiara di aver bisogno di questo service nel costruttore, Angular lo fornisce automaticamente.

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

Un `Observable` che non viene mai "sottoscritto" non fa mai la chiamata HTTP.

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

## 7. Helpers e pattern dei Componenti

### Cosa si intende per "helper" in Angular

In Angular non esiste un concetto ufficiale chiamato "helper" — è un termine generico che si usa per indicare **metodi di supporto** che semplificano la logica all'interno di un componente o di un service. Un metodo helper:

- Non è chiamato direttamente dall'utente (non è legato a un click diretto nel template HTML)
- Incapsula un'operazione ripetuta (costruire il DTO + chiamare il service + gestire errori)
- È riusato da più punti del componente

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
    this.isValueDuplicatedInGroup(questionId, value); // ← parte ma non aspetta
    this.setPickAnswer(questionId, choiceId); // ← parte subito, senza aspettare il risultato sopra
}

// SOLUZIONE — con await (sequenziale, ognuna aspetta la precedente)
async handleRadioButton(questionId: string, value: string) {
    await this.isValueDuplicatedInGroup(questionId, value); // ← aspetta che finisca
    await this.setPickAnswer(questionId, choiceId); // ← poi parte questa
}
```

`await` può essere usato solo all'interno di una funzione dichiarata `async`. Una funzione `async` restituisce sempre una `Promise` (automaticamente), anche se non hai scritto `return new Promise(...)`.

### Template del componente — come gli helpers si collegano all'HTML

Un componente Angular ha tre parti:

```typescript
// componente.ts — logica
@Component({
    selector: 'app-gruppo-domande',
    templateUrl: './gruppo-domande.component.html',
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
    <p>Si è verificato un errore</p>
</div>

<input type="radio" (change)="handleRadioButton(question.id, $event.target.value)">
```

Il legame tra template e logica TypeScript è il **data binding** di Angular:
- `{{ espressione }}` — interpolazione: mostra il valore di una variabile
- `[proprietà]="espressione"` — property binding: imposta una proprietà HTML
- `(evento)="metodo()"` — event binding: chiama un metodo quando scatta l'evento
- `*ngIf="condizione"` — direttiva strutturale: mostra/nasconde un elemento
- `*ngFor="let item of lista"` — direttiva strutturale: ripete un elemento per ogni item

### Observer pattern — subscribe() spiegato in profondità

`subscribe()` implementa il **pattern Observer**: hai un "produttore di valori" (l'Observable — la chiamata HTTP) e uno o più "consumatori" (gli observer — le callback che hai passato a subscribe).

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

---

## Note personali

- **Observable è "lazy", Promise è "eager"** — creare un `Observable` non fa partire la chiamata HTTP. Devi chiamare `subscribe()`. Questo è il motivo per cui puoi costruire catene di operatori (`.pipe(map(...), filter(...))`) prima della sottoscrizione senza fare nessuna chiamata di rete. La Promise invece parte subito alla creazione.

- **`this` nelle arrow function** — in JavaScript classico, il valore di `this` all'interno di una callback dipende da come la callback è chiamata. Le **arrow function** (`() => { ... }`) catturano il `this` del contesto circostante. Per questo le callback di `subscribe()` sono scritte come arrow function: `error: (err) => { this.somethingWentWrong = true }` — `this` si riferisce al componente Angular, non all'Observable.

- **Il Service Angular è un singleton** — Angular crea una sola istanza del service per tutta l'applicazione (se usi `providedIn: 'root'`). Questo significa che puoi usarlo per condividere stato tra componenti diversi. Ma attenzione: se memorizzi dati nel service, rimangono in memoria anche quando navighi su un'altra pagina.
