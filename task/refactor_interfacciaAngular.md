# Fix: Ricerca Utenti nel Capitolato — Migrazione GWT → Angular

## Contesto

Nell'applicazione LISA il form di dettaglio capitolato (`FrmCapitolatoDettaglio`) permette di
associare una **Persona di Riferimento** (campo `TSPERR`). Questa viene selezionata tramite una
dialog di ricerca che carica utenti di sistema (tabella `JAVAARCH.USERS`), distinti dai clienti
anagrafici (`ANCL200F`).

La dialog di ricerca utenti è stata migrata dal frontend GWT legacy (`FrmUtenti.java`) al
componente Angular `UtenteLookupDialogComponent`. La migrazione AI ha replicato correttamente
la struttura visiva ma ha introdotto un bug funzionale che rendeva la ricerca sempre vuota.

---

## Logica legacy GWT (corretta)

### Flusso completo

1. L'utente clicca `btnCercaUtente` in `FrmCapitolatoDettaglio` → si apre `FrmUtenti` come
   dialog modale.
2. L'utente inserisce facoltativamente un ID o un NOME e clicca "Ricerca".
3. Viene invocato il metodo RPC:

   ```java
   // IServiceCapitolatiResource.java
   @POST
   @Path("caricautenti/{servicekey}/{authToken}")
   void caricaUtenti(
       @PathParam("servicekey") String serviceKey,
       @PathParam("authToken")  String authToken,
       DOUtente filtro,
       MethodCallback<List<DOUtente>> records
   );
   ```

4. Il filtro `DOUtente` viene costruito così in `FrmUtenti.java`:

   ```java
   DOUtente filtro = new DOUtente();
   filtro.ID   = ID.getText();    // testo digitato dall'utente (può essere vuoto)
   filtro.NOME = NOME.getText();  // testo digitato dall'utente (può essere vuoto)

   // CAMPO CRITICO — logica derivata dall'utente loggato
   if (Lisa3Service.INSTANCE.USER.startsWith("LT")) {
       filtro.AZIENDA = "LT";
   } else {
       filtro.AZIENDA = "SP2";
   }
   ```

5. Il server (`CapitolatiResource.java`) riceve il body e chiama:

   ```java
   // Il server usa SOLO il campo AZIENDA — ID e NOME vengono ignorati
   ArrayList<DOUtente> caricaElenco =
       userManagement.eseguiRicercaPerAzienda(session, readValue.AZIENDA, false);
   ```

   La query eseguita è equivalente a:
   ```sql
   SELECT * FROM JAVAARCH.USERS
   WHERE AZIENDA   = :azienda
     AND ANNULLATO <> 'A'
   ORDER BY ID
   ```

6. Il risultato viene mostrato nella griglia. L'utente seleziona una riga e clicca "Ok".
7. L'evento `EVENT_RICERCA_UTENTE` viene lanciato. `FrmCapitolatoDettaglio` lo intercetta e
   scrive `selectedUser.ID` nel campo `TSPERR`.

### Punto chiave

Il server **ignora completamente** `ID` e `NOME` inviati nel body. L'unico campo che determina
i risultati è **`AZIENDA`**, ricavato lato client dal prefisso dell'username loggato.

---

## Perché non funzionava su Angular

### Implementazione originale (errata)

```typescript
// utente-lookup-dialog.component.ts — metodo cerca() originale
cerca(): void {
  const id   = this.filtroId.trim();
  const nome = this.filtroNome.trim();

  if (!id && !nome) {
    this.messages.add({ severity: 'warn', detail: 'Inserire ID o nome utente.' });
    return;
  }

  this.api.caricaUtenti({
    ...(id   && { ID:   '%' + id }),   // inviato ma il server lo ignora
    ...(nome && { NOME: '%' + nome }), // inviato ma il server lo ignora
    // AZIENDA mai inviato!
  }).subscribe(...);
}
```

### Causa del bug

Il campo `AZIENDA` non veniva mai incluso nel body della richiesta. Il server riceveva quindi
`AZIENDA = ""` (o campo assente) e la query diventava:

```sql
WHERE AZIENDA = ''   -- nessun utente corrisponde
```

Il risultato era sempre una lista vuota, indipendentemente da cosa l'utente digitasse nei campi
ID e NOME.

**Ulteriore conseguenza**: la validazione "Inserire ID o nome utente" rendeva impossibile
effettuare una ricerca senza filtri, mentre il GWT non aveva questa restrizione (i filtri sono
facoltativi e il server comunque ignorava ID e NOME).

---

## Come è stato risolto

### 1. `auth.service.ts` — aggiunto computed signal `azienda`

```typescript
/** Codice azienda derivato dal prefisso username, replica logica GWT FrmUtenti. */
readonly azienda = computed(() =>
  this._user().startsWith('LT') ? 'LT' : 'SP2'
);
```

Nessun dato aggiuntivo da persistere: il valore si calcola sempre dall'username già in sessione,
esattamente come faceva il GWT.

### 2. `utente-lookup-dialog.component.ts` — metodo `cerca()` corretto

```typescript
cerca(): void {
  const id   = this.filtroId.trim().toUpperCase();
  const nome = this.filtroNome.trim().toUpperCase();

  this.loading.set(true);

  // Il server filtra SOLO per AZIENDA (eseguiRicercaPerAzienda).
  // ID e NOME vengono inviati ma ignorati lato server: il filtraggio
  // per questi campi avviene client-side sui risultati restituiti.
  this.api.caricaUtenti({ AZIENDA: this.auth.azienda() })
    .subscribe({
      next: (rows) => {
        let result = rows ?? [];
        if (id)   result = result.filter(r => r.ID.toUpperCase().includes(id));
        if (nome) result = result.filter(r => r.NOME.toUpperCase().includes(nome));
        this.records.set(result);
        this.searched.set(true);
        this.loading.set(false);
      },
      error: (err) => { /* gestione errore */ }
    });
}
```

### Differenze rispetto all'implementazione originale

| Aspetto | Originale (errato) | Corretto |
|---|---|---|
| `AZIENDA` nel body | Assente | `this.auth.azienda()` (`"LT"` o `"SP2"`) |
| Filtro ID/NOME | Inviato al server (ignorato) | Applicato client-side sui risultati |
| Validazione obbligatoria | ID o NOME richiesto | Filtri opzionali (come nel GWT) |
| Wildcard `%` | Aggiunta ai parametri | Rimossa (non serve al server) |

---

## File modificati

- `C:\dev\capitolati-angular\src\app\core\auth\auth.service.ts`
- `C:\dev\capitolati-angular\src\app\shared\lookup\utente-lookup-dialog.component.ts`

## File server (NON modificati — vietato)

- `C:\dev\LISA2Next\LISA2API\src\main\java\it\limontainformatica\rest\resources\capitolati\CapitolatiResource.java`
