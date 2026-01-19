# DevOps Practice – Jenkins CI/CD with Docker Compose

Questo repository è un **progetto di studio/pratica DevOps** che mostra come progettare e implementare una pipeline di **CI/CD completa con Jenkins**, partendo da un **mono-repo** che contiene:

- un **frontend Angular**
- un **backend Spring Boot**
- un’infrastruttura locale basata su **Docker Compose**

L’obiettivo è simulare un setup realistico, vicino a contesti enterprise, mantenendo però l’ambiente **riproducibile in locale**.


## Obiettivi del progetto

- Applicare **best practice DevOps** su un progetto full-stack
- Utilizzare **Jenkins Pipeline as Code (Jenkinsfile)**
- Costruire e testare FE e BE in **container dedicati**
- Creare immagini Docker versionate
- Effettuare **deploy automatico su branch `master`**
- Ricreare **solo i servizi che cambiano** (deploy mirato)
- Gestire correttamente PR, branch e change detection


##  Struttura del repository

```bash
├── front-end/
│   └── webapp/              # Applicazione Angular
│       ├── Dockerfile       # Build FE (Node → Nginx)
│       └── …
│
├── back-end/
│   └── api/                 # Applicazione Spring Boot
│       ├── Dockerfile       # Runtime Java 17
│       └── …
│
├── infra/
│   └── docker-compose.yml   # Orchestrazione FE + BE in locale
│
├── jenkins-local/
│   ├── Dockerfile           # Jenkins con Docker CLI
│   └── docker-compose.yml   # Avvio Jenkins in locale
│
└── Jenkinsfile              # Pipeline CI/CD (root)
```


## Workflow CI/CD

### Trigger
- **Push su `master`**
  - build selettiva (FE / BE)
  - deploy automatico locale
- **Pull Request**
  - build e test
  - ❌ nessun deploy

### Pipeline (alto livello)
1. Checkout del repository
2. Rilevamento file modificati (`git diff`)
3. Build FE (container Node)
4. Build BE (container Maven)
5. Build immagini Docker
6. Deploy locale con Docker Compose

### Deploy mirato
- Se cambia solo il frontend → viene ricreato **solo `frontend`**
- Se cambia solo il backend → viene ricreato **solo `backend`**
- Se cambia `infra/` o `Jenkinsfile` → deploy completo


## Tecnologie utilizzate

- **Jenkins** (Multibranch Pipeline)
- **Docker & Docker Compose**
- **Angular**
- **Spring Boot (Java 17)**
- **GitHub**
- **Node.js / Maven** (in container)


## Avvio Jenkins in locale

Prerequisiti:
- Docker Desktop
- Git

```bash
cd jenkins-local
docker compose up -d --build
```

Jenkins sarà disponibile in locale: ```http://localhost:8080```

## Avvio applicazione

```bash
cd infra
docker-compose up -d
```
A questo punto saranno disponibili sia la Springboot Web API che Angular Web App ai seguenti indirizzi:
	•	Frontend: http://localhost:8082
	•	Backend:  http://localhost:8081