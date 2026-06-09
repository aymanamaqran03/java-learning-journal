# 05/06/2026 - filtro per data documento

- **Errore:** Non ho effettuato l'associazione dei campi attraverso `setAssociatedBeanField` dunque il server non riceveva i valori di fromPRDDOC e toPRDDOC.
  **Soluzione:** Ho effettuato il collegamento all'interno delle `Properties` del `DateInputBox` nel campo `associatedBeanField`.

- **Errore:** Lato server Avevo impostato erroneamente i campi fromPRDDOC e toPRDDOC con l'argomento `persistent=true`.
  **Soluzione:** Ho cambiato il parametro in `persistent=false` in quanto si tratta di campi dedicati al filtraggio per cui non è necessario effettuare INSERT/UPDATE e dunque vengono salvati solo in memeoria.

  # 08/06/2026 - refactor query elenco provvigioni

- **Errore:** Nel `tryGetString` avevo inserito il nome della colonna sbagliato.
  **Soluzione:** Ho sostituito il nome della colonna passata nell'argomento del metodo. In caso di imputazione di Alias alla colonna, è possibile utilizzare anche quest'ultimo.
