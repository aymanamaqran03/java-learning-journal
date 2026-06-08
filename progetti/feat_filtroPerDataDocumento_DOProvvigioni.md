# filtro per data documento

> **Categoria:** `OOP`  
> **Data:** `08/06/2026`  
> **Difficoltà percepita:** ⭐

---

### Obiettivo

Aggiungere al pannello filtro provvigioni la possibilità di filtrare per un range di date del documento (`PRDDOC`).

### Modifiche server-side

**`DOProvvigioni.java` (JavaCore)**

Aggiunta di due campi `persistent = false` senza `tables`, usati solo come contenitori del filtro:

```java
@DOMetaData(description = "Da data documento", size = 8, separator = true, persistent = false)
public Data fromPRDDOC;

@DOMetaData(description = "A data documento", size = 8, separator = true, persistent = false)
public Data toPRDDOC;
```

**`GestioneContabilitaBean.java` (POJOEJBServices)**

Aggiunta dei vincoli alla query in `elencoProvvigioni`:

```java
query.addConstraint("PRDDOC", ">=", filtro.fromPRDDOC, Data.getNullDate());
query.addConstraint("PRDDOC", "<=", filtro.toPRDDOC,   Data.getNullDate());
```

La firma a 4 argomenti garantisce che il vincolo venga aggiunto **solo se l'utente ha inserito una data**. Se il campo rimane al valore di default (`Data.getNullDate()`), il filtro viene ignorato e la query estrae tutti i record senza limitazioni di data.

### Modifiche client-side

**`FiltroPanel.java` (LISAClient)**

Nel form NetBeans sono stati aggiunti due componenti `DateInputBox` con variable name `fromPRDDOC` e `toPRDDOC`. Nel costruttore è stato aggiunto il collegamento con i campi del DO:

```java
fromPRDDOC.setAssociatedBeanField("fromPRDDOC");
toPRDDOC.setAssociatedBeanField("toPRDDOC");
```

### Come funziona `setAssociatedBeanField`

Il `filtroLisaPanel` (di tipo `LISAPanel`) gestisce automaticamente il trasferimento UI → DO tramite `salvaValori(doFiltro)`. Per ogni componente figlio, legge il valore di `getAssociatedBeanField()` per sapere in quale campo del DO scrivere il valore letto dall'UI.

Senza `setAssociatedBeanField`, il componente viene visualizzato correttamente ma il suo valore non viene mai trasferito nel DO e quindi non arriva al server.

### Flusso completo end-to-end

```
Utente inserisce date in fromPRDDOC e toPRDDOC (UI)
        ↓
"Cerca" → FiltroPanel.ricerca()
        ↓
filtroLisaPanel.salvaValori(doFiltro)
  → legge fromPRDDOC via getAssociatedBeanField()
  → scrive in doFiltro.fromPRDDOC
  → stessa cosa per toPRDDOC
        ↓
handler.elencoProvvigioni(doFiltro, page)
  → serializza doFiltro e lo manda al server HTTP
        ↓
GestioneContabilitaBean.elencoProvvigioni()
  → legge filtro.fromPRDDOC e filtro.toPRDDOC
  → addConstraint("PRDDOC", ">=", filtro.fromPRDDOC, getNullDate())
  → addConstraint("PRDDOC", "<=", filtro.toPRDDOC,   getNullDate())
  → esegue SELECT su PROVV00F con il range di date
        ↓
Risultati ritornano serializzati nell'ApplicationContext
        ↓
FiltroPanel popola xElenco con la lista filtrata
```
---

## Errori che ho fatto


- **Errore:** Non ho effettuato l'associazione attraverso `setAssociatedBeanField` dunque il server non riceveva i valori di fromPRDDOC e toPRDDOC.
  **Soluzione:** Ho effettuato il collegamento all'interno delle `Properties` del `DateInputBox` nel campo `associatedBeanField`.

- **Errore:** Lato server Avevo impostato erroneamente i campi fromPRDDOC e toPRDDOC con l'argomento `persistent=true`.
  **Soluzione:** Ho cambiato il parametro in `persistent=false` in quanto si tratta di campi dedicati al filtraggio per cui non è necessario effettuare INSERT/UPDATE e dunque vengono salvati solo in memeoria.

---

## Concetti collegati

- Collegato a: `..\concetti\LISA2.md`

---

## Note personali

Per poter visualizzare il risultato di una query in fase di debug è possibile utilizzare metodo `getFullQuery()` in questa maniera:

```java
System.out.println(query.getFullQuery());
```

## Risorse utili

