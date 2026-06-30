# Concetti — Docker

> **Categoria:** `Docker / DevOps / Container / CI-CD`
> **Aggiornato:** `30/06/2026`

---

## Indice

**Fondamentali**

1. [Il problema che Docker risolve](#1-il-problema-che-docker-risolve)
2. [Container](#2-container)
3. [Image](#3-image)
4. [Volumi](#4-volumi)
5. [Docker Compose](#5-docker-compose)
6. [File YAML](#6-file-yaml)

**Build e Deploy**

7. [Panoramica dell'architettura build/deploy](#7-panoramica-dellarchitettura-builddeploy)
8. [Le due macchine coinvolte](#8-le-due-macchine-coinvolte)
9. [Il Dockerfile — multi-stage build](#9-il-dockerfile--multi-stage-build)
10. [Il file .env](#10-il-file-env)
11. [build.sh — build dell'immagine](#11-buildsh--build-dellimmagine)
12. [publish.sh — push su Docker Hub](#12-publishsh--push-su-docker-hub)
13. [compose.yaml — avvio del container](#13-composeyaml--avvio-del-container)
14. [Flusso completo step-by-step](#14-flusso-completo-step-by-step)
15. [Glossario concetti chiave](#15-glossario-concetti-chiave)

---

## 1. Il problema che Docker risolve

Docker è uno strumento che permette di eseguire applicazioni in ambienti isolati e portabili chiamati **container**.

Immagina di dover eseguire un'applicazione Java che richiede esattamente JDK 17, una specifica versione di SQL Server, e alcune variabili d'ambiente configurate in un certo modo. Sul tuo PC potresti avere JDK 21 installato di default, una versione diversa di SQL Server, e configurazioni diverse. Il risultato classico è: "funziona sulla mia macchina ma non sulla tua". Docker elimina questo problema mettendo l'applicazione e tutto il suo ambiente in una scatola autonoma, che funzionerà in modo identico ovunque Docker sia installato.

---

## 2. Container

Un **container** è un processo in esecuzione isolato dal resto del sistema operativo. Non è una macchina virtuale — non simula un hardware diverso, non ha un kernel separato. Usa il kernel del sistema operativo host ma è separato in termini di filesystem, rete e processi. Questo lo rende molto più leggero di una VM: si avvia in secondi, consuma poca memoria, e puoi averne decine in esecuzione contemporaneamente.

Pensa a un container come a un appartamento in un condominio: ogni appartamento ha il suo interno, i suoi oggetti, la sua porta chiusa. Gli inquilini non vedono cosa succede negli altri appartamenti. Ma tutti condividono le fondamenta, il tetto e l'impianto idraulico (il kernel del sistema operativo host). È molto diverso dall'avere case separate su terreni separati (le macchine virtuali), che richiedono ognuna la propria fondamenta e il proprio tetto.

Ogni container ha:
- Il suo **filesystem** isolato (non vede i file dell'host né degli altri container, a meno che non si configurino esplicitamente dei "ponti")
- La sua **rete** interna (ha il suo indirizzo IP, le sue porte)
- I suoi **processi** (il sistema operativo host non vede direttamente cosa gira dentro)
- Le sue **variabili d'ambiente**

Quando un container viene fermato (`docker stop`), i processi al suo interno si fermano, ma il container rimane disponibile. Quando viene rimosso (`docker rm`), scompare. Quando viene avviato di nuovo (`docker start`), riparte dallo stesso stato in cui era stato creato.

---

## 3. Image

Un'**image** (immagine Docker) è il "progetto" da cui nasce un container. Se il container è una casa in esecuzione con persone che ci vivono dentro, l'image è il progetto architettonico: descrive come deve essere costruita la casa, cosa deve contenere, come è configurata. Da un unico progetto puoi costruire quante case identiche vuoi.

Un'image è composta da **layer** (strati) impilati uno sopra l'altro. Per esempio, l'image di SQL Server usata nel progetto (`mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04`) è costruita così:

1. **Layer base**: Ubuntu 16.04 (il sistema operativo)
2. **Layer SQL Server**: i binari di SQL Server 2019 installati sopra Ubuntu
3. **Layer configurazione**: script di avvio, variabili di default, ecc.

Questo sistema a layer ha un vantaggio enorme: se hai due immagini che partono dallo stesso Ubuntu, Docker scarica quel layer una volta sola e lo condivide. Risparmia spazio su disco e tempo di download.

Le immagini vengono scaricate da un **registry** — un archivio pubblico o privato di immagini. Il registry pubblico più famoso è **Docker Hub** (`hub.docker.com`). Le immagini del progetto LISA (`javadevliminf/lisamodulobase:dev`, ecc.) vengono scaricate da un registry privato di Limonta Informatica.

Il tag dopo i due punti (`:dev`, `:2019-GA-ubuntu-16.04`) identifica la **versione** dell'immagine. `dev` significa che è la versione di sviluppo, aggiornata regolarmente dal team.

---

## 4. Volumi

I container sono **efimeri per natura**: quando un container viene rimosso, tutto quello che era scritto nel suo filesystem sparisce. Per i database questo è un problema enorme — se fermi e rimuovi il container SQL Server, perdi tutti i dati.

I **volumi** sono il meccanismo di Docker per rendere i dati persistenti. Un volume è essenzialmente un collegamento tra una cartella nel filesystem del **container** e una cartella nel filesystem dell'**host** (il tuo PC). Ogni scrittura che il container fa in quella cartella finisce realmente sul tuo PC, e sopravvive alla rimozione del container.

Nel `mssql2019.yml` del Survey:

```yaml
volumes:
  - ${DOCKER_VOLUMES_ROOT}/mssql2019:/var/opt/mssql/data
```

Questa riga dice: "la cartella `/var/opt/mssql/data` dentro il container (dove SQL Server scrive i suoi file di database) deve essere mappata sulla cartella `C:\dev\NGP\volumes\mssql2019` sul tuo PC". Quando SQL Server crea il file del database `SURVEY-DEV`, lo scrive in `/var/opt/mssql/data`, che in realtà è `C:\dev\NGP\volumes\mssql2019` sul tuo disco. Se domani fermi e rimuovi il container e lo riavvii, SQL Server trova i file già lì e riparte con tutti i dati intatti.

---

## 5. Docker Compose

Gestire un container alla volta con i comandi `docker run` diventa complicato quando hai più servizi che devono partire insieme, comunicare tra loro e condividere configurazioni. **Docker Compose** nasce per questo: ti permette di descrivere una **collezione di container** in un unico file di configurazione e gestirli tutti insieme con un solo comando.

Con Docker Compose puoi:
- Definire tutti i servizi che compongono la tua applicazione
- Specificare come si parlano tra loro (reti interne)
- Configurare variabili d'ambiente, porte, volumi per ciascuno
- Avviarli tutti con `docker compose up -d` e fermarli tutti con `docker compose down`

Il flag `-d` (detached) significa "avvia i container in background e restituisci il controllo del terminale" — senza di esso, i log di tutti i container scorrerebbero nel terminale finché non premi Ctrl+C.

---

## 6. File YAML

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

## 7. Panoramica dell'architettura build/deploy

Il rilascio dell'applicazione Angular segue un flusso in **tre fasi**:

```
[Sviluppatore]         [DOCKER-BUILD]             [ELM-DOCKER03]
     │                      │                           │
     │  git push (main)      │                           │
     │─────────────────────>│                           │
     │                      │  docker build             │
     │                      │  (clona da GitHub)        │
     │                      │  docker push → Docker Hub │
     │                      │─────────────────────────>│
     │                      │                           │  docker compose pull
     │                      │                           │  docker compose up -d
     │                      │                           │  [app aggiornata]
```

Il codice sorgente non viene mai copiato manualmente sul server: è **GitHub** il punto di verità, e il Dockerfile si occupa di clonare l'ultima versione in fase di build.

---

## 8. Le due macchine coinvolte

| Macchina | Ruolo | Cosa contiene |
|---|---|---|
| **DOCKER-BUILD** | Build server | Dockerfile, build.sh, publish.sh, .env |
| **ELM-DOCKER03** | Production server | compose.yaml, container in esecuzione |

Le due macchine sono separate per motivi di sicurezza e organizzazione: la macchina di build ha accesso a Docker Hub (credenziali push), quella di produzione ha solo accesso in pull.

---

## 9. Il Dockerfile — multi-stage build

Il Dockerfile usa un pattern chiamato **multi-stage build**: divide il processo in due fasi distinte per ottenere un'immagine finale piccola e pulita.

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────────
FROM node:18.20.8-alpine AS build

ARG GIT_REPO_URL        # URL del repository GitHub
ARG GIT_BRANCH=main     # Branch da clonare (default: main)

RUN git clone --depth 1 --branch "${GIT_BRANCH}" "${GIT_REPO_URL}" .
RUN npm install --legacy-peer-deps
RUN npx ng build

# ── Stage 2: Serve ──────────────────────────────────────────
FROM nginx:1.30-alpine

COPY --from=build /src/dist/capitolati-angular/browser /usr/share/nginx/html
EXPOSE 80
```

### Perché multi-stage?

- Lo **Stage 1** usa Node.js (immagine pesante ~300MB) per compilare il codice Angular
- Lo **Stage 2** usa nginx (immagine leggera ~40MB) per servire solo i file statici compilati
- Il risultato finale non contiene Node.js, npm, il codice sorgente, né le `node_modules` → immagine finale ~65MB invece di ~700MB

### `--depth 1`

Il flag `--depth 1` clona solo l'ultimo commit del branch, senza tutta la history di git. Questo rende il clone molto più veloce.

---

## 10. Il file .env

Il file `.env` si trova in `/opt/docker-liminf/build/capitolati-web/` ed è caricato da `build.sh`. Contiene le variabili di configurazione del build:

```bash
GIT_REPO_URL=https://github.com/limontainformatica/capitolati-angular.git
GIT_BRANCH=main
BUILD_CONFIGURATION=production
ANGULAR_PROJECT_NAME=
IMAGE_NAME=capitolati-web:latest
```

Il file `.env` non è versionato su git (è nell'infrastruttura del server) perché potrebbe contenere credenziali o configurazioni specifiche dell'ambiente.

---

## 11. build.sh — build dell'immagine

```bash
set -a
source .env     # carica le variabili dal file .env
set +a

docker build --no-cache \
  --build-arg GIT_REPO_URL="$GIT_REPO_URL" \
  --build-arg GIT_BRANCH="$GIT_BRANCH" \
  --build-arg BUILD_CONFIGURATION="$BUILD_CONFIGURATION" \
  --build-arg ANGULAR_PROJECT_NAME="$ANGULAR_PROJECT_NAME" \
  --progress=plain \
  -t $IMAGE_NAME .
```

- `--no-cache`: forza il rebuild completo senza usare layer cachati. Importante perché altrimenti Docker potrebbe riutilizzare il layer del `git clone` e non scaricare le ultime modifiche
- `--build-arg`: passa variabili al Dockerfile (accessibili come `ARG` dentro il Dockerfile)
- `-t $IMAGE_NAME`: assegna un nome e tag all'immagine (es. `capitolati-web:latest`)
- `--progress=plain`: mostra l'output del build in modo esteso (utile per debug)

---

## 12. publish.sh — push su Docker Hub

```bash
docker tag capitolati-web:latest javadevliminf/capitolati-web:latest
docker push javadevliminf/capitolati-web:latest
```

- `docker tag`: crea un alias dell'immagine locale con il nome completo per Docker Hub (`account/repository:tag`)
- `docker push`: carica l'immagine su Docker Hub, rendendola disponibile a qualsiasi macchina che la voglia scaricare

Docker Hub funziona come un "GitHub per immagini Docker": è un registry pubblico (o privato) dove le immagini vengono archiviate e distribuite.

---

## 13. compose.yaml — avvio del container

Il file si trova su **ELM-DOCKER03** e descrive come avviare il container in produzione:

```yaml
services:
  frontend:
    image: javadevliminf/capitolati-web:latest   # scarica da Docker Hub
    ports:
      - "8080:80"    # porta 8080 del server → porta 80 del container (nginx)
    restart: unless-stopped
```

- `image`: Docker Compose scarica automaticamente l'immagine da Docker Hub se non è presente in locale, oppure usa quella locale se già presente
- `ports: "8080:80"`: mappa la porta 80 di nginx (dentro il container) sulla porta 8080 del server host
- `restart: unless-stopped`: il container si riavvia automaticamente se crasha o se il server viene riavviato, tranne se fermato manualmente

---

## 14. Flusso completo step-by-step

### Prerequisiti
- La modifica al codice è già stata committata e pushata su GitHub (`main`)

### Step 1 — Build (su DOCKER-BUILD via PuTTY)

```bash
cd /opt/docker-liminf/build/capitolati-web

./build.sh      # clona da GitHub e builda l'immagine → capitolati-web:latest
./publish.sh    # tagga e pusha su Docker Hub → javadevliminf/capitolati-web:latest
```

### Step 2 — Deploy (su ELM-DOCKER03 via PuTTY)

```bash
cd <cartella-con-compose.yaml>

docker compose pull     # scarica la nuova immagine da Docker Hub
docker compose up -d    # riavvia il container con la nuova immagine
```

Il flag `-d` (detached) avvia il container in background, senza bloccare il terminale.

### Verifica

```bash
docker ps                        # controlla che il container sia in esecuzione
docker logs <container-id>       # leggi i log del container se qualcosa non va
```

---

## 15. Glossario concetti chiave

| Termine | Significato |
|---|---|
| **Image** | "Foto" immutabile dell'applicazione con tutto il necessario per eseguirla |
| **Container** | Istanza in esecuzione di un'immagine |
| **Layer** | Ogni istruzione del Dockerfile crea un layer. Docker li cacha per velocizzare i rebuild |
| **Registry** | Archivio di immagini (Docker Hub è quello pubblico, ne esistono di privati) |
| **Tag** | Versione di un'immagine (es. `latest`, `1.0.0`, `stable`) |
| **Multi-stage build** | Tecnica per separare il build environment dal runtime environment, ottenendo immagini più piccole |
| **Build arg** | Variabile passata al Dockerfile durante il build (non disponibile a runtime) |
| **Port mapping** | Collega una porta del container a una porta dell'host (`host:container`) |
| **Volume** | Collegamento tra cartella del container e cartella dell'host per rendere i dati persistenti |
| **Docker Compose** | Strumento per definire e gestire più container come un'unica applicazione |

---

## Note personali

- **La rete interna Docker è trasparente per i container, non per l'host** — I container sulla stessa rete si vedono usando il `container_name` come hostname. Ma dall'host (il tuo PC) devi usare `localhost:<porta mappata>`. Quando una variabile d'ambiente dentro un container dice `http://discoveryservice:9201`, funziona perché entrambi i container sono sulla stessa rete Docker. Se metti quell'URL nel browser, non funzionerà.

- **`--no-cache` è obbligatorio per il `git clone`** — Senza questo flag, Docker riutilizza il layer del `git clone` dalla cache precedente anche se il codice su GitHub è cambiato. Con `--no-cache` ogni build parte da zero e clona l'ultima versione del branch.

- **Il tag `:latest` non è automaticamente "la più recente"** — È solo un tag come un altro. Se fai `docker push` di una nuova versione senza aggiornare il tag `latest`, le macchine che fanno `docker pull` continueranno a scaricare la versione vecchia. `docker compose pull` aggiorna l'immagine locale solo se l'immagine con quel tag sul registry è cambiata rispetto a quella locale.
