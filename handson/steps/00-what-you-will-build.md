# Part 0 — What you will build

[Index](../README.md) | [Part 1 — Prerequisites and permission checks →](01-prerequisites-and-permissions.md)

---

## Architecture

```text
                              Internet (your browser)
                                       │  http(s)://<App Gateway public IP>
                                       ▼
┌──────────────────────────── Spoke VNet 10.1.0.0/16 ─────────────────────────────┐
│ snet-appgw 10.1.0.0/24                                                          │
│   Application Gateway WAF_v2  (Layer 7 load balancer + Web Application Firewall)│
│        │  AGIC keeps the backend pool = IPs of the frontend PODS                │
│        ▼                                                                        │
│ snet-aks 10.1.4.0/22   private AKS, Azure CNI (nodes AND pods get VNet IPs)     │
│   ┌── namespace "app" ─────────────────────────────────────────────────────┐    │
│   │ frontend pods (nginx + React) ──/api/──► backend Service (ClusterIP)   │    │
│   │                                            └──► backend pods (Spring)  │    │
│   └────────────────────────────────────────────────────────────────────────┘    │
│   private endpoint of the AKS API server                                        │
│        │ TCP 5432                         │ TCP 443 (Secrets Store CSI driver)  │
│        ▼                                  ▼                                     │
│ snet-private-endpoints 10.1.1.0/24                                              │
│   private endpoint → PostgreSQL Flexible Server   private endpoint → Key Vault  │
└──────────────────────────────────────▲──────────────────────────────────────────┘
                                       │ VNet peering (both directions)
┌──────────── Hub VNet 10.0.0.0/16 ────┴────────────┐
│ snet-jumpbox 10.0.1.0/24                          │
│   jumpbox VM = your admin box + GitHub runner     │◄── SSH only from YOUR public IP
└───────────────────────────────────────────────────┘

Private DNS zones (linked to BOTH VNets):
  privatelink.postgres.database.azure.com   privatelink.vaultcore.azure.net   privatelink.<region>.azmk8s.io

Outside the VNets: ACR, Log Analytics, Application Insights, alert action group, Terraform state storage
```

## The three tiers

| Tier | Technology | Runs in | Reached by |
|---|---|---|---|
| Presentation | React (Vite) served by nginx | AKS pods (`frontend`) | Internet → App Gateway → pod IP |
| Application | Java 21, Spring Boot 4 (REST API) | AKS pods (`backend`) | Only from frontend pods (NetworkPolicy) |
| Data | Azure Database for PostgreSQL Flexible Server 16 | Azure PaaS, public access **disabled** | Only through its private endpoint |

The app is a simple **Todo list**. `GET /api/info` returns the pod name, so you can watch load balancing work.

## How a request flows (learn this by heart)

1. The browser opens `http://<App Gateway public IP>/`.
2. **App Gateway** receives it on the listener for port 80. The **WAF** checks it against the OWASP rules.
3. The routing rule sends it to the backend pool. **AGIC** keeps that pool equal to the **frontend pod IPs**. App Gateway talks to the pods directly (this is why we use Azure CNI).
4. **nginx** in the frontend pod returns the React files. The browser then calls `/api/todos` on the same host.
5. App Gateway sends `/api/...` to a frontend pod as well. nginx **reverse-proxies** it to `http://ttapp-backend:8080`, the backend **ClusterIP Service**. CoreDNS resolves the name, and kube-proxy picks a backend pod.
6. The backend connects to `psql-ttapp-xxx.postgres.database.azure.com`. CoreDNS forwards to Azure DNS (168.63.129.16). The name resolves through a **CNAME** to `privatelink.postgres.database.azure.com`. The **private DNS zone** linked to the VNet answers with the **private endpoint IP** (10.1.1.x).
7. The NSG on the private-endpoint subnet allows 5432 only from the AKS and jumpbox subnets. PostgreSQL answers.

## How secrets flow

The pod's service account is linked to a **managed identity** (Workload Identity). The **Secrets Store CSI driver** uses that identity to read `db-url`, `db-username`, `db-password` and `appinsights-connection-string` from **Key Vault** through its **private endpoint**. It mounts them as files and also syncs them into a Kubernetes Secret, and the container reads that Secret as environment variables. **No password exists in Git, in GitHub, or in the image.**

## How a deployment flows

1. You push to `main`. The GitHub-hosted runner builds and tests the code, then runs SonarQube Cloud, Trivy and Gitleaks.
2. It logs in to Azure with **OIDC** (a federated credential, no client secret). It builds the images, scans them with Trivy, and pushes them to **ACR** tagged with the Git commit SHA.
3. The **deploy** job waits for your approval (GitHub environment `dev`). It then runs on the **self-hosted runner in the hub VNet**, because only machines inside the VNets can reach the **private API server**.
4. `helm upgrade --install --rollback-on-failure`. If the pods don't become Ready, Helm rolls back automatically.
5. A smoke test calls `http://<App Gateway IP>/api/info`.

## Two Terraform stacks (why?)

| Stack | What | Lifetime | Cost when idle |
|---|---|---|---|
| `foundation` | VNets, subnets, NSGs, peering, private DNS zones, Log Analytics, App Insights, ACR, Key Vault + private endpoint, jumpbox VM | Create once, keep | Small (stop the VM) |
| `platform` | AKS, App Gateway + WAF, PostgreSQL + private endpoint, identities, alerts | **Create at the start of each lab session, destroy at the end** | Zero once destroyed |

This split also gives you a good interview answer. Separate state files keep the blast radius small, and the stack that changes often is kept apart from the one that rarely changes. The platform stack reads the foundation outputs with `terraform_remote_state`.

## Repository layout you will create

```text
three-tier-aks/
├── .github/workflows/
│   ├── app-ci-cd.yml        # build, scan, push, deploy
│   └── infra.yml            # terraform validate / plan / apply / destroy
├── backend/                 # Spring Boot + Dockerfile
├── frontend/                # React + nginx + Dockerfile
├── helm/three-tier-app/     # Helm chart for both tiers
├── infra/
│   ├── foundation/          # Terraform stack 1
│   └── platform/            # Terraform stack 2
├── k8s/                     # monitoring values, alerts, VPA
├── docker-compose.yml       # run all three tiers on your laptop
├── .gitignore
└── .trivyignore
```

## Time plan (suggested)

| Session | Parts | Time |
|---|---|---|
| 1 | 0–3 (prerequisites, bootstrap, networking theory) | 2–3 h |
| 2 | 4–5 (foundation, jumpbox, runner) | 2–3 h |
| 3 | 6–7 (app code, Docker, local run) | 2–3 h |
| 4 | 8–10 (platform, Helm, first deploy) | 3–4 h |
| 5 | 11–13 (SonarQube Cloud, CI/CD, HTTPS, WAF) | 3–4 h |
| 6 | 14–15 (monitoring) | 2–3 h |
| 7–8 | 16–17 (exercises, break-and-fix) | 4–6 h |

**Destroy the platform stack at the end of every session** ([Part 18](18-start-stop-cost-teardown.md)).

---

[Index](../README.md) | [Part 1 — Prerequisites and permission checks →](01-prerequisites-and-permissions.md)
