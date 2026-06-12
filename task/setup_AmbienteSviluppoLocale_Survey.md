# Setup Ambiente di Sviluppo Locale — Modulo Survey

> **Categoria:** `DevOps / Docker / Spring Boot / Angular`
> **Data:** `12/06/2026`
> **Difficoltà percepita:** ⭐⭐⭐

---

## Obiettivo

Preparare l'ambiente di sviluppo locale completo per lavorare sul bugfix del progetto **LISA Survey**, composto da due repository distinte (backend Spring Boot + frontend Angular) e un'infrastruttura Docker condivisa.

---

## Architettura del progetto

Il progetto LISA Survey è basato su un'architettura a **microservizi**. Non esiste un singolo server monolitico: ogni componente ha un ruolo preciso e gira in modo indipendente.

```
Browser (localhost:4200)
    │
    ▼
Frontend Angular (ng serve, porta 4200)
    │ chiamate HTTP
    ▼
Backend Survey Spring Boot (porta 9090)
    │
    ├── Auth OAuth2 ──────────► authorization-dev.lisasuite.it (NGP Modulo Base, porta 9000 via nginx)
    ├── Service Discovery ────► NGP discovery service (porta 9201)
    ├── API Gateway ──────────► trinity-apigateway (porta 9102)
    └── Database ─────────────► SQL Server container (porta 1433)
                                  └── database: SURVEY-DEV
```

### Repository coinvolte

| Repo | Percorso | Contenuto |
|---|---|---|
| Backend | `C:\dev\Projects\lisa-moduli-backend\` | Monorepo con tutti i microservizi (survey, core, apigateway, ecc.) |
| Frontend | `C:\dev\Projects\lisa-moduli-angular\` | Monorepo con tutte le app Angular |

### Stack tecnologico

- **Backend**: Spring Boot 3.0, JDK 17, Spring Security OAuth2, Spring Cloud Eureka
- **Frontend**: Angular 15.x, DevExtreme UI, angular-auth-oidc-client, Node.js v20
- **Database**: SQL Server 2019 (container Docker)
- **Infrastruttura**: Docker Compose

---

## Parte 1 — I servizi di infrastruttura (NGP Stack)

### Cos'è il NGP Stack

L'infrastruttura base per lo sviluppo locale è il progetto **NGP**, dove ogni servizio ha il proprio file Docker Compose dedicato. Non esiste più un unico `trinity.yml` — i colleghi hanno migrato tutto qui con immagini aggiornate e allineate. Tutti i file si trovano in:

```
C:\dev\NGP\ngp-stack\
├── discovery-service.yml   → Eureka (porta 9201)
├── api-gateway.yml         → API Gateway (porta 9102)
├── modulobase-backend.yml  → Modulo Base OAuth2 (porta 9000)
├── modulobase-angular.yml  → Frontend Modulo Base (porta 4200) — NON serve per Survey
├── nginx.yml               → Nginx (porta 80/443 — serve authorization-dev.lisasuite.it)
├── postgresql.yml          → PostgreSQL (porta 5432 — DB del Modulo Base)
└── .env                    → Variabili d'ambiente condivise da tutti i file
```

| File | Container | Porta | Ruolo |
|---|---|---|---|
| `discovery-service.yml` | `discoveryservice` | 9201 | Eureka — registro dove i microservizi si iscrivono |
| `api-gateway.yml` | `apigateway` | 9102 | API Gateway — smista le richieste ai microservizi giusti |
| `modulobase-backend.yml` | `lisamodulobase` | 9000 | Modulo Base — server di autenticazione OAuth2 |
| `nginx.yml` | `nginx` | 80/443 | Nginx — proxy che espone `authorization-dev.lisasuite.it` |
| `postgresql.yml` | `postgresql` | 5432 | PostgreSQL — database del Modulo Base |

### Rete Docker condivisa

Tutti i file condividono la stessa rete Docker esterna chiamata `modulobase`. Se la rete non esiste ancora (primo avvio assoluto):

```powershell
docker network create modulobase
```

### Comandi per avviare i servizi

I comandi vanno eseguiti dalla cartella `C:\dev\NGP\ngp-stack\` affinché Docker Compose legga automaticamente il file `.env`:

```powershell
cd C:\dev\NGP\ngp-stack
docker compose -f postgresql.yml up -d
docker compose -f nginx.yml up -d
docker compose -f discovery-service.yml up -d
docker compose -f api-gateway.yml up -d
docker compose -f modulobase-backend.yml up -d
```

> Tutti i container hanno `restart: unless-stopped`: si avviano automaticamente con Docker Desktop. Dopo un riavvio del PC dovrebbero già essere attivi.

### Verifica che i servizi siano attivi

```powershell
docker ps --format "table {{.Names}}`t{{.Status}}`t{{.Ports}}" | Select-String "discoveryservice|apigateway|lisamodulobase|nginx|postgresql"
```

### File .env — variabili condivise

Il file `C:\dev\NGP\ngp-stack\.env` contiene le variabili d'ambiente lette da tutti i compose file:

```env
VOLUMES_ROOT=C:/Dev/NGP/volumes
MODULOBASE_NETWORK=modulobase
POSTGRES_PASSWORD=localpassword
MODULOBASE_ISSUER_URI_HOST=https://authorization-dev.lisasuite.it
```

`MODULOBASE_ISSUER_URI_HOST` è l'URL pubblico del Modulo Base — dice al server OAuth2 quale URL mettere come issuer nei token JWT. Deve corrispondere esattamente all'URL usato da frontend e backend (`https://authorization-dev.lisasuite.it`).

---

## Parte 2 — SQL Server (container separato per il Survey)

### Perché un container separato

Il Survey ha bisogno del proprio database (`SURVEY-DEV`) su SQL Server. Esiste un file Docker Compose dedicato:

```
C:\dev\Projects\lisa-moduli-backend\survey\resources\mssql2019.yml
```

Contenuto del file:

```yaml
version: '3.7'
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "Password1!"
    ports:
      - "1433:1433"
    networks:
      - network-mssql2019
    volumes:
      - ${DOCKER_VOLUMES_ROOT}/mssql2019:/var/opt/mssql/data

networks:
    network-mssql2019:
      driver: bridge
```

### Cosa fa ogni campo

- `ACCEPT_EULA: "Y"` — accetta la licenza Microsoft (obbligatorio per l'immagine)
- `SA_PASSWORD` — password dell'utente amministratore `sa`
- `ports: 1433:1433` — espone SQL Server sulla porta standard del host
- `volumes` — persiste i dati del database su disco (cartella `C:\dev\NGP\volumes\mssql2019`), così sopravvivono al riavvio del container

### Comando per avviare SQL Server

```powershell
$env:DOCKER_VOLUMES_ROOT = "C:\dev\NGP\volumes"
docker compose -f C:\dev\Projects\lisa-moduli-backend\survey\resources\mssql2019.yml up -d
```

La variabile `DOCKER_VOLUMES_ROOT` viene letta dal `mssql2019.yml` per costruire il percorso del volume. La cartella `C:\dev\NGP\volumes` è la directory centralizzata dei volumi Docker del progetto (contiene già `modulobase`, `nginx`, ecc.).

---

## Parte 3 — Inizializzazione del Database (solo prima volta)

Il container SQL Server parte vuoto — contiene solo i database di sistema (`master`, `tempdb`, `model`, `msdb`). Il database `SURVEY-DEV` va creato manualmente e poi inizializzato con gli script presenti in:

```
C:\dev\Projects\lisa-moduli-backend\survey\resources\sql\
├── 00-purge.sql         → script di pulizia (non usato al primo setup)
├── 00-purge-quiz.sql    → script di pulizia quiz (non usato al primo setup)
├── 01-ddl.sql           → crea tutte le tabelle del database
└── 02-dll-views.sql     → crea le viste
```

### Step 1 — Crea il database

```powershell
docker exec resources-sqlserver-1 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "Password1!" -Q "CREATE DATABASE [SURVEY-DEV]"
```

> **Nota:** se il comando non stampa nulla, è andato a buon fine. Il silenzio è il segnale di successo in sqlcmd.

### Step 2 — Inietta lo schema (tabelle)

Poiché PowerShell non supporta la redirezione `<` verso `docker exec`, si usa `docker cp` per copiare il file dentro il container e poi eseguirlo dall'interno:

```powershell
docker cp "C:\dev\Projects\lisa-moduli-backend\survey\resources\sql\01-ddl.sql" resources-sqlserver-1:/tmp/01-ddl.sql
docker exec resources-sqlserver-1 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "Password1!" -d "SURVEY-DEV" -i /tmp/01-ddl.sql
```

### Step 3 — Inietta le viste

```powershell
docker cp "C:\dev\Projects\lisa-moduli-backend\survey\resources\sql\02-dll-views.sql" resources-sqlserver-1:/tmp/02-dll-views.sql
docker exec resources-sqlserver-1 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "Password1!" -d "SURVEY-DEV" -i /tmp/02-dll-views.sql
```

### Tabelle create dagli script

Le tabelle principali del Survey sono:

| Tabella | Contenuto |
|---|---|
| `sr_quiz` | I questionari |
| `sr_phase` | Le fasi di ogni questionario |
| `sr_group` | I gruppi di domande |
| `sr_question` | Le domande |
| `sr_choice` | Le risposte possibili |
| `sr_localization` | Testi tradotti in più lingue (IT, EN, DE, FR, VI) |

---

## Parte 4 — Avvio del Backend Survey

### Requisito: JDK 17

Il progetto richiede **JDK 17**. Sul PC sono installati JDK 8, 17 e 21. Java 21 è quello di default nel PATH, ma causa un errore di compilazione (`NoSuchFieldError: JCImport.qualid`) perché alcune API interne del compilatore sono cambiate in Java 21.

Soluzione: impostare `JAVA_HOME` prima di lanciare Maven, solo per il terminale corrente (non cambia le impostazioni di sistema):

```powershell
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
```

### Profili Spring Boot

Il progetto ha profili diversi per ambienti diversi:

| Profilo | Auth server | Database |
|---|---|---|
| `local` | `https://authorization-dev.lisasuite.it` | `localhost:1433` / `SURVEY-DEV` |
| `develop` | Server remoto Limonta | Server remoto |
| `production` | Server produzione | Server produzione |

### Comando per avviare il backend

```powershell
cd C:\dev\Projects\lisa-moduli-backend\survey
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=local"
```

> **Nota:** in PowerShell il parametro `-D` va messo tra virgolette doppie, altrimenti PowerShell lo interpreta male e lo spezza.

### Primo avvio — inizializzazione dati (lenta)

Al primo avvio, Spring Boot carica tutti i dati iniziali nel database (localizzazioni, quiz, domande, fasi, ecc.) tramite insert automatici. Questo processo può richiedere **10-15 minuti** e la console rimane ferma sulle righe di INSERT senza dare feedback visivo.

Per verificare che il processo sia ancora vivo e non bloccato:

```powershell
# Controlla se il processo Java sta consumando CPU (se sale, sta lavorando)
Get-Process -Id <PID> | Select-Object CPU, WorkingSet

# Controlla quante righe ha già inserito nel DB
docker exec resources-sqlserver-1 /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "Password1!" -d "SURVEY-DEV" -Q "SELECT COUNT(*) FROM sr_localization"

# Verifica se il backend è già partito anche se non lo hai visto nei log
netstat -ano | Select-String ":9090"
```

Quando la porta 9090 è in ascolto, il backend è pronto — anche se la riga `Started LisaSurvey` non è visibile nella console per via dello scroll.

**Swagger UI:** `http://localhost:9090/swagger-ui/index.html`

---

## Parte 5 — Avvio del Frontend Survey

### Requisiti

- Node.js v20 (già installato)
- Angular CLI (installato con `npm install -g @angular/cli`)

### Prerequisito: liberare la porta 4200

La porta 4200 è normalmente occupata dal container `lisamodulobase-angular` (app Angular del progetto NGP). Il frontend Survey **deve** girare sulla porta **4200** perché è l'unica redirect URI registrata per il client OAuth2 `angular-client`. Prima di avviare il frontend, fermare quel container:

```powershell
docker stop lisamodulobase-angular
```

> Una volta terminato il lavoro sul Survey, puoi riavviarlo con `docker start lisamodulobase-angular`.

### Installazione dipendenze (solo prima volta)

```powershell
cd C:\dev\Projects\lisa-moduli-angular\lisa-survey
npm install
```

### Avvio

```powershell
npx ng serve --port 4200
```

> **Nota:** si usa `npx ng` invece di `ng` direttamente perché PowerShell blocca l'esecuzione di script `.ps1` di terze parti per policy di sicurezza (`UnauthorizedAccess`). `npx` aggira questo problema.

### Configurazione auth server

Il file `src\assets\settings.json` contiene la configurazione runtime del frontend. Per sviluppo locale deve essere:

```json
{
    "API_BASE_PATH": "http://localhost:9090",
    "AUTHSERVER": "https://authorization-dev.lisasuite.it",
    "APP_NAME": "MyAxia"
}
```

Il valore di default di `AUTHSERVER` punta a `http://authorization-myaxia.axiahrm.com:9000` (server remoto) — va cambiato a `https://authorization-dev.lisasuite.it`, che è il Modulo Base NGP locale servito tramite nginx. Anche il backend (`application-local.yaml`) deve puntare allo stesso URL.

### Cosa succede quando premi "Login" — il flusso callback

Angular non gestisce il login da solo: lo delega completamente al Modulo Base tramite il protocollo OAuth2. Quando apri `http://localhost:4200` e non sei autenticato, parte questa sequenza:

**Step 1 — Angular ti manda al Modulo Base**

Il browser viene reindirizzato a:
```
https://authorization-dev.lisasuite.it/oauth2/authorize
  ?client_id=angular-client
  &redirect_uri=http://localhost:4200/callback
  &response_type=code
  &scope=openid
  &code_challenge=<hash>
  &code_challenge_method=S256
```

**Step 2 — Inserisci le credenziali sul Modulo Base**

La pagina di login è servita da `authorization-dev.lisasuite.it`. Inserisci username e password. Il Modulo Base le verifica nel database PostgreSQL.

**Step 3 — Il Modulo Base ti rimanda su `/callback` con un codice**

Se le credenziali sono corrette, il Modulo Base reindirizza il browser a:
```
http://localhost:4200/callback?code=XXXXX&state=YYYYY
```
`code` è un codice temporaneo monouso valido pochi secondi.

**Step 4 — Angular scambia il codice per il token JWT**

Angular intercetta la rotta `/callback`, estrae il `code` e fa una chiamata POST al Modulo Base per ricevere il token JWT reale. Salva il token e ti porta automaticamente su `/home`.

Da quel momento ogni chiamata al backend Survey include nell'header:
```
Authorization: Bearer <token-jwt>
```

> Per un'analisi dettagliata del protocollo (PKCE, state, code_verifier, ecc.) vedi il file dei concetti.

**URL frontend:** `http://localhost:4200`

---

## Errori riscontrati e soluzioni

### Errore 1 — Porta 9102 già allocata

```
Bind for 0.0.0.0:9102 failed: port is already allocated
```

**Causa:** Container di un altro progetto (`apigateway`, `lisamodulobase`, `discoveryservice`) già in esecuzione sulle stesse porte della trinità.

**Soluzione:** Fermare i container in conflitto prima di avviare la trinità.
```powershell
docker stop lisamodulobase discoveryservice apigateway
```

---

### Errore 2 — Maven: lifecycle phase non riconosciuta

```
Unknown lifecycle phase ".run.profiles=local"
```

**Causa:** In PowerShell, `-Dspring-boot.run.profiles=local` viene spezzato — `-D` viene interpretato come un flag PowerShell e il resto come argomento separato.

**Soluzione:** Mettere l'argomento tra virgolette doppie:
```powershell
./mvnw spring-boot:run "-Dspring-boot.run.profiles=local"
```

---

### Errore 3 — Compilazione fallita con JDK 21

```
java.lang.NoSuchFieldError: Class com.sun.tools.javac.tree.JCTree$JCImport does not have member field 'qualid'
```

**Causa:** Il progetto è scritto per JDK 17. In JDK 21 alcune API interne del compilatore sono state modificate e il campo `qualid` nella classe `JCImport` non esiste più.

**Soluzione:** Impostare `JAVA_HOME` a JDK 17 nel terminale corrente:
```powershell
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
```

---

### Errore 4 — SQL Server non risponde (0 bytes)

```
Prelogin error: host localhost port 1433 Unexpected end of prelogin response after 0 bytes read
```

**Causa:** La porta 1433 era mappata dal container `trinity-modulobase` (che però non ha SQL Server al suo interno). Il socket Docker accettava la connessione TCP ma la chiudeva subito perché nulla era in ascolto sulla 1433 dentro il container.

**Soluzione:**
1. Rimosso il mapping `1433:1433` dal `trinity.yml` (era inutile)
2. Avviato il container SQL Server dedicato con `mssql2019.yml`

---

### Errore 5 — `ng` non riconosciuto in PowerShell

```
Impossibile caricare il file C:\nvm4w\nodejs\ng.ps1. L'esecuzione di script è disabilitata nel sistema in uso.
```

**Causa:** PowerShell ha una policy di sicurezza che blocca l'esecuzione di script `.ps1` non firmati, incluso quello di Angular CLI.

**Soluzione:** Usare `npx ng` invece di `ng` direttamente:
```powershell
npx ng serve --port 4200
```

---

### Errore 6 — `ERR_CONNECTION_TIMED_OUT` sull'auth server

Nel browser appariva:
```
authorization-myaxia.axiahrm.com:9000/.well-known/openid-configuration: Failed to load resource: ERR_CONNECTION_TIMED_OUT
```

**Causa:** Il file `settings.json` del frontend puntava all'auth server remoto di un altro prodotto invece di quello corretto.

**Soluzione:** Modificato `src\assets\settings.json`:
```json
"AUTHSERVER": "https://authorization-dev.lisasuite.it"
```

---

### Errore 7 — Login fallisce con status 400 / errore "Whitelabel Error Page"

Dopo il tentativo di login il browser finisce su:
```
https://authorization-dev.lisasuite.it/error?continue
There was an unexpected error (type=Bad Request, status=400).
```

Nei log del modulobase appare:
```
OAuth2AuthorizationCodeRequestAuthenticationException
```

**Causa:** Il client OAuth2 `angular-client` non aveva `http://localhost:4200/callback` tra i redirect URI registrati nel database. Spring Authorization Server rifiuta qualsiasi richiesta con un redirect URI non esattamente corrispondente a quelli registrati.

**Come si è individuata la causa:** guardando i log del modulobase si vede la richiesta arrivare a `/oauth2/authorize` con `redirect_uri=http://localhost:4200/callback`, seguita immediatamente dall'eccezione e dal redirect a `/error`.

**Soluzione:** Aggiunto il redirect URI mancante direttamente nel database PostgreSQL del Modulo Base NGP:

```powershell
# Verificare i redirect URI attuali per angular-client
docker exec postgresql psql -U admin -d modulobase -c "SELECT a.clientid, r.redirect_uri_set FROM mb_registered_application a JOIN mb_registered_application_redirect_uri_set r ON a.uniqueuuid = r.registered_application_uniqueuuid WHERE a.clientid = 'angular-client' AND r.redirect_uri_set LIKE '%localhost:4200%';"

# Aggiungere il redirect URI mancante
docker exec postgresql psql -U admin -d modulobase -c "INSERT INTO mb_registered_application_redirect_uri_set (registered_application_uniqueuuid, redirect_uri_set) SELECT uniqueuuid, 'http://localhost:4200/callback' FROM mb_registered_application WHERE clientid = 'angular-client';"
```

> **Nota:** il database PostgreSQL del Modulo Base NGP usa utente `admin`, database `modulobase`, container `postgresql`. I redirect URI sono nella tabella `mb_registered_application_redirect_uri_set`, non nella tabella standard Spring `oauth2_registered_client` — il progetto usa una struttura dati personalizzata.

---

## Flusso completo per i prossimi avvii

Dalla seconda volta in poi, il database esiste già e i dati sono caricati. Il flusso è molto più veloce:

```powershell
# 0. Libera la porta 4200
docker stop lisamodulobase-angular

# 1. Verifica che i servizi NGP siano attivi (di solito lo sono già)
docker ps --format "table {{.Names}}`t{{.Status}}" | Select-String "discoveryservice|apigateway|lisamodulobase|nginx|postgresql"
# Se manca qualcuno, avviarlo dalla cartella C:\dev\NGP\ngp-stack\:
# docker compose -f <nome-file>.yml up -d

# 2. Avvia SQL Server
$env:DOCKER_VOLUMES_ROOT = "C:\dev\NGP\volumes"
docker compose -f C:\dev\Projects\lisa-moduli-backend\survey\resources\mssql2019.yml up -d

# 3. Avvia backend (in C:\dev\Projects\lisa-moduli-backend\survey\)
$env:JAVA_HOME = "C:\Program Files\Java\jdk-17"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=local"

# 4. Avvia frontend (in C:\dev\Projects\lisa-moduli-angular\lisa-survey\)
npx ng serve --port 4200
```

---

## Concetti appresi

- **Docker Compose**: file YAML che descrive e avvia più container insieme come se fossero un'unica applicazione
- **Profili Spring Boot**: configurazioni diverse per lo stesso progetto (locale, sviluppo, produzione) — si attivano con `-Dspring-boot.run.profiles=<nome>`
- **OAuth2 / Auth server**: il login non è gestito dall'app Survey ma delegato a un server centrale (Modulo Base) — il frontend viene reindirizzato lì per il login e poi torna con un token JWT
- **redirect_uri OAuth2**: per sicurezza l'auth server accetta solo URI di ritorno esplicitamente registrati nel proprio database — se manca anche solo `localhost` vs `127.0.0.1` o `/callback` nel path, la richiesta viene rifiutata con errore 400
- **Eureka**: registro dei microservizi — ogni servizio si "iscrive" all'avvio così gli altri lo trovano per nome invece che per IP fisso
- **JAVA_HOME**: variabile d'ambiente che dice al sistema quale installazione Java usare — impostarla nel terminale sovrascrive quella di sistema solo per quella sessione
- **docker exec**: esegue un comando dentro un container già in esecuzione
- **docker cp**: copia file tra il filesystem del host e quello del container

---

## Note personali

- La trinità è la base di tutto: senza di essa nessun modulo LISA funziona
- Il primo avvio del backend è molto lento a causa del caricamento iniziale dei dati — non allarmarsi se sembra bloccato
- Per verificare se un'app Spring Boot è partita senza guardare i log: `netstat -ano | Select-String ":<porta>"`
- Il file `settings.json` del frontend va ricordato: ogni volta che si clona il progetto andrà modificato per puntare ad `https://authorization-dev.lisasuite.it`
- Il frontend Survey deve girare sulla porta **4200**, non 4201 — il redirect URI registrato nel DB è esattamente `http://localhost:4200/callback` e Spring Security fa un confronto esatto
- Prima di avviare il frontend, fermare sempre `lisamodulobase-angular` che occupa la porta 4200
