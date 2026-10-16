# 01 — Application Architecture and Environments

## Components

```
                ┌──────────────────────┐        ┌──────────────────────────┐
Mobile app ────►│                      │◄───────│ Admin UI (React + Vite)  │
(React Native)  │  Backend APIs        │        │ Azure Static Web App     │
Firebase push ◄─│  Node.js / Express   │        └──────────────────────────┘
                │  Azure App Service   │
                │  (VNet integrated)   │
                └──┬────────┬───────┬──┘
                   │        │       │
       private     │        │       │ HTTPS
       endpoint    ▼        ▼       ▼
   MySQL Flexible Server   Blob     Azure OpenAI (chat + embeddings)
   (private only)          Storage  Azure AI Translator
                                    ChromaDB on a Linux VM (vector search)
                   │
                   ▼
   Application Insights ──► Log Analytics workspace
```

| Component | Hosting | Notes |
|---|---|---|
| Backend APIs | Azure App Service, Linux, `NODE\|20-lts`, B2 plan | VNet-integrated through a delegated subnet; access restricted to VPN |
| Admin UI | Azure Static Web Apps (Standard) | SPA fallback rewrite to `/index.html` |
| Mobile app | Android / iOS stores, Firebase App Distribution for testers | Firebase Cloud Messaging for push |
| Database | MySQL Flexible Server 8.0.21, Burstable B1ms, 20 GB, 7-day backups | Public access disabled; private endpoint + private DNS |
| Vector store | ChromaDB in Docker on an Ubuntu 22.04 VM (Pinecone also tested) | Persistent volume, health check on `/api/v1/heartbeat` |
| AI | Azure OpenAI (`gpt-35-turbo-16k`, `text-embedding-3-large`), AI Translator (S1) | Summaries, embeddings, translation |
| Files | Blob storage container for article images | Cool tier |
| Secrets | Key Vault (soft delete 90 days, purge protection) | |
| Monitoring | Application Insights (workspace-based), Log Analytics, diagnostic settings | Winston logs shipped to App Insights |

## How the backend works

- **Scheduled jobs** (`node-cron`, driven by a cron-master table) pull articles from configured news sources.
- Articles are **translated** (AI Translator), **summarised** (Azure OpenAI) and **embedded**; embeddings go to the vector store and **similarity scores** are stored to group duplicate stories.
- Admins review and **publish** articles; published news triggers **notifications** to company groups through Firebase.
- Authentication with **JWT**, passwords hashed with **bcrypt**, email through SMTP (STARTTLS 1.2+).
- **Sequelize migrations** manage the schema (`npm run migrate` / `migrate:undo`).

### Data model (from the migrations)

| Area | Tables |
|---|---|
| Users and access | users, admin users, user devices, permissions, permission mappings, user preferences, activity logs |
| Companies | companies, user–company mapping, company–location mapping, locations, group–company mapping |
| Commercial | products, pricing, contracts, contract details, contract locations, contract–location mapping |
| News | news sources, source articles, published articles, article similarity scores, news settings, cron master |
| Notifications | notifications, notification groups, notification–company mapping, read status, notification logs, admin alerts |
| i18n | language translation |

## Environments

| Environment | Purpose | VNet CIDR |
|---|---|---|
| `dev` | Development | 10.1.0.0/16 |
| `devops` | DevOps sandbox for pipeline and IaC work | — |
| `stage` | Pre-production / UAT | 10.2.0.0/16 |
| `prod` | Production with custom domains | 10.3.0.0/16 |

Each environment VNet is **peered with a shared hub VNet** (where the VPN lives), so private resources are reachable only over VPN.

### Requirements raised for production

- **Custom domains** for the production endpoints.
- **An approval gate before production deploys**: a production deployment must not proceed until the named approvers sign it off (implemented as a manual approval / `input` step, or environment approvals).

## Application configuration

Each environment has its own `.env` file with the same keys; real values are stored as pipeline / App Service settings or Key Vault secrets, never committed.

| Key group | Keys |
|---|---|
| Runtime | `PORT`, `NODE_ENV` |
| Auth | `JWT_SECRET` |
| Database | `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_NAME`, `DATABASE_URL` |
| URLs | `SWAGGER_URL`, `FRONTEND_URL` |
| Email | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` |

### ChromaDB container

```yaml
services:
  server:
    image: chromadb/chroma:<pinned-version>
    volumes:
      - chroma-data:/chroma/chroma
    command: "--workers 1 --host 0.0.0.0 --port 8000 --proxy-headers --timeout-keep-alive 30"
    environment:
      - IS_PERSISTENT=TRUE
      - PERSIST_DIRECTORY=/chroma/chroma
    restart: unless-stopped
    ports:
      - "8888:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/heartbeat"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks: [ngl]
networks:
  ngl: { driver: bridge }
volumes:
  chroma-data: { driver: local }
```

Notes: the original health check called the **host** port (8888) from inside the container; inside the container the service listens on **8000**, so the check should target 8000. One compose file used `chroma:latest` — pin a version so a rebuild can't silently upgrade the database.

### Static Web App config

```json
{ "navigationFallback": { "rewrite": "/index.html" } }
```

Client-side routes (React Router) return `index.html` instead of 404 on refresh or deep links.
