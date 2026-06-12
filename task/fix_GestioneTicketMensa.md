# Gestione Ticket Mensa

> **Categoria:** `OOP`  
> **Data:** `10/06/2026`  
> **Difficoltà percepita:** ⭐⭐

## PARTE 1 - OTTIMIZZAZIONE INTERFACCIA ESITO SCAN QR CODE

### Obiettivo
L'obiettivo di questa implementazione è stato quello di ottimizzare l'interfaccia che viene mostrata dopo aver scannerizzato il QR code del ticket della mensa. Nel caso di ticket inesistente o già vidimato viene mostrata la schermata di errore. Nel caso di ticket disponibile, invece, viene mostrato un modale che chiede la conferma di attivazione.

### Step 1 — Aggiunta del metodo verificaStatoTicket() di sola lettura in GestioneTicketMensa.java

Ho aggiunto un metodo chiamato `verificaStatoTicket(String codiceRisorsa, String progressivoTicket)` che non attiva il ticket ma ne legge lo stato (disponibile / già usato / non trovato). Questo serve per separare il "controlla" dal "attiva".

La query cerca nella tabella `MENRI00F` il record con `RMCDIP = codiceRisorsa` e `RMNRPR = progressivoTicket` e restituire il valore di `RMDTLE`, ovvero la data di lettura del ticket. Se la data di lettura è pari a zero significa che il ticket non è ancora stato attivato.

### Step 2 — Modifica mensa.jsp con un flusso in due fasi

La JSP distinguere due fasi tramite il parametro aggiuntivo `conferma`:

1. Fase 1 - PRIMA CHIAMATA HTTP (la prima scansione del QR code. In questa fase il parametro è pari a `null`):
- Chiama `verificaStatoTicket()`
- Se ticket già usato → mostra pagina rossa con messaggio "Ticket già vidimato o non corrispondente alla risorsa."
- Se ticket disponibile → mostra il modale che chiede la conferma di attivazione.

2. Fase 2 - SECONDA CHIAMATA HTTP (utente clicca conferma, parametro `conferma=true` presente):
- Chiama `vidimaTicketMensa()` che scrive sul db la data di attivazione nel campo `RMDTLE`.
- Mostra pagina con "Ticket abilitato con successo. <br> Buon pranzo "+risorsa+"!""

Il bottone di conferma è un form HTML che riposta i parametri operatore e numeroticket aggiungendo conferma=true.


## CONCETTI DA RICORDARE

1. il metodo `addKey()` aggiunge una condizione alla clausola WHERE della query SQL che verrà eseguita da SQLWriter. Internamente fa questo:

```java
keyList.add(new SQLParameter(fieldName, operator, value));
```

Quindi accumula le condizioni in una lista, che poi vengono combinate in `WHERE campo1 = val1 AND campo2 = val2 AND ...` al momento dell'esecuzione.

Viene utilizzato in `vidimaTicketMensa` come segue:

```java
updater.addKey("RMCDIP", "=", codiceRisorsa);
updater.addKey("RMNRPR", "=", prgTicket);
updater.addKey("RMDTLE", "=", 0);
updater.writeData("RMDTLE", Data.getData());
```

Tradotto in SQL diventa:

UPDATE LIMENSA.MENRI00F
SET RMDTLE = <data_odierna>
WHERE RMCDIP = 'XX'
  AND RMNRPR = 123
  AND RMDTLE = 0


2. Una volta scritta la logica di lancio della query, bisogna eseguire il metodo `.prepare()` per lanciarla. In seguito è necessario inizializzare un cursore e leggere l'intero `ResultSet` per poter navigare all'interno del risultato ottenuto.

## REMINDER

Il bottone `Annulla` non funziona su browser che non permettono `window.close()` su pagine aperte direttamente (es. da scansione QR). Da gestire con una soluzione alternativa (reindirizzamento, messaggio, ecc.).

## PARTE 2 - MODIFICA LOGICA GENERAZIONE TICKET SU LISA

### Obiettivo

L'obiettivo di questa implementazione è stato quello di modificare la logica di generazione dei ticket su LISA. Attualmente venivano generati i ticket al netto di quelli già disponibili e presenti sul DB. Questa modifica è volta ha cambiare questa logica in modo tale che venga creato l'esatto numero di ticket desiderato dall'operatore, a prescindere dai ticket ancora disponibili.


### Modifica funzione creaTicket() in GestioneTicketMensa()

È stato rimosso dalla funzione il frammento di codice che faceva la `COUNT` dei ticket già disponibili e presenti a sistema. Inoltre è stato variato il campo che cila nel `for`, impostando la variabile passata dal cliente.