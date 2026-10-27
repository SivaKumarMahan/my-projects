# Hands-on: Three-tier application on private AKS (end to end)

This guide is a lab you build yourself, step by step. At the end you will have **done** everything you currently explain only in theory in interviews:

- **Terraform**: remote state, two stacks, every Azure resource
- **Networking**: hub-spoke VNets, subnets, NSGs, peering, private endpoints, private DNS zones, Application Gateway + WAF, load balancers, Ingress
- **Private AKS**: Entra ID login, Azure CNI, Calico network policies, probes, HPA, VPA, Cluster Autoscaler, PDB, upgrades
- **Application**: React frontend, Spring Boot backend, PostgreSQL, multistage Dockerfiles, Helm chart
- **CI/CD**: GitHub Actions with OIDC (no passwords), SonarQube Cloud, Trivy, Gitleaks, ACR, self-hosted runner
- **Security**: Key Vault + Secrets Store CSI driver + Workload Identity, WAF, non-root containers
- **Monitoring**: Azure Monitor (Container Insights, Application Insights, Log Analytics, alerts) and Prometheus + Grafana
- **Operations**: rollback, zero-downtime deploys, secret rotation, AKS upgrade, and a **break-and-fix lab** for the troubleshooting questions

The topics come from the questions in your `interview-questions` repository (the `real-interview-questions` and `repetitive-questions` folders). [Part 19](#part-19--interview-questions--where-you-did-it) maps each question to the step where you did it.

> **How this guide was tested.** All code in this guide was checked before it was written down. Both Terraform stacks pass `terraform validate` with azurerm 5.11. The Spring Boot backend builds and its tests pass. The frontend and backend images were built and run together with PostgreSQL. Trivy, Gitleaks, Helm lint, kubeconform and actionlint all pass. Nothing was deployed to Azure, so the Azure steps can still fail because of your subscription's policies, quotas or region. The [blockers appendix](#appendix-a--blockers-and-the-easy-way-around) lists the likely problems and the easy way around each one.

---

## Table of contents

- [Part 0 — What you will build](#part-0--what-you-will-build)
- [Part 1 — Prerequisites and permission checks](#part-1--prerequisites-and-permission-checks)
- [Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities](#part-2--bootstrap-state-storage-github-repo-oidc-identities)
- [Part 3 — Networking explained (read before Terraform)](#part-3--networking-explained-read-before-terraform)
- [Part 4 — Terraform stack 1: foundation](#part-4--terraform-stack-1-foundation)
- [Part 5 — Jumpbox and GitHub self-hosted runner](#part-5--jumpbox-and-github-self-hosted-runner)
- [Part 6 — The application code](#part-6--the-application-code)
- [Part 7 — Multistage Dockerfiles and local scans](#part-7--multistage-dockerfiles-and-local-scans)
- [Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL)](#part-8--terraform-stack-2-platform-aks-app-gateway-postgresql)
- [Part 9 — The Helm chart](#part-9--the-helm-chart)
- [Part 10 — First deployment by hand](#part-10--first-deployment-by-hand)
- [Part 11 — SonarQube Cloud](#part-11--sonarqube-cloud)
- [Part 12 — GitHub Actions CI/CD](#part-12--github-actions-cicd)
- [Part 13 — HTTPS and WAF](#part-13--https-and-waf)
- [Part 14 — Monitoring with Azure Monitor](#part-14--monitoring-with-azure-monitor)
- [Part 15 — Monitoring with Prometheus and Grafana](#part-15--monitoring-with-prometheus-and-grafana)
- [Part 16 — Kubernetes exercises](#part-16--kubernetes-exercises)
- [Part 17 — Break-and-fix lab](#part-17--break-and-fix-lab)
- [Part 18 — Daily start/stop, cost and teardown](#part-18--daily-startstop-cost-and-teardown)
- [Part 19 — Interview questions → where you did it](#part-19--interview-questions--where-you-did-it)
- [Appendix A — Blockers and the easy way around](#appendix-a--blockers-and-the-easy-way-around)
- [Appendix B — Adding QA and Prod later](#appendix-b--adding-qa-and-prod-later)
- [Appendix C — Optional upgrades](#appendix-c--optional-upgrades)
- [Appendix D — Command cheat sheet](#appendix-d--command-cheat-sheet)

---

## Part 0 — What you will build

### Architecture

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

### The three tiers

| Tier | Technology | Runs in | Reached by |
|---|---|---|---|
| Presentation | React (Vite) served by nginx | AKS pods (`frontend`) | Internet → App Gateway → pod IP |
| Application | Java 21, Spring Boot 4 (REST API) | AKS pods (`backend`) | Only from frontend pods (NetworkPolicy) |
| Data | Azure Database for PostgreSQL Flexible Server 16 | Azure PaaS, public access **disabled** | Only through its private endpoint |

The app is a simple **Todo list**. `GET /api/info` returns the pod name, so you can watch load balancing work.

### How a request flows (learn this by heart)

1. The browser opens `http://<App Gateway public IP>/`.
2. **App Gateway** receives it on the listener for port 80. The **WAF** checks it against the OWASP rules.
3. The routing rule sends it to the backend pool. **AGIC** keeps that pool equal to the **frontend pod IPs**. App Gateway talks to the pods directly (this is why we use Azure CNI).
4. **nginx** in the frontend pod returns the React files. The browser then calls `/api/todos` on the same host.
5. App Gateway sends `/api/...` to a frontend pod as well. nginx **reverse-proxies** it to `http://ttapp-backend:8080`, the backend **ClusterIP Service**. CoreDNS resolves the name, and kube-proxy picks a backend pod.
6. The backend connects to `psql-ttapp-xxx.postgres.database.azure.com`. CoreDNS forwards to Azure DNS (168.63.129.16). The name resolves through a **CNAME** to `privatelink.postgres.database.azure.com`. The **private DNS zone** linked to the VNet answers with the **private endpoint IP** (10.1.1.x).
7. The NSG on the private-endpoint subnet allows 5432 only from the AKS and jumpbox subnets. PostgreSQL answers.

### How secrets flow

The pod's service account is linked to a **managed identity** (Workload Identity). The **Secrets Store CSI driver** uses that identity to read `db-url`, `db-username`, `db-password` and `appinsights-connection-string` from **Key Vault** through its **private endpoint**. It mounts them as files and also syncs them into a Kubernetes Secret, and the container reads that Secret as environment variables. **No password exists in Git, in GitHub, or in the image.**

### How a deployment flows

1. You push to `main`. The GitHub-hosted runner builds and tests the code, then runs SonarQube Cloud, Trivy and Gitleaks.
2. It logs in to Azure with **OIDC** (a federated credential, no client secret). It builds the images, scans them with Trivy, and pushes them to **ACR** tagged with the Git commit SHA.
3. The **deploy** job waits for your approval (GitHub environment `dev`). It then runs on the **self-hosted runner in the hub VNet**, because only machines inside the VNets can reach the **private API server**.
4. `helm upgrade --install --rollback-on-failure`. If the pods don't become Ready, Helm rolls back automatically.
5. A smoke test calls `http://<App Gateway IP>/api/info`.

### Two Terraform stacks (why?)

| Stack | What | Lifetime | Cost when idle |
|---|---|---|---|
| `foundation` | VNets, subnets, NSGs, peering, private DNS zones, Log Analytics, App Insights, ACR, Key Vault + private endpoint, jumpbox VM | Create once, keep | Small (stop the VM) |
| `platform` | AKS, App Gateway + WAF, PostgreSQL + private endpoint, identities, alerts | **Create at the start of each lab session, destroy at the end** | Zero once destroyed |

This split also gives you a good interview answer. Separate state files keep the blast radius small, and the stack that changes often is kept apart from the one that rarely changes. The platform stack reads the foundation outputs with `terraform_remote_state`.

### Repository layout you will create

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

### Time plan (suggested)

| Session | Parts | Time |
|---|---|---|
| 1 | 0–3 (prerequisites, bootstrap, networking theory) | 2–3 h |
| 2 | 4–5 (foundation, jumpbox, runner) | 2–3 h |
| 3 | 6–7 (app code, Docker, local run) | 2–3 h |
| 4 | 8–10 (platform, Helm, first deploy) | 3–4 h |
| 5 | 11–13 (SonarQube Cloud, CI/CD, HTTPS, WAF) | 3–4 h |
| 6 | 14–15 (monitoring) | 2–3 h |
| 7–8 | 16–17 (exercises, break-and-fix) | 4–6 h |

**Destroy the platform stack at the end of every session** ([Part 18](#part-18--daily-startstop-cost-and-teardown)).

---

## Part 1 — Prerequisites and permission checks

### 1.1 Tools on your laptop

| Tool | Why | Check |
|---|---|---|
| Azure CLI 2.60+ | everything Azure | `az version` |
| Terraform 1.9+ (tested with 1.17) | infrastructure | `terraform version` |
| Docker | build and run images locally | `docker version` |
| Git + a GitHub account | code and pipelines | `git --version` |
| Node.js 22+ (24 recommended) | create `package-lock.json` | `node -v` |
| An SSH key | log in to the jumpbox | `ls ~/.ssh/id_ed25519.pub` (create with `ssh-keygen -t ed25519`) |
| Java/Maven (optional) | not needed; Docker builds the backend | |

`kubectl`, `kubelogin` and `helm` are only needed **on the jumpbox**, because the cluster is private. cloud-init installs them there.

### 1.2 One file with all your names

Create `~/three-tier-env.sh` and run `source ~/three-tier-env.sh` in every new terminal:

```bash
# ---- change these ----
export SUBSCRIPTION_ID="<your-subscription-id>"
export RG="<your-existing-resource-group>"
export SUFFIX="sk01"                  # 3-6 lowercase letters/digits, makes global names unique
export GH_OWNER="<your-github-username>"
export GH_REPO="three-tier-aks"
# ---- derived, do not change ----
export LOCATION=$(az group show -n "$RG" --query location -o tsv)
export PREFIX="ttapp"
export TFSTATE_SA="sttfstate${SUFFIX}"
export ACR_NAME="acr${PREFIX}${SUFFIX}"
export KV_NAME="kv-${PREFIX}-${SUFFIX}"
export PG_NAME="psql-${PREFIX}-${SUFFIX}"
export AKS_NAME="aks-${PREFIX}-dev"
export APPGW_NAME="agw-${PREFIX}-dev"
```

> `LOCATION` comes from your resource group, so everything lands in the same region. It must be the short name (`centralindia`, not `Central India`). The private DNS zone for AKS is `privatelink.<location>.azmk8s.io`.

### 1.3 Log in and check your role

```bash
az login
az account set --subscription "$SUBSCRIPTION_ID"
source ~/three-tier-env.sh

export MY_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
echo "My object ID: $MY_OBJECT_ID"        # add this line to three-tier-env.sh too

az role assignment list --assignee "$MY_OBJECT_ID" --resource-group "$RG" \
  --include-inherited --query "[].roleDefinitionName" -o tsv
```

Expected: `Contributor` and `User Access Administrator`. With those two roles you can:

- create every resource in this guide **inside your resource group**
- create **role assignments** at RG scope or below, which managed identities need
- create **managed identities and federated credentials**. This replaces app registrations in Entra ID, which you may not be allowed to create.

You **cannot**:

- create resource groups (we reuse yours)
- register resource providers (checked in 1.4)
- purge a deleted Key Vault (handled in the Terraform code)

### 1.4 Resource providers (the only possible admin request)

A resource provider must be registered once per subscription, and that needs subscription-level rights. Check them:

```bash
for p in Microsoft.Network Microsoft.Compute Microsoft.ContainerService Microsoft.ContainerRegistry \
         Microsoft.KeyVault Microsoft.DBforPostgreSQL Microsoft.ManagedIdentity Microsoft.Storage \
         Microsoft.OperationalInsights Microsoft.Insights Microsoft.AlertsManagement Microsoft.DevTestLab; do
  printf "%-32s %s\n" "$p" "$(az provider show -n $p --query registrationState -o tsv 2>/dev/null || echo 'cannot read')"
done
```

- All `Registered`: good.
- Any `NotRegistered`: send the list to your subscription admin (`az provider register -n <name>`). This is a one-time request.
- `cannot read`: you can't see the registration state. Just continue. If an apply fails with `MissingSubscriptionRegistration`, that is the provider to ask for.
- `Microsoft.DevTestLab` is only used for the VM auto-shutdown schedule. If it is missing, delete `azurerm_dev_test_global_vm_shutdown_schedule` from `jumpbox.tf`.

> Terraform is told **not** to register providers (`resource_provider_registrations = "none"`). Otherwise it would try, and fail with "authorization failed" on a resource-group-only role.

### 1.5 Quota and policy (quick check)

```bash
# vCPU quota in your region (may fail without subscription read access; that is fine)
az vm list-usage -l "$LOCATION" -o table 2>/dev/null | grep -iE "Total Regional vCPUs|DSv5|BS Family"

# Azure Policy assignments that apply to your RG (deny rules can block public IPs, regions, SKUs)
az policy assignment list --resource-group "$RG" --query "[].displayName" -o tsv
```

The lab needs about **10 vCPUs** at peak: 1 system node and up to 3 user nodes, each `Standard_D2s_v5` (2 vCPU), plus a `Standard_B2s` jumpbox. If a policy denies **public IPs**, read [Appendix A](#appendix-a--blockers-and-the-easy-way-around) first.

---

## Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities

Terraform cannot store its state in a storage account that Terraform itself creates (chicken-and-egg), and the pipeline identities must exist before any pipeline runs. So these few things are created with the Azure CLI. This is also a good interview point: "the backend and pipeline identities are bootstrapped once; everything else is code".

### 2.1 Terraform state storage account

```bash
source ~/three-tier-env.sh

# Entra ID only (no access keys), TLS 1.2, no public blobs
az storage account create -n "$TFSTATE_SA" -g "$RG" -l "$LOCATION" \
  --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2 \
  --allow-blob-public-access false --allow-shared-key-access false

# Versioning + soft delete = you can recover a broken or deleted state file
az storage account blob-service-properties update -n "$TFSTATE_SA" -g "$RG" \
  --enable-versioning true --enable-delete-retention true --delete-retention-days 7

export SA_ID=$(az storage account show -n "$TFSTATE_SA" -g "$RG" --query id -o tsv)

# You need a DATA-plane role to read/write blobs (Contributor is control-plane only)
az role assignment create --assignee "$MY_OBJECT_ID" --role "Storage Blob Data Contributor" --scope "$SA_ID"

sleep 60   # role assignments take a minute to work
az storage container create -n tfstate --account-name "$TFSTATE_SA" --auth-mode login
```

**State locking.** The azurerm backend locks the state with a **blob lease**. While `plan` or `apply` runs, the blob shows `leaseState: leased`. You will see this in [Part 17](#lab-11--terraform-state-lock).

### 2.2 Create the GitHub repository

1. On GitHub, create a **public** repository named `three-tier-aks`. Public keeps SonarQube Cloud and environment approvals free. Never commit secrets; Gitleaks will check that for you.
2. Clone it:

```bash
cd ~ && git clone https://github.com/$GH_OWNER/$GH_REPO.git && cd $GH_REPO
mkdir -p .github/workflows backend frontend helm infra/foundation infra/platform k8s/monitoring
```

### 2.3 Two managed identities for GitHub Actions (OIDC)

We use **user-assigned managed identities with federated credentials**, so you don't need Entra ID app-registration rights. Each identity trusts GitHub's OIDC token for specific subjects only.

| Identity | Used by | Roles |
|---|---|---|
| `id-gh-infra` | `infra.yml` (Terraform) | Contributor + Role Based Access Control Administrator on the RG, Storage Blob Data Contributor on the state account, Key Vault Secrets Officer (Terraform adds this) |
| `id-gh-app` | `app-ci-cd.yml` | Reader on the RG, AcrPush on ACR, AKS Cluster User + AKS RBAC Admin **only on namespace `app`** (Terraform adds the AKS roles) |

Two identities instead of one is **least privilege**. The app pipeline can't change infrastructure, and the infra pipeline can't deploy pods.

```bash
source ~/three-tier-env.sh
export RG_ID=$(az group show -n "$RG" --query id -o tsv)

az identity create -n id-gh-infra -g "$RG" -l "$LOCATION"
az identity create -n id-gh-app   -g "$RG" -l "$LOCATION"

export INFRA_CLIENT_ID=$(az identity show -n id-gh-infra -g "$RG" --query clientId -o tsv)
export INFRA_PRINCIPAL_ID=$(az identity show -n id-gh-infra -g "$RG" --query principalId -o tsv)
export APP_CLIENT_ID=$(az identity show -n id-gh-app -g "$RG" --query clientId -o tsv)
export APP_PRINCIPAL_ID=$(az identity show -n id-gh-app -g "$RG" --query principalId -o tsv)
export TENANT_ID=$(az account show --query tenantId -o tsv)

# Save them for later sessions
cat >> ~/three-tier-env.sh <<EOF
export MY_OBJECT_ID=$MY_OBJECT_ID
export INFRA_CLIENT_ID=$INFRA_CLIENT_ID
export INFRA_PRINCIPAL_ID=$INFRA_PRINCIPAL_ID
export APP_CLIENT_ID=$APP_CLIENT_ID
export APP_PRINCIPAL_ID=$APP_PRINCIPAL_ID
export TENANT_ID=$TENANT_ID
EOF

# Roles for the infra identity
for role in "Contributor" "Role Based Access Control Administrator"; do
  az role assignment create --assignee-object-id "$INFRA_PRINCIPAL_ID" \
    --assignee-principal-type ServicePrincipal --role "$role" --scope "$RG_ID"
done
az role assignment create --assignee-object-id "$INFRA_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --role "Storage Blob Data Contributor" --scope "$SA_ID"

# Reader for the app identity (to look up names/IDs during deploy)
az role assignment create --assignee-object-id "$APP_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal --role "Reader" --scope "$RG_ID"
```

Now the **federated credentials**. The `subject` must match **exactly** what GitHub puts in its token:

| Workflow situation | Subject GitHub sends |
|---|---|
| job with `environment: dev` | `repo:OWNER/REPO:environment:dev` |
| pull request | `repo:OWNER/REPO:pull_request` |
| push to main (no environment) | `repo:OWNER/REPO:ref:refs/heads/main` |

```bash
ISSUER="https://token.actions.githubusercontent.com"
AUD="api://AzureADTokenExchange"

az identity federated-credential create -g "$RG" --identity-name id-gh-infra -n gh-pr \
  --issuer "$ISSUER" --audiences "$AUD" --subject "repo:$GH_OWNER/$GH_REPO:pull_request"
az identity federated-credential create -g "$RG" --identity-name id-gh-infra -n gh-env-dev \
  --issuer "$ISSUER" --audiences "$AUD" --subject "repo:$GH_OWNER/$GH_REPO:environment:dev"

az identity federated-credential create -g "$RG" --identity-name id-gh-app -n gh-main \
  --issuer "$ISSUER" --audiences "$AUD" --subject "repo:$GH_OWNER/$GH_REPO:ref:refs/heads/main"
az identity federated-credential create -g "$RG" --identity-name id-gh-app -n gh-env-dev \
  --issuer "$ISSUER" --audiences "$AUD" --subject "repo:$GH_OWNER/$GH_REPO:environment:dev"
```

**How OIDC login works (interview answer):** the workflow asks GitHub for a signed OIDC token (`permissions: id-token: write`). `azure/login` sends it to Entra ID. Entra ID checks the issuer, audience and subject against the federated credential, then returns a short-lived Azure access token for the managed identity. No secret is stored anywhere, so there is nothing to rotate or leak.

### 2.4 GitHub settings

In the repo, open **Settings → Secrets and variables → Actions**.

**Variables** tab:

| Name | Value |
|---|---|
| `AZURE_TENANT_ID` | `echo $TENANT_ID` |
| `AZURE_SUBSCRIPTION_ID` | `echo $SUBSCRIPTION_ID` |
| `AZURE_INFRA_CLIENT_ID` | `echo $INFRA_CLIENT_ID` |
| `AZURE_APP_CLIENT_ID` | `echo $APP_CLIENT_ID` |
| `RESOURCE_GROUP` | `echo $RG` |
| `ACR_NAME` | `echo $ACR_NAME` |
| `AKS_NAME` | `echo $AKS_NAME` |
| `SONAR_ORG` | set in [Part 11](#part-11--sonarqube-cloud) |
| `SERVICE_MONITOR_ENABLED` | `false` (set `true` in Part 15) |
| `TLS_ENABLED` | `false` (set `true` in Part 13) |

**Secrets** tab:

| Name | Value |
|---|---|
| `TF_VAR_MY_IP` | your public IP: `curl -s ifconfig.me` |
| `TF_VAR_ALERT_EMAIL` | your email |
| `SONAR_TOKEN` | set in Part 11 |

**Settings → Environments → New environment `dev`:**

- **Required reviewers**: add yourself. Every deploy and every Terraform apply now waits for your click (manual approval).
- **Deployment branches and tags**: *Selected branches* → `main`.

**Settings → Actions → General → Fork pull request workflows**: choose **Require approval for all external contributors**. A public repo plus a self-hosted runner means a stranger's PR must never run on your VM. Our workflows also never send fork PRs to the self-hosted runner.

**Branch protection** (do it after the first workflow run in Part 12, when the check names exist): **Settings → Rules → Rulesets → New branch ruleset** for `main`:

- Require a pull request before merging
- Require status checks to pass: `secret-scan`, `backend`, `frontend`, `helm-scan`, `validate (foundation)`, `validate (platform)`
- Block force pushes

---

## Part 3 — Networking explained (read before Terraform)

Read this part slowly. Most interview questions about networking are answered by **this** design.

### 3.1 Address plan

| VNet / subnet | CIDR | What lives there | Why this size |
|---|---|---|---|
| **Hub** VNet | 10.0.0.0/16 | shared services | room to grow (Firewall, Bastion, VPN gateway later) |
| ├ snet-jumpbox | 10.0.1.0/24 | jumpbox + GitHub runner VM | |
| **Spoke** VNet | 10.1.0.0/16 | the application | |
| ├ snet-appgw | 10.1.0.0/24 | Application Gateway v2 only | App Gateway needs its **own dedicated** subnet; /24 is the recommended size |
| ├ snet-private-endpoints | 10.1.1.0/24 | private endpoints (PostgreSQL, Key Vault) | one IP per endpoint |
| ├ snet-aks | 10.1.4.0/22 | AKS nodes **and pods** | Azure CNI gives every pod a VNet IP: (nodes + surge) × (max_pods 30 + 1) ≈ 160 IPs during an upgrade; /22 = 1024 |
| AKS service CIDR | 10.100.0.0/16 | ClusterIP Services (virtual) | must **not** overlap any VNet or on-prem range |
| AKS DNS service IP | 10.100.0.10 | CoreDNS Service | must be inside the service CIDR |

**Avoiding overlapping CIDRs (an interview question).** Plan all ranges centrally (an IPAM sheet), and never overlap peered VNets, on-prem ranges, or the AKS service CIDR. Fixing an overlap later means recreating the VNet or cluster, because peering refuses overlapping ranges.

### 3.2 Components and their job

| Component | Layer | Job in this lab |
|---|---|---|
| **VNet** | network | private address space; nothing gets in from the internet unless it has a public IP |
| **Subnet** | network | a slice of the VNet; NSGs and route tables attach here |
| **NSG** | L3/L4 firewall | allow/deny by source, destination, port. Rules are stateful (return traffic is automatic) |
| **VNet peering** | network | connects hub and spoke over the Microsoft backbone; **not transitive** (A-B and B-C does not give A-C) |
| **Private endpoint** | network | a NIC with a **private IP from your subnet** that represents one PaaS resource (PostgreSQL, Key Vault) |
| **Private DNS zone** | DNS | makes the PaaS hostname resolve to that private IP, **but only for VNets linked to the zone** |
| **Application Gateway** | **L7** load balancer | HTTP/HTTPS: host/path routing, TLS termination, cookie affinity, health probes |
| **WAF** (policy on App Gateway) | L7 security | OWASP rules (SQL injection, XSS…) + custom rules; Prevention mode blocks with 403 |
| **Azure Load Balancer** (Standard) | **L4** | TCP/UDP only. AKS uses it for **outbound** (SNAT) traffic and for `Service type: LoadBalancer`. You build an internal one in Part 16 |
| **AGIC** | Kubernetes controller | watches Ingress objects and **writes the App Gateway configuration** through ARM |
| **Ingress** | Kubernetes object | "send traffic for this host/path to this Service" |
| **Ingress controller** | the software that implements Ingress | here AGIC (App Gateway). The alternative is NGINX Ingress behind an internal LB |
| **Azure CNI** | network plugin | pods get **real VNet IPs**, so App Gateway can reach them directly |
| **Calico** | network policy engine | enforces Kubernetes `NetworkPolicy` (pod-level firewall) |
| **CoreDNS** | cluster DNS | resolves Service names; forwards everything else to Azure DNS 168.63.129.16 |

### 3.3 Why the PostgreSQL hostname resolves to a private IP

```text
pod → CoreDNS → Azure DNS (168.63.129.16)
   psql-ttapp-sk01.postgres.database.azure.com
     CNAME psql-ttapp-sk01.privatelink.postgres.database.azure.com
       → private DNS zone "privatelink.postgres.database.azure.com" (linked to spoke + hub)
         A record 10.1.1.5  (created by the private endpoint's DNS zone group)
```

Run the same lookup **from your laptop** and you get the public IP. Because public access is **disabled**, the connection is refused there. That is the whole point: the database is reachable only from inside the VNets. You will see this yourself in Part 8.

**If the DNS zone is not linked to a VNet, that VNet gets the public IP and the connection fails.** This is one of the most common production issues ([Lab 8](#lab-8--private-dns-zone-link-removed)).

### 3.4 Private AKS: how do we reach the API server?

- The API server gets a **private endpoint** in snet-aks, so there is no public endpoint.
- Its name lives in our **own** private DNS zone `privatelink.<region>.azmk8s.io` (a "custom" private DNS zone). That zone is linked to **both** VNets, so the jumpbox in the hub can resolve it.
- AKS writes the A record into the zone. That's why its control-plane identity needs **Private DNS Zone Contributor** on the zone, and **Network Contributor** on the spoke VNet so it can use the subnet.
- **Who can call it?** Only machines that can route to it (the jumpbox and the runner, over peering) **and** that have an Entra ID role on the cluster (Azure RBAC for Kubernetes).
- **How the pipeline deploys:** a self-hosted runner inside the hub. The alternative, `az aks command invoke`, runs commands through the Azure API without any network path; it is the fallback in Appendix A.

### 3.5 What each NSG allows

| NSG on | Inbound rules (plus Azure defaults) | Why |
|---|---|---|
| snet-appgw | GatewayManager → 65200-65535; Internet → 80, 443 | App Gateway v2 **will not deploy** without the GatewayManager rule |
| snet-aks | App Gateway subnet → 8080 (documentation; the default AllowVnetInBound already allows it) | AGIC sends traffic straight to pod port 8080 |
| snet-private-endpoints | AKS + jumpbox → 5432; VirtualNetwork → 443; **deny everything else from VirtualNetwork** | only these sources can talk to the database |
| snet-jumpbox | **your IP only** → 22 | SSH |

> NSGs only apply to private endpoints if the subnet has `private_endpoint_network_policies = "Enabled"`. Many people miss this.

### 3.6 Outbound traffic (egress)

- **AKS nodes**: `outbound_type = loadBalancer`. The AKS Standard Load Balancer does SNAT through a public IP. Alternatives: a NAT Gateway (more SNAT ports), or `userDefinedRouting` through Azure Firewall (central egress control). See Appendix C.
- **Jumpbox**: has its own public IP.
- **App Gateway**: has its own public IP.

### 3.7 Why App Gateway + AGIC (and not NGINX Ingress)?

| | AGIC + App Gateway (this lab) | NGINX Ingress + internal LB |
|---|---|---|
| Where the L7 proxy runs | outside the cluster (managed PaaS) | inside the cluster (pods) |
| WAF | built in (WAF_v2) | needs ModSecurity or a WAF in front |
| Hops | App Gateway → pod IP directly | LB → NGINX pod → app pod |
| Config | Ingress → AGIC → ARM API (slower to apply: tens of seconds) | instant |
| Needs | Azure CNI (routable pod IPs) | any CNI |
| Cost | App Gateway hourly | only node resources |

Interview line: *"We used App Gateway WAF_v2 with the AGIC add-on, because we needed a managed WAF at the edge and wanted App Gateway to send traffic directly to pod IPs. Inside the cluster, NetworkPolicies allow only the App Gateway subnet to reach the frontend."*

---

## Part 4 — Terraform stack 1: foundation

Create every file below in `infra/foundation/`. Read the comments in each file; they are the explanation.

### 4.1 Provider and backend — `versions.tf`

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.11"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.13"
    }
  }

  # Values come from backend.hcl at "terraform init" time (partial configuration).
  backend "azurerm" {}
}

provider "azurerm" {
  features {
    key_vault {
      # Purging a deleted vault needs a subscription-level permission we do not have.
      purge_soft_delete_on_destroy = false
    }
  }

  subscription_id = var.subscription_id

  # We only have rights on one resource group, so never try to register providers.
  resource_provider_registrations = "none"

  # Use Entra ID (not storage keys) for data-plane calls to storage.
  storage_use_azuread = true
}
```

Key points:

- `backend "azurerm" {}` is empty because the values come from `backend.hcl` at `init` time ("partial configuration"). The same code can then point at a different state file per environment.
- `resource_provider_registrations = "none"`: azurerm 5.x already defaults to this. We set it explicitly so it is visible.
- `storage_use_azuread = true`: Terraform talks to storage with Entra ID, not account keys (we disabled keys).

### 4.2 Backend config — `backend.hcl`

```hcl
resource_group_name  = "rg-your-existing-rg" # echo $RG
storage_account_name = "sttfstatesk01"       # echo $TFSTATE_SA
container_name       = "tfstate"
key                  = "foundation.tfstate"
use_azuread_auth     = true
```

Replace the two values with `echo $RG` and `echo $TFSTATE_SA`.

### 4.3 Variables — `variables.tf`

```hcl
variable "subscription_id" {
  type        = string
  description = "Subscription that holds your resource group."
}

variable "resource_group_name" {
  type        = string
  description = "Existing resource group where you have Contributor + User Access Administrator."
}

variable "location" {
  type        = string
  description = "Azure region, for example centralindia or eastus2."
}

variable "prefix" {
  type        = string
  description = "Short name used in every resource name."
  default     = "ttapp"
}

variable "suffix" {
  type        = string
  description = "3-6 lowercase letters/digits to make global names unique (ACR, Key Vault, PostgreSQL)."

  validation {
    condition     = can(regex("^[a-z0-9]{3,6}$", var.suffix))
    error_message = "suffix must be 3-6 lowercase letters or digits."
  }
}

variable "my_ip" {
  type        = string
  description = "Your public IP (curl -s ifconfig.me). Allowed to SSH to the jumpbox and to reach Key Vault."
}

variable "ssh_public_key" {
  type        = string
  description = "Contents of your ~/.ssh/id_ed25519.pub (or id_rsa.pub)."
}

variable "alert_email" {
  type        = string
  description = "Email address that receives Azure Monitor alerts."
}

variable "kv_admin_object_ids" {
  type        = list(string)
  description = "Object IDs that may read/write Key Vault secrets: you and the GitHub infra identity."
}

variable "github_app_principal_id" {
  type        = string
  description = "Principal ID of the GitHub app identity (id-gh-app). Gets AcrPush."
}

variable "hub_address_space" {
  type    = string
  default = "10.0.0.0/16"
}

variable "spoke_address_space" {
  type    = string
  default = "10.1.0.0/16"
}

variable "tags" {
  type = map(string)
  default = {
    project     = "three-tier-aks"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

### 4.4 `main.tf`

```hcl
data "azurerm_client_config" "current" {}

# We cannot create resource groups, so we read the one we were given.
data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

locals {
  rg       = data.azurerm_resource_group.rg.name
  location = var.location
  name     = "${var.prefix}-dev"
}
```

We **read** the resource group with a `data` source, because we are not allowed to create resource groups.

### 4.5 Network — `network.tf`

```hcl
locals {
  # Hub: shared things (jumpbox + GitHub runner).
  snet_jumpbox_cidr = cidrsubnet(var.hub_address_space, 8, 1) # 10.0.1.0/24

  # Spoke: the application.
  snet_appgw_cidr = cidrsubnet(var.spoke_address_space, 8, 0) # 10.1.0.0/24
  snet_pe_cidr    = cidrsubnet(var.spoke_address_space, 8, 1) # 10.1.1.0/24
  snet_aks_cidr   = cidrsubnet(var.spoke_address_space, 6, 1) # 10.1.4.0/22 (nodes + pods with Azure CNI)
}

# ---------- Hub VNet ----------
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-${local.name}-hub"
  resource_group_name = local.rg
  location            = local.location
  address_space       = [var.hub_address_space]
  tags                = var.tags
}

resource "azurerm_subnet" "jumpbox" {
  name                 = "snet-jumpbox"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = [local.snet_jumpbox_cidr]
}

# ---------- Spoke VNet ----------
resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-${local.name}-spoke"
  resource_group_name = local.rg
  location            = local.location
  address_space       = [var.spoke_address_space]
  tags                = var.tags
}

resource "azurerm_subnet" "appgw" {
  name                 = "snet-appgw"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_appgw_cidr]
}

resource "azurerm_subnet" "pe" {
  name                 = "snet-private-endpoints"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_pe_cidr]

  # Without this, NSG rules are NOT applied to private endpoints.
  private_endpoint_network_policies = "Enabled"
}

resource "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_aks_cidr]
}

# ---------- Peering (both directions are required) ----------
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  name                      = "peer-hub-to-spoke"
  resource_group_name       = local.rg
  virtual_network_name      = azurerm_virtual_network.hub.name
  remote_virtual_network_id = azurerm_virtual_network.spoke.id
  allow_forwarded_traffic   = false
}

resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                      = "peer-spoke-to-hub"
  resource_group_name       = local.rg
  virtual_network_name      = azurerm_virtual_network.spoke.name
  remote_virtual_network_id = azurerm_virtual_network.hub.id
  allow_forwarded_traffic   = false
}

# ---------- NSGs ----------
resource "azurerm_network_security_group" "appgw" {
  name                = "nsg-${local.name}-appgw"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  # Application Gateway v2 health/management traffic. Without it the gateway fails.
  security_rule {
    name                       = "Allow-GatewayManager"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "GatewayManager"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "65200-65535"
  }

  security_rule {
    name                       = "Allow-Internet-HTTP-HTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "Internet"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_ranges    = ["80", "443"]
  }
}

resource "azurerm_network_security_group" "aks" {
  name                = "nsg-${local.name}-aks"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  # Documents the one flow that must work: App Gateway -> pod IPs.
  # The default AllowVnetInBound rule would allow it anyway.
  security_rule {
    name                       = "Allow-AppGw-To-Pods"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = local.snet_appgw_cidr
    source_port_range          = "*"
    destination_address_prefix = local.snet_aks_cidr
    destination_port_range     = "8080"
  }
}

resource "azurerm_network_security_group" "pe" {
  name                = "nsg-${local.name}-pe"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  security_rule {
    name                       = "Allow-Postgres-From-AKS-And-Jumpbox"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefixes    = [local.snet_aks_cidr, local.snet_jumpbox_cidr]
    source_port_range          = "*"
    destination_address_prefix = local.snet_pe_cidr
    destination_port_range     = "5432"
  }

  security_rule {
    name                       = "Allow-KeyVault-From-VNet"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "VirtualNetwork"
    source_port_range          = "*"
    destination_address_prefix = local.snet_pe_cidr
    destination_port_range     = "443"
  }

  security_rule {
    name                       = "Deny-All-Other-VNet-Inbound"
    priority                   = 4000
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_address_prefix      = "VirtualNetwork"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "*"
  }
}

resource "azurerm_network_security_group" "jumpbox" {
  name                = "nsg-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  security_rule {
    name                       = "Allow-SSH-From-My-IP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "${var.my_ip}/32"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "22"
  }
}

resource "azurerm_subnet_network_security_group_association" "appgw" {
  subnet_id                 = azurerm_subnet.appgw.id
  network_security_group_id = azurerm_network_security_group.appgw.id
}

resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks.id
}

resource "azurerm_subnet_network_security_group_association" "pe" {
  subnet_id                 = azurerm_subnet.pe.id
  network_security_group_id = azurerm_network_security_group.pe.id
}

resource "azurerm_subnet_network_security_group_association" "jumpbox" {
  subnet_id                 = azurerm_subnet.jumpbox.id
  network_security_group_id = azurerm_network_security_group.jumpbox.id
}
```

Things to notice:

- `cidrsubnet()` calculates the subnet ranges from the VNet range, so no hard-coded subnets are scattered around.
- **Peering needs two resources**, one in each direction. If you create only one, the status stays `Initiated` instead of `Connected`.
- The App Gateway NSG has the **GatewayManager** rule. Without it, App Gateway v2 deployment fails.

### 4.6 Private DNS zones — `dns.tf`

```hcl
locals {
  # One private DNS zone per Azure service we reach over a private endpoint,
  # plus the zone for the private AKS API server.
  private_dns_zones = {
    keyvault = "privatelink.vaultcore.azure.net"
    postgres = "privatelink.postgres.database.azure.com"
    aks      = "privatelink.${var.location}.azmk8s.io"
  }

  vnets = {
    hub   = azurerm_virtual_network.hub.id
    spoke = azurerm_virtual_network.spoke.id
  }

  # Every zone is linked to every VNet, so names resolve to private IPs from both.
  zone_links = {
    for pair in setproduct(keys(local.private_dns_zones), keys(local.vnets)) :
    "${pair[0]}-${pair[1]}" => { zone = pair[0], vnet = pair[1] }
  }
}

resource "azurerm_private_dns_zone" "zones" {
  for_each            = local.private_dns_zones
  name                = each.value
  resource_group_name = local.rg
  tags                = var.tags
}

resource "azurerm_private_dns_zone_virtual_network_link" "links" {
  for_each             = local.zone_links
  name                 = "link-${each.key}"
  private_dns_zone_id  = azurerm_private_dns_zone.zones[each.value.zone].id
  virtual_network_id   = local.vnets[each.value.vnet]
  registration_enabled = false
  tags                 = var.tags
}
```

`setproduct()` creates 3 zones × 2 VNets = **6 links** from one resource block. Each private DNS zone is a **global** resource; it is linked to VNets, not placed in one.

### 4.7 Monitoring base — `monitoring.tf`

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "log-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
  daily_quota_gb      = 1 # protects your wallet in a lab
  tags                = var.tags
}

resource "azurerm_application_insights" "appi" {
  name                = "appi-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  workspace_id        = azurerm_log_analytics_workspace.law.id
  application_type    = "java"
  tags                = var.tags
}

resource "azurerm_monitor_action_group" "email" {
  name                = "ag-${local.name}-email"
  resource_group_name = local.rg
  short_name          = "ttappmail"
  tags                = var.tags

  email_receiver {
    name                    = "me"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }
}
```

- **Log Analytics workspace**: the database for logs. Container Insights, App Insights, App Gateway and AKS diagnostic logs all go here.
- **Application Insights** (workspace-based): app telemetry (requests, dependencies, exceptions). Its data also lands in the same workspace (`AppRequests`, `AppDependencies` tables).
- **Action group**: *who* gets notified. The alerts themselves live in the platform stack.

### 4.8 Container registry — `acr.tf`

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "acr${var.prefix}${var.suffix}" # alphanumeric only, globally unique
  resource_group_name = local.rg
  location            = local.location
  sku                 = "Basic" # Premium is needed for private endpoints; see the optional step
  admin_enabled       = false   # only Entra ID / RBAC logins
  tags                = var.tags
}

# GitHub Actions (app pipeline) pushes images.
resource "azurerm_role_assignment" "github_app_acr_push" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPush"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}
```

- `admin_enabled = false`: nobody uses the shared admin username and password. Pushes use OIDC + **AcrPush**; AKS pulls with its kubelet identity + **AcrPull**.
- **Basic** SKU keeps the cost low. Private endpoints for ACR need **Premium**; see Appendix C.

### 4.9 Key Vault + private endpoint — `keyvault.tf`

```hcl
resource "azurerm_key_vault" "kv" {
  name                       = "kv-${var.prefix}-${var.suffix}"
  resource_group_name        = local.rg
  location                   = local.location
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  rbac_authorization_enabled = true # Azure RBAC instead of access policies
  soft_delete_retention_days = 7
  purge_protection_enabled   = false
  tags                       = var.tags

  # Public endpoint stays on, but only your IP is allowed.
  # Everything inside the VNets uses the private endpoint.
  public_network_access_enabled = true
  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
    ip_rules       = ["${var.my_ip}/32"]
  }
}

resource "azurerm_role_assignment" "kv_admins" {
  for_each             = toset(var.kv_admin_object_ids)
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = each.value
}

resource "azurerm_private_endpoint" "kv" {
  name                = "pe-${local.name}-kv"
  resource_group_name = local.rg
  location            = local.location
  subnet_id           = azurerm_subnet.pe.id
  tags                = var.tags

  private_service_connection {
    name                           = "psc-kv"
    private_connection_resource_id = azurerm_key_vault.kv.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  # Creates the A record in privatelink.vaultcore.azure.net automatically.
  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [azurerm_private_dns_zone.zones["keyvault"].id]
  }
}

# RBAC role assignments take a minute or two to apply.
resource "time_sleep" "wait_for_kv_rbac" {
  depends_on      = [azurerm_role_assignment.kv_admins]
  create_duration = "90s"
}

resource "azurerm_key_vault_secret" "appinsights" {
  name         = "appinsights-connection-string"
  value        = azurerm_application_insights.appi.connection_string
  key_vault_id = azurerm_key_vault.kv.id
  depends_on   = [time_sleep.wait_for_kv_rbac]
}
```

- `rbac_authorization_enabled = true` uses Azure RBAC roles instead of the old access policies. In azurerm 5.x this argument is **required**.
- **Network design**: the public endpoint is ON but the firewall allows only *your* IP, so you can manage secrets from your laptop. Everything in the VNets resolves `kv-….vault.azure.net` to the **private endpoint**.
- `time_sleep` waits for role assignments to apply. This is a real-world fix for `403 Forbidden` on the first apply.
- Soft delete is 7 days and purge protection is off. On `destroy`, the vault goes to "soft-deleted" state; we cannot purge it without subscription rights. So keep the foundation stack, or wait 7 days before reusing the name.

### 4.10 Jumpbox — `jumpbox.tf` and `cloud-init.yaml`

```hcl
# One small VM in the hub VNet. It is:
#   1. your jumpbox to reach the private AKS API, PostgreSQL and Key Vault
#   2. the GitHub self-hosted runner that deploys to the private cluster
resource "azurerm_public_ip" "jumpbox" {
  name                = "pip-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_network_interface" "jumpbox" {
  name                = "nic-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.jumpbox.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.jumpbox.id
  }
}

resource "azurerm_linux_virtual_machine" "jumpbox" {
  name                  = "vm-${local.name}-jumpbox"
  resource_group_name   = local.rg
  location              = local.location
  size                  = "Standard_B2s"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.jumpbox.id]
  custom_data           = base64encode(file("${path.module}/cloud-init.yaml"))
  tags                  = var.tags

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = 64
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "ubuntu-24_04-lts"
    sku       = "server"
    version   = "latest"
  }
}

# Stops the VM every night so you do not pay for it while sleeping.
resource "azurerm_dev_test_global_vm_shutdown_schedule" "jumpbox" {
  virtual_machine_id    = azurerm_linux_virtual_machine.jumpbox.id
  location              = local.location
  enabled               = true
  daily_recurrence_time = "2300"
  timezone              = "India Standard Time"
  tags                  = var.tags

  notification_settings {
    enabled = false
  }
}
```

```yaml
#cloud-config
package_update: true
packages:
  - ca-certificates
  - curl
  - jq
  - unzip
  - git
  - docker.io
  - postgresql-client
  - dnsutils
  - netcat-openbsd
runcmd:
  # Azure CLI, kubectl + kubelogin, Helm 4
  - curl -sL https://aka.ms/InstallAzureCLIDeb | bash
  - az aks install-cli
  - curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
  - systemctl enable --now docker
  - usermod -aG docker azureuser
  # Separate Linux user for the GitHub runner, so its "az login" never mixes with yours
  - useradd -m -s /bin/bash ghrunner
  - usermod -aG docker ghrunner
  - touch /var/log/cloud-init-tools-done
```

Change `timezone` to yours. It uses Windows time zone IDs, for example `"UTC"`, `"India Standard Time"`, `"Eastern Standard Time"`.

### 4.11 Outputs — `outputs.tf`

```hcl
# The platform stack reads these through terraform_remote_state.
output "spoke_vnet_id" { value = azurerm_virtual_network.spoke.id }
output "snet_appgw_id" { value = azurerm_subnet.appgw.id }
output "snet_aks_id" { value = azurerm_subnet.aks.id }
output "snet_pe_id" { value = azurerm_subnet.pe.id }
output "snet_appgw_cidr" { value = local.snet_appgw_cidr }
output "snet_pe_cidr" { value = local.snet_pe_cidr }

output "private_dns_zone_ids" {
  value = { for k, z in azurerm_private_dns_zone.zones : k => z.id }
}

output "log_analytics_workspace_id" { value = azurerm_log_analytics_workspace.law.id }
output "action_group_id" { value = azurerm_monitor_action_group.email.id }
output "acr_id" { value = azurerm_container_registry.acr.id }
output "acr_login_server" { value = azurerm_container_registry.acr.login_server }
output "key_vault_id" { value = azurerm_key_vault.kv.id }
output "key_vault_name" { value = azurerm_key_vault.kv.name }
output "jumpbox_public_ip" { value = azurerm_public_ip.jumpbox.ip_address }
```

The platform stack reads these with `terraform_remote_state`.

### 4.12 Values — `dev.tfvars`

```hcl
subscription_id     = "00000000-0000-0000-0000-000000000000" # echo $SUBSCRIPTION_ID
resource_group_name = "rg-your-existing-rg"                  # echo $RG
location            = "centralindia"                         # echo $LOCATION (must match your RG)
prefix              = "ttapp"
suffix              = "sk01"                           # echo $SUFFIX
ssh_public_key      = "ssh-ed25519 AAAA... you@laptop" # cat ~/.ssh/id_ed25519.pub

# my_ip and alert_email are NOT stored here. Pass them as environment variables:
#   export TF_VAR_my_ip=$(curl -s ifconfig.me)
#   export TF_VAR_alert_email="you@example.com"

# echo $MY_OBJECT_ID and echo $INFRA_PRINCIPAL_ID (Part 2)
kv_admin_object_ids = [
  "11111111-1111-1111-1111-111111111111",
  "22222222-2222-2222-2222-222222222222",
]

# echo $APP_PRINCIPAL_ID (Part 2)
github_app_principal_id = "33333333-3333-3333-3333-333333333333"
```

Fill in your values. Get them with `echo $SUBSCRIPTION_ID`, `echo $RG`, and so on, from Part 2.

> `dev.tfvars` has no secrets, so it is committed. `my_ip` and `alert_email` are passed as `TF_VAR_...` environment variables, from your shell locally and from GitHub secrets in the pipeline. That's how you keep values out of Git without a secret manager.

### 4.13 Apply the foundation (from your laptop)

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks/infra/foundation

export TF_VAR_my_ip=$(curl -s ifconfig.me)
export TF_VAR_alert_email="you@example.com"

terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan
terraform apply tfplan          # ~5-8 minutes
terraform output
```

Read the plan before you apply. Count the resources, and look for anything marked `-/+` (replace). That is what interviewers mean by "always review the plan".

### 4.14 Verify, and understand what you built

```bash
az network vnet list -g $RG -o table
az network vnet peering list -g $RG --vnet-name vnet-ttapp-dev-hub -o table     # peeringState = Connected
az network vnet subnet list -g $RG --vnet-name vnet-ttapp-dev-spoke -o table
az network nsg rule list -g $RG --nsg-name nsg-ttapp-dev-pe -o table
az network private-dns zone list -g $RG -o table
az network private-dns link vnet list -g $RG -z privatelink.vaultcore.azure.net -o table
az network private-endpoint list -g $RG -o table

# DNS from your laptop: Key Vault resolves to a PUBLIC IP (via privatelink CNAME)
nslookup $KV_NAME.vault.azure.net
```

Write down the CNAME chain from `nslookup`. You'll run the same lookup from the jumpbox in Part 5 and get **10.1.1.x**.

Also open the portal and look at the **Key Vault → Networking** blade (firewall + private endpoint connection) and the **private DNS zone → Recordsets** (the A record the endpoint created).

### 4.15 Commit

```bash
cd ~/three-tier-aks
git add infra/foundation && git commit -m "Foundation stack: network, DNS, ACR, Key Vault, monitoring, jumpbox" && git push
```

Do commit `.terraform.lock.hcl`. It pins the exact provider versions and checksums, so your laptop and the pipeline use the same provider.

(`.gitignore` comes in Part 6. Until then, make sure you are not committing `.terraform/` or `tfplan`: run `git status` before `git add`.)

---

## Part 5 — Jumpbox and GitHub self-hosted runner

### 5.1 Log in to the jumpbox

```bash
JUMPBOX_IP=$(cd ~/three-tier-aks/infra/foundation && terraform output -raw jumpbox_public_ip)
echo "export JUMPBOX_IP=$JUMPBOX_IP" >> ~/three-tier-env.sh
ssh azureuser@$JUMPBOX_IP

# on the jumpbox
cloud-init status --wait            # wait until "status: done"
ls /var/log/cloud-init-tools-done   # exists = all tools installed
az version && kubectl version --client && kubelogin --version && helm version && psql --version
```

If SSH times out, your public IP has changed. Update `TF_VAR_my_ip` and run `terraform apply` on the foundation again. The NSG rule and the Key Vault firewall both use it.

### 5.2 Prove private DNS works from inside the VNet

```bash
# on the jumpbox
az login --use-device-code
az account set --subscription <your-subscription-id>
nslookup kv-ttapp-<suffix>.vault.azure.net
# → ...privatelink.vaultcore.azure.net  Address: 10.1.1.4   (private endpoint IP!)
```

Same name, different answer. Your laptop got a public IP; the jumpbox gets the private endpoint IP because the hub VNet is **linked** to the zone. Explain this in interviews with exactly this example.

Copy your environment file to the jumpbox so the same variables work there:

```bash
# on your laptop
scp ~/three-tier-env.sh azureuser@$JUMPBOX_IP:~/
# on the jumpbox: add "source ~/three-tier-env.sh" to ~/.bashrc
echo 'source ~/three-tier-env.sh' >> ~/.bashrc
```

(`LOCATION` in that file runs `az group show`, so it works once `az login` is done on the jumpbox.)

### 5.3 Register the GitHub self-hosted runner

1. GitHub repo → **Settings → Actions → Runners → New self-hosted runner** → Linux, x64. Keep the page open; it shows a download URL and a **token** that is valid for 1 hour.
2. On the jumpbox, install the runner **as the `ghrunner` user** (created by cloud-init). That keeps the pipeline's `az login` separate from yours.

```bash
sudo -iu ghrunner
mkdir actions-runner && cd actions-runner
# paste the "Download" commands from the GitHub page (curl -o actions-runner-linux-x64-<ver>.tar.gz ... ; tar xzf ...)
./config.sh --url https://github.com/<owner>/three-tier-aks --token <TOKEN> \
  --name jumpbox-runner --labels aks-deployer --unattended
exit

# back as azureuser: install dependencies and run it as a service (starts on boot)
cd /home/ghrunner/actions-runner
sudo ./bin/installdependencies.sh
sudo ./svc.sh install ghrunner
sudo ./svc.sh start
sudo ./svc.sh status
```

The runner now shows **Idle** in GitHub with labels `self-hosted, Linux, X64, aks-deployer`. Workflows select it with `runs-on: [self-hosted, linux, aks-deployer]`.

> The VM shuts down at 23:00 every night (auto-shutdown). Start it before you run pipelines: `az vm start -g $RG -n vm-ttapp-dev-jumpbox`.

**Interview answer — Microsoft/GitHub-hosted vs self-hosted runners/agents:** hosted runners are clean, maintained by the provider, and sit on the public internet. Self-hosted runners live in *your* network (they can reach private AKS, private endpoints and on-prem), keep caches between runs, and can be bigger machines. But **you** patch them, secure them, and isolate jobs, and you must never let untrusted fork PRs run on them.

---

## Part 6 — The application code

### 6.1 Repository root files

`.gitignore`:

```text
# Build output
node_modules/
dist/
target/

# Terraform
.terraform/
*.tfstate
*.tfstate.*
tfplan
plan.txt
crash.log

# Local files
.kubeconfig
*.pfx
*.key
*.crt
.env
```

`.trivyignore` (why this exception is fine is explained in Part 7.5):

```text
# App Gateway is the public entry point: it must accept 80/443 from the internet. WAF policy protects it.
AZU-0047
```

### 6.2 Backend (Spring Boot 4, Java 21)

Generate the skeleton (this gives you `mvnw`, the folder structure and `BackendApplication.java`):

```bash
cd ~/three-tier-aks
curl -sf https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d bootVersion=4.1.2 -d javaVersion=21 \
  -d groupId=com.example -d artifactId=backend -d name=backend -d packageName=com.example.backend \
  -d dependencies=web,data-jpa,postgresql,flyway,actuator,prometheus,validation \
  -o backend.zip && unzip -o backend.zip -d backend && rm backend.zip
rm -f backend/HELP.md backend/src/main/resources/application.properties \
      backend/src/test/java/com/example/backend/BackendApplicationTests.java
```

> If start.spring.io says 4.1.2 is no longer offered, remove `-d bootVersion=4.1.2` and then set `<version>` in the pom below to the version it generated.
>
> We delete the generated `BackendApplicationTests` because it starts the whole app and needs a real database. CI has no database, so we use a web-layer test instead (6.2.8).

**Spring Boot 4 note:** starters are now modular. You'll see `spring-boot-starter-webmvc` (not `-web`), a separate `spring-boot-starter-flyway`, and matching `-test` starters. Older blog posts show the Boot 3 names.

#### 6.2.1 `backend/pom.xml` (replace the generated one; it adds JaCoCo for code coverage)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>4.1.2</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>backend</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>backend</name>
	<description/>
	<url/>
	<licenses>
		<license/>
	</licenses>
	<developers>
		<developer/>
	</developers>
	<scm>
		<connection/>
		<developerConnection/>
		<tag/>
		<url/>
	</scm>
	<properties>
		<java.version>21</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-flyway</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.flywaydb</groupId>
			<artifactId>flyway-database-postgresql</artifactId>
		</dependency>

		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.postgresql</groupId>
			<artifactId>postgresql</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-flyway-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-webmvc-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<!-- Code coverage report for SonarQube Cloud -->
			<plugin>
				<groupId>org.jacoco</groupId>
				<artifactId>jacoco-maven-plugin</artifactId>
				<version>0.8.16</version>
				<executions>
					<execution>
						<goals>
							<goal>prepare-agent</goal>
						</goals>
					</execution>
					<execution>
						<id>report</id>
						<phase>verify</phase>
						<goals>
							<goal>report</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

</project>
```

#### 6.2.2 `backend/src/main/resources/application.yml`

```yaml
spring:
  application:
    name: backend
  datasource:
    # In AKS these come from Key Vault (via the CSI driver -> Kubernetes Secret -> env vars).
    # The defaults are for running locally against docker-compose.
    url: ${DB_URL:jdbc:postgresql://localhost:5432/appdb}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate # Flyway owns the schema
  flyway:
    enabled: true
  lifecycle:
    timeout-per-shutdown-phase: 20s

server:
  port: 8080
  shutdown: graceful # finish in-flight requests when Kubernetes sends SIGTERM

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true # /actuator/health/liveness and /actuator/health/readiness
      group:
        readiness:
          include: readinessState,db # not ready if the database is unreachable
  metrics:
    tags:
      application: ${spring.application.name}
    distribution:
      percentiles-histogram:
        http.server.requests: true # enables p95/p99 latency queries in Prometheus
```

What each part gives you:

- `${DB_URL:...}`: the env var if set, otherwise the default. The same image runs locally and in AKS (12-factor config).
- `ddl-auto: validate` + **Flyway**: the schema is versioned in SQL files (`V1__...sql`). Hibernate only checks it, so it can never drop a column by accident. This is your answer to "how do you handle database changes in CI/CD and rollback?"
- `server.shutdown: graceful`: on SIGTERM, Spring stops taking new requests and finishes the in-flight ones (zero-downtime deploys).
- **Actuator probes**: `/actuator/health/liveness` (is the JVM alive?) and `/actuator/health/readiness` (can it serve traffic?). Readiness includes `db`, so if PostgreSQL is unreachable the pod becomes NotReady and gets **no traffic**, but it is **not restarted**.
- `/actuator/prometheus`: metrics for Prometheus. `percentiles-histogram` enables p95/p99 latency.

#### 6.2.3 Database migration — `backend/src/main/resources/db/migration/V1__create_todo.sql`

```sql
CREATE TABLE todo (
    id         BIGSERIAL PRIMARY KEY,
    title      VARCHAR(200) NOT NULL,
    done       BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

#### 6.2.4 `backend/src/main/java/com/example/backend/BackendApplication.java` (generated, keep it)

```java
package com.example.backend;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BackendApplication {

	public static void main(String[] args) {
		SpringApplication.run(BackendApplication.class, args);
	}

}
```

#### 6.2.5 Entity — `Todo.java`

```java
package com.example.backend;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import java.time.Instant;

@Entity
public class Todo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false)
    private boolean done;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    protected Todo() {
    }

    public Todo(String title) {
        this.title = title;
    }

    public Long getId() { return id; }
    public String getTitle() { return title; }
    public boolean isDone() { return done; }
    public Instant getCreatedAt() { return createdAt; }
    public void setDone(boolean done) { this.done = done; }
}
```

#### 6.2.6 Repository — `TodoRepository.java`

```java
package com.example.backend;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TodoRepository extends JpaRepository<Todo, Long> {
    List<Todo> findAllByOrderByIdAsc();
}
```

#### 6.2.7 REST controller — `TodoController.java`

```java
package com.example.backend;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import java.net.InetAddress;
import java.net.UnknownHostException;
import java.util.List;
import java.util.Map;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api")
public class TodoController {

    public record NewTodo(@NotBlank @Size(max = 200) String title) {
    }

    private final TodoRepository repository;

    public TodoController(TodoRepository repository) {
        this.repository = repository;
    }

    @GetMapping("/todos")
    public List<Todo> list() {
        return repository.findAllByOrderByIdAsc();
    }

    @PostMapping("/todos")
    @ResponseStatus(HttpStatus.CREATED)
    public Todo create(@Valid @RequestBody NewTodo body) {
        return repository.save(new Todo(body.title()));
    }

    @PutMapping("/todos/{id}/toggle")
    public Todo toggle(@PathVariable Long id) {
        Todo todo = repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        todo.setDone(!todo.isDone());
        return repository.save(todo);
    }

    @DeleteMapping("/todos/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        repository.deleteById(id);
    }

    // Shows which pod answered: useful to see load balancing across replicas.
    @GetMapping("/info")
    public Map<String, String> info() throws UnknownHostException {
        return Map.of("pod", InetAddress.getLocalHost().getHostName(), "version",
                System.getenv().getOrDefault("APP_VERSION", "local"));
    }
}
```

#### 6.2.8 Unit test — `backend/src/test/java/com/example/backend/TodoControllerTest.java`

```java
package com.example.backend;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import java.util.List;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

// Tests only the web layer, so CI does not need a database.
@WebMvcTest(TodoController.class)
class TodoControllerTest {

    @Autowired
    private MockMvc mvc;

    @MockitoBean
    private TodoRepository repository;

    @Test
    void listsTodos() throws Exception {
        when(repository.findAllByOrderByIdAsc()).thenReturn(List.of(new Todo("learn AKS")));

        mvc.perform(get("/api/todos"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].title").value("learn AKS"));
    }

    @Test
    void createsTodo() throws Exception {
        when(repository.save(any(Todo.class))).thenAnswer(inv -> inv.getArgument(0));

        mvc.perform(post("/api/todos").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"write helm chart\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.title").value("write helm chart"));
    }

    @Test
    void rejectsBlankTitle() throws Exception {
        mvc.perform(post("/api/todos").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"\"}"))
                .andExpect(status().isBadRequest());
    }
}
```

`@WebMvcTest` starts only the web layer, and `@MockitoBean` replaces the repository. These tests need no database, so they run in seconds in CI. (`@MockBean` was removed in Boot 4; `@MockitoBean` replaces it.)

Run the tests without installing Java:

```bash
cd ~/three-tier-aks/backend
docker run --rm -v "$PWD":/w -v "$HOME/.m2":/root/.m2 -w /w maven:3.9-eclipse-temurin-21 mvn -B verify
# BUILD SUCCESS, Tests run: 3, and target/site/jacoco/jacoco.xml exists
```

### 6.3 Frontend (React + Vite, served by nginx)

The frontend is small on purpose. The lab is about the platform, not React.

`frontend/package.json`:

```json
{
  "name": "frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "19.3.1",
    "react-dom": "19.3.1"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "6.1.2",
    "vite": "8.3.5"
  }
}
```

`frontend/vite.config.js`:

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    // Local development only: forward /api to the Spring Boot app.
    proxy: { '/api': 'http://localhost:8080' },
  },
})
```

`frontend/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Three-tier Todo</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

`frontend/src/main.jsx`:

```jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

`frontend/src/App.jsx`:

```jsx
import { useEffect, useState } from 'react'

// The browser only ever calls /api on the same host.
// nginx (in the frontend pod) forwards /api to the backend Service inside the cluster.
export default function App() {
  const [todos, setTodos] = useState([])
  const [title, setTitle] = useState('')
  const [info, setInfo] = useState(null)
  const [error, setError] = useState('')

  async function load() {
    try {
      const [todosRes, infoRes] = await Promise.all([fetch('/api/todos'), fetch('/api/info')])
      if (!todosRes.ok) throw new Error(`GET /api/todos -> ${todosRes.status}`)
      setTodos(await todosRes.json())
      setInfo(await infoRes.json())
      setError('')
    } catch (e) {
      setError(e.message)
    }
  }

  useEffect(() => {
    load()
  }, [])

  async function add(e) {
    e.preventDefault()
    if (!title.trim()) return
    await fetch('/api/todos', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title }),
    })
    setTitle('')
    load()
  }

  async function toggle(id) {
    await fetch(`/api/todos/${id}/toggle`, { method: 'PUT' })
    load()
  }

  async function remove(id) {
    await fetch(`/api/todos/${id}`, { method: 'DELETE' })
    load()
  }

  return (
    <main style={{ fontFamily: 'sans-serif', maxWidth: 560, margin: '40px auto', padding: '0 16px' }}>
      <h1>Three-tier Todo</h1>
      <p>React (nginx) → Spring Boot → PostgreSQL on private AKS</p>
      {info && <p><small>Answered by pod <b>{info.pod}</b>, version <b>{info.version}</b></small></p>}
      {error && <p style={{ color: 'crimson' }}>Error: {error}</p>}

      <form onSubmit={add} style={{ display: 'flex', gap: 8 }}>
        <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="New todo" style={{ flex: 1 }} />
        <button type="submit">Add</button>
      </form>

      <ul>
        {todos.map((t) => (
          <li key={t.id}>
            <label style={{ textDecoration: t.done ? 'line-through' : 'none' }}>
              <input type="checkbox" checked={t.done} onChange={() => toggle(t.id)} /> {t.title}
            </label>{' '}
            <button onClick={() => remove(t.id)}>x</button>
          </li>
        ))}
      </ul>
    </main>
  )
}
```

`frontend/nginx/default.conf.template`:

```nginx
# ${BACKEND_URL} is replaced at container start by the nginx image's envsubst step.
server {
    listen 8080;
    root /usr/share/nginx/html;

    # Used by Kubernetes probes AND by Application Gateway's health probe.
    location = /healthz {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    # Reverse proxy: browser -> /api/... -> backend Service (ClusterIP) inside the cluster.
    location /api/ {
        proxy_pass ${BACKEND_URL};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # React single-page app: unknown paths fall back to index.html.
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**How the frontend connects to the backend (an interview question).** The browser only calls `/api/...` on the **same host**, so there is no CORS and no backend URL baked into the JavaScript. nginx in the frontend pod forwards `/api/` to the backend **ClusterIP Service** (`BACKEND_URL`), which is never exposed to the internet. The nginx image runs `envsubst` on `/etc/nginx/templates/*.template` at start-up and replaces `${BACKEND_URL}` with the environment variable. The same image works locally (docker-compose) and in AKS (Helm sets the variable).

Create the lock file (CI uses `npm ci`, which **requires** `package-lock.json`):

```bash
cd ~/three-tier-aks/frontend
npm install            # or: docker run --rm -v "$PWD":/w -w /w node:24-alpine npm install
npm run build          # creates dist/
```

Commit `package-lock.json`. It pins every transitive dependency, so builds are reproducible and Trivy can scan exact versions.

### 6.4 Run all three tiers locally

docker-compose builds the images with the Dockerfiles from Part 7. **Create `backend/Dockerfile`, `backend/.dockerignore`, `frontend/Dockerfile` and `frontend/.dockerignore` from [7.1](#71-backend--backenddockerfile) and [7.2](#72-frontend--frontenddockerfile) first**, then come back here.

`docker-compose.yml` at the repo root:

```yaml
# Local run of all three tiers: http://localhost:3000
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d appdb"]
      interval: 5s
      retries: 10

  backend:
    build: ./backend
    environment:
      DB_URL: jdbc:postgresql://db:5432/appdb
      DB_USERNAME: postgres
      DB_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: ./frontend
    environment:
      BACKEND_URL: http://backend:8080
    ports:
      - "3000:8080"
    depends_on:
      - backend
```

```bash
cd ~/three-tier-aks
docker compose up -d --build
curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"first todo"}' localhost:3000/api/todos
curl -s localhost:3000/api/todos
curl -s localhost:3000/api/info
# open http://localhost:3000 in the browser
docker compose logs backend | grep -i flyway   # "Successfully applied 1 migration"
docker compose down -v
```

When this works locally, you know any problem in AKS is about **the platform** (network, identity, config), not the code. Say this in interviews: "first reproduce locally, then isolate the layer."

```bash
git add . && git commit -m "Three-tier app: React, Spring Boot, Flyway, docker-compose" && git push
```

---

## Part 7 — Multistage Dockerfiles and local scans

### 7.1 Backend — `backend/Dockerfile`

```dockerfile
# ---------- Stage 1: build the JAR ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src

# Copy only pom.xml first: dependencies are cached until pom.xml changes.
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

COPY src ./src
RUN mvn -B -q package -DskipTests

# Azure Monitor Application Insights Java agent (auto-collects requests, SQL calls, exceptions).
# --chmod matters: a remote ADD is saved as mode 600 (root only), so the non-root user could not read it.
ADD --chmod=644 https://github.com/microsoft/ApplicationInsights-Java/releases/download/3.7.10/applicationinsights-agent-3.7.10.jar /src/applicationinsights-agent.jar

# ---------- Stage 2: small runtime image ----------
# Distroless: no shell, no package manager, runs as non-root (uid 65532).
FROM gcr.io/distroless/java21-debian13:nonroot
WORKDIR /app
COPY --from=build /src/target/backend-0.0.1-SNAPSHOT.jar /app/app.jar
COPY --from=build /src/applicationinsights-agent.jar /app/applicationinsights-agent.jar

EXPOSE 8080
USER nonroot
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

`backend/.dockerignore`:

```text
target/
.mvn/
mvnw
mvnw.cmd
.git
*.md
```

### 7.2 Frontend — `frontend/Dockerfile`

```dockerfile
# ---------- Stage 1: build static files ----------
FROM node:24-alpine AS build
WORKDIR /app

# Install dependencies first so this layer is cached until package*.json changes.
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY . .
RUN npm run build

# ---------- Stage 2: serve with nginx as non-root ----------
# nginx-unprivileged runs as uid 101 and listens on 8080 (not 80).
FROM nginxinc/nginx-unprivileged:1.31-alpine

# Patch OS packages that have published fixes (keeps Trivy green between base-image releases).
USER root
RUN apk upgrade --no-cache
USER 101

COPY nginx/default.conf.template /etc/nginx/templates/default.conf.template
COPY --from=build /app/dist /usr/share/nginx/html

ENV BACKEND_URL=http://backend:8080
EXPOSE 8080
```

`frontend/.dockerignore`:

```text
node_modules
dist
.git
*.md
```

### 7.3 What to say about these Dockerfiles

| Technique | Where | Why |
|---|---|---|
| **Multistage build** | both | Maven/JDK and Node exist only in the build stage. The final image has only the JRE + JAR, or nginx + static files. Measured: backend final image ≈ 300 MB (the Maven + JDK build image alone is ≈ 540 MB, before dependencies); frontend ≈ 65 MB (the Node build image alone is ≈ 170 MB, plus `node_modules`) |
| **Layer caching** | `COPY pom.xml` / `package*.json` first | dependencies download again only when those files change, so code changes rebuild in seconds. Your answer to "how do you reduce Docker build time?" |
| **Non-root** | `distroless:nonroot`, `nginx-unprivileged` | a container escape lands as an unprivileged user; required by `runAsNonRoot: true` in Kubernetes |
| **Distroless** | backend runtime | no shell, no package manager, so a much smaller attack surface and fewer CVEs |
| **`.dockerignore`** | both | smaller build context, no `target/` or `node_modules/` leaking in, faster builds |
| **`ENTRYPOINT` vs `CMD`** | backend | `ENTRYPOINT` is the fixed executable; `CMD` gives default arguments that `docker run image <args>` can override. JVM options come from `JAVA_TOOL_OPTIONS`, so the entrypoint never changes |
| **`ADD --chmod`** | backend | a real bug found while testing: a remote `ADD` is saved as mode 600 (root only), so the non-root JVM failed with "Error opening zip file". `--chmod=644` fixes it |
| **`apk upgrade`** | frontend | patches OS packages that have fixes not yet in the base image |

### 7.4 Build and run the images locally

```bash
cd ~/three-tier-aks
docker build -t ttapp-backend:local backend
docker build -t ttapp-frontend:local frontend
docker images | grep ttapp

# Run the backend like Kubernetes will: read-only root filesystem, writable /tmp only
docker network create ttnet
docker run -d --name pg --network ttnet -e POSTGRES_DB=appdb -e POSTGRES_PASSWORD=postgres postgres:16-alpine
docker run -d --name be --network ttnet --read-only --tmpfs /tmp -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://pg:5432/appdb ttapp-backend:local
sleep 25 && curl -s localhost:8080/actuator/health/readiness     # {"status":"UP"}

docker exec be sh     # fails: "executable file not found" -> distroless has no shell (on purpose)
docker rm -f be pg && docker network rm ttnet
```

How to debug a distroless container is a good interview question. In Kubernetes you use `kubectl debug` to attach an **ephemeral debug container** that shares the process namespace (Part 10.5).

### 7.5 Scan locally with Trivy and Gitleaks

You don't need to install these tools; run them as containers:

```bash
cd ~/three-tier-aks
TRIVY=aquasec/trivy:0.76.1

# 1) Image scan: OS packages + Java/npm libraries inside the image
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY \
  image --severity HIGH,CRITICAL --ignore-unfixed ttapp-backend:local
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY \
  image --severity HIGH,CRITICAL --ignore-unfixed ttapp-frontend:local

# 2) Filesystem scan: pom.xml and package-lock.json dependencies + hard-coded secrets
docker run --rm -v "$PWD":/src -v "$HOME/.m2":/root/.m2 $TRIVY \
  fs --offline-scan --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed \
  --skip-dirs /src/frontend/node_modules --skip-dirs /src/backend/target /src

# 3) Misconfiguration scan: Terraform (Helm is added in Part 9). One target per run.
docker run --rm -v "$PWD":/src -w /src $TRIVY config --severity HIGH,CRITICAL \
  --ignorefile /src/.trivyignore /src/infra

# 4) Secrets in the whole Git history
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git /repo --redact
```

**Real findings from testing this guide (use them as interview stories):**

1. **Base images had fixable HIGH CVEs.** `distroless/java21-debian12` and `nginx-unprivileged:1.29-alpine` failed the gate (expat, brotli, curl…). The fix was moving to `java21-debian13` and `1.31-alpine`, plus `apk upgrade`. Then the scan was clean. *"How do you fix Trivy findings?"*: update the base image first, then the dependency; use `--ignore-unfixed` so you only block on things you *can* fix; record a justified exception in `.trivyignore` only when the risk is accepted.
2. **Maven Central returned HTTP 429** when Trivy looked up the parent POM. The fix is to run Trivy *after* the Maven build, mount `~/.m2`, and use `--offline-scan`.
3. **Trivy config flagged AZU-0047** ("NSG allows ingress from any IP") on the App Gateway subnet. That rule is the whole point of a public web entry point, and WAF protects it, so it is suppressed in `.trivyignore` **with a comment explaining why**. Never suppress silently.
4. **Trivy config flagged KSV-0014** (frontend without a read-only root filesystem). It was fixed in the chart (Part 9), not ignored.

Test that Gitleaks really works by committing a fake token on a throwaway branch:

```bash
git checkout -b leak-test
echo "token=ghp_$(head -c 200 /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 36)" > leak.txt
git add leak.txt && git commit -m "test leak"
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git /repo --redact
# "leaks found: 1", exit code 1
git checkout main && git branch -D leak-test     # never push that branch
```

```bash
git add . && git commit -m "Multistage Dockerfiles" && git push
```

---

## Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL)

Create these files in `infra/platform/`.

### 8.1 `versions.tf`, `backend.hcl`, `variables.tf`, `main.tf`

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.11"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.13"
    }
  }

  backend "azurerm" {}
}

provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "none"
  storage_use_azuread             = true
}
```

```hcl
resource_group_name  = "rg-your-existing-rg" # echo $RG
storage_account_name = "sttfstatesk01"       # echo $TFSTATE_SA
container_name       = "tfstate"
key                  = "platform.tfstate"
use_azuread_auth     = true
```

```hcl
variable "subscription_id" { type = string }
variable "resource_group_name" { type = string }
variable "location" { type = string }

variable "prefix" {
  type    = string
  default = "ttapp"
}

variable "suffix" { type = string }

variable "tfstate_storage_account" {
  type        = string
  description = "Storage account that holds the Terraform state (from Part 2)."
}

variable "kubernetes_version" {
  type        = string
  description = "Pick one version BELOW the latest from 'az aks get-versions' so you can practise an upgrade."
}

variable "system_vm_size" {
  type    = string
  default = "Standard_D2s_v5"
}

variable "user_vm_size" {
  type    = string
  default = "Standard_D2s_v5"
}

variable "aks_admin_object_ids" {
  type        = list(string)
  description = "Users who get 'Azure Kubernetes Service RBAC Cluster Admin' (you)."
}

variable "github_app_principal_id" {
  type        = string
  description = "Principal ID of id-gh-app. Gets deploy rights on the 'app' namespace."
}

variable "app_namespace" {
  type    = string
  default = "app"
}

variable "backend_service_account" {
  type    = string
  default = "backend"
}

variable "tags" {
  type = map(string)
  default = {
    project     = "three-tier-aks"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

```hcl
data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

# Read the outputs of the foundation stack from its state file.
data "terraform_remote_state" "foundation" {
  backend = "azurerm"
  config = {
    resource_group_name  = var.resource_group_name
    storage_account_name = var.tfstate_storage_account
    container_name       = "tfstate"
    key                  = "foundation.tfstate"
    use_azuread_auth     = true
  }
}

locals {
  rg       = data.azurerm_resource_group.rg.name
  location = var.location
  name     = "${var.prefix}-dev"
  f        = data.terraform_remote_state.foundation.outputs
}
```

`terraform_remote_state` reads the **outputs** of the foundation state file. The platform stack never changes foundation resources; it only references their IDs. (A common alternative is a `data` source for each resource by name.)

### 8.2 Application Gateway + WAF — `appgw.tf`

```hcl
resource "azurerm_public_ip" "appgw" {
  name                = "pip-${local.name}-appgw"
  resource_group_name = local.rg
  location            = local.location
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_web_application_firewall_policy" "waf" {
  name                = "waf-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  policy_settings {
    enabled                     = true
    mode                        = "Prevention" # "Detection" only logs
    request_body_check          = true
    max_request_body_size_in_kb = 128
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }
  }

  # A custom rule you can test with: curl -A "BadBot" http://<appgw-ip>/
  custom_rules {
    name      = "BlockBadBot"
    priority  = 1
    rule_type = "MatchRule"
    action    = "Block"

    match_conditions {
      match_variables {
        variable_name = "RequestHeaders"
        selector      = "User-Agent"
      }
      operator           = "Contains"
      negation_condition = false
      match_values       = ["BadBot"]
      transforms         = ["Lowercase"]
    }
  }
}

locals {
  # AGIC overwrites these names; we only need a valid starting configuration.
  appgw_fe_ip    = "appgw-fe-public"
  appgw_fe_port  = "port-80"
  appgw_pool     = "default-pool"
  appgw_settings = "default-settings"
  appgw_listener = "default-listener"
  appgw_rule     = "default-rule"
}

resource "azurerm_application_gateway" "appgw" {
  name                = "agw-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  firewall_policy_id  = azurerm_web_application_firewall_policy.waf.id
  tags                = var.tags

  sku {
    name = "WAF_v2"
    tier = "WAF_v2"
  }

  autoscale_configuration {
    min_capacity = 0
    max_capacity = 2
  }

  gateway_ip_configuration {
    name      = "appgw-ipconfig"
    subnet_id = local.f.snet_appgw_id
  }

  frontend_ip_configuration {
    name                 = local.appgw_fe_ip
    public_ip_address_id = azurerm_public_ip.appgw.id
  }

  frontend_port {
    name = local.appgw_fe_port
    port = 80
  }

  backend_address_pool {
    name = local.appgw_pool
  }

  backend_http_settings {
    name                  = local.appgw_settings
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 30
  }

  http_listener {
    name                           = local.appgw_listener
    frontend_ip_configuration_name = local.appgw_fe_ip
    frontend_port_name             = local.appgw_fe_port
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = local.appgw_rule
    rule_type                  = "Basic"
    http_listener_name         = local.appgw_listener
    backend_address_pool_name  = local.appgw_pool
    backend_http_settings_name = local.appgw_settings
    priority                   = 19000
  }

  # AGIC owns these blocks after the first deploy.
  # Without ignore_changes every "terraform apply" would wipe the AGIC config and cause an outage.
  lifecycle {
    ignore_changes = [
      backend_address_pool,
      backend_http_settings,
      frontend_port,
      http_listener,
      probe,
      redirect_configuration,
      request_routing_rule,
      ssl_certificate,
      url_path_map,
      rewrite_rule_set,
      tags,
    ]
  }
}
```

Two things are important here:

1. **App Gateway cannot be created empty.** Terraform gives it one dummy listener, rule and pool. After that, **AGIC owns the configuration**: it adds listeners, rules, pools and probes from your Ingress objects.
2. **`ignore_changes`.** Without it, the next `terraform apply` would "fix" the drift by deleting everything AGIC created, and the site would go down. This is a known AGIC + Terraform problem and a good interview story.

`autoscale_configuration` with `min_capacity = 0` keeps the cost at the fixed hourly price only. Production would use `min_capacity >= 2` for zone redundancy.

### 8.3 AKS — `aks.tf`

```hcl
# ---------- Control-plane identity ----------
# A custom private DNS zone needs a user-assigned identity that we can grant rights to
# BEFORE the cluster is created.
resource "azurerm_user_assigned_identity" "aks" {
  name                = "id-${local.name}-aks"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags
}

resource "azurerm_role_assignment" "aks_dns" {
  scope                = local.f.private_dns_zone_ids["aks"]
  role_definition_name = "Private DNS Zone Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "aks_network" {
  scope                = local.f.spoke_vnet_id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
  principal_type       = "ServicePrincipal"
}

resource "time_sleep" "wait_for_aks_rbac" {
  depends_on      = [azurerm_role_assignment.aks_dns, azurerm_role_assignment.aks_network]
  create_duration = "60s"
}

# ---------- Cluster ----------
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  kubernetes_version  = var.kubernetes_version
  sku_tier            = "Free"
  node_resource_group = "${local.rg}-aks-nodes" # AKS creates this RG itself
  tags                = var.tags

  # --- Private API server ---
  private_cluster_enabled             = true
  private_dns_zone_id                 = local.f.private_dns_zone_ids["aks"]
  private_cluster_public_fqdn_enabled = false
  dns_prefix_private_cluster          = local.name

  # --- Identity and access ---
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }

  local_account_disabled            = true # no static admin kubeconfig
  role_based_access_control_enabled = true
  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true # Kubernetes permissions come from Azure role assignments
    tenant_id          = data.azurerm_client_config.current.tenant_id
  }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  # --- Nodes ---
  node_provisioning_profile {
    mode = "Manual" # classic node pools (not Node Auto Provisioning)
  }

  default_node_pool {
    name                         = "system"
    vm_size                      = var.system_vm_size
    orchestrator_version         = var.kubernetes_version
    node_count                   = 1
    vnet_subnet_id               = local.f.snet_aks_id
    max_pods                     = 30
    only_critical_addons_enabled = true # taint: CriticalAddonsOnly=true:NoSchedule
    temporary_name_for_rotation  = "systemtmp"

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # --- Networking ---
  network_profile {
    network_plugin    = "azure"  # Azure CNI: pods get real VNet IPs (needed for AGIC)
    network_policy    = "calico" # enforces Kubernetes NetworkPolicy objects
    service_cidr      = "10.100.0.0/16"
    dns_service_ip    = "10.100.0.10"
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
  }

  # --- Add-ons ---
  ingress_application_gateway {
    gateway_id = azurerm_application_gateway.appgw.id
  }

  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  oms_agent {
    log_analytics_workspace_id      = local.f.log_analytics_workspace_id
    msi_auth_for_monitoring_enabled = true
  }

  workload_autoscaler_profile {
    vertical_pod_autoscaler_enabled = true
  }

  node_os_upgrade_channel = "NodeImage"

  depends_on = [time_sleep.wait_for_aks_rbac]
}

resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  mode                  = "User"
  vm_size               = var.user_vm_size
  orchestrator_version  = var.kubernetes_version
  vnet_subnet_id        = local.f.snet_aks_id
  max_pods              = 30
  auto_scaling_enabled  = true # Cluster Autoscaler
  min_count             = 1
  max_count             = 3
  node_labels           = { "workload" = "apps" }
  tags                  = var.tags

  upgrade_settings {
    max_surge = "33%"
  }

  lifecycle {
    ignore_changes = [node_count] # the autoscaler changes it
  }
}

# ---------- Role assignments the cluster needs ----------
# Kubelet pulls images from ACR (replaces "az aks update --attach-acr").
resource "azurerm_role_assignment" "kubelet_acr_pull" {
  scope                = local.f.acr_id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  principal_type       = "ServicePrincipal"
}

# AGIC add-on identity: manage the gateway, read the RG, join the gateway subnet.
locals {
  agic_principal_id = azurerm_kubernetes_cluster.aks.ingress_application_gateway[0].ingress_application_gateway_identity[0].object_id
}

resource "azurerm_role_assignment" "agic_appgw" {
  scope                = azurerm_application_gateway.appgw.id
  role_definition_name = "Contributor"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "agic_rg_reader" {
  scope                = data.azurerm_resource_group.rg.id
  role_definition_name = "Reader"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "agic_subnet" {
  scope                = local.f.snet_appgw_id
  role_definition_name = "Network Contributor"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

# You: full Kubernetes admin through Entra ID.
resource "azurerm_role_assignment" "aks_admins" {
  for_each             = toset(var.aks_admin_object_ids)
  scope                = azurerm_kubernetes_cluster.aks.id
  role_definition_name = "Azure Kubernetes Service RBAC Cluster Admin"
  principal_id         = each.value
}

# GitHub app pipeline: may fetch a kubeconfig, and is admin ONLY inside the app namespace.
resource "azurerm_role_assignment" "github_cluster_user" {
  scope                = azurerm_kubernetes_cluster.aks.id
  role_definition_name = "Azure Kubernetes Service Cluster User Role"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "github_namespace_admin" {
  scope                = "${azurerm_kubernetes_cluster.aks.id}/namespaces/${var.app_namespace}"
  role_definition_name = "Azure Kubernetes Service RBAC Admin"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}

# ---------- Container Insights data collection rule ----------
resource "azurerm_monitor_data_collection_rule" "ci" {
  name                = "MSCI-${local.location}-aks-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  destinations {
    log_analytics {
      name                  = "ciworkspace"
      workspace_resource_id = local.f.log_analytics_workspace_id
    }
  }

  data_flow {
    streams      = ["Microsoft-ContainerInsights-Group-Default"]
    destinations = ["ciworkspace"]
  }

  data_sources {
    extension {
      name           = "ContainerInsightsExtension"
      extension_name = "ContainerInsights"
      streams        = ["Microsoft-ContainerInsights-Group-Default"]
      extension_json = jsonencode({
        dataCollectionSettings = {
          interval               = "1m"
          namespaceFilteringMode = "Off"
          enableContainerLogV2   = true
        }
      })
    }
  }
}

resource "azurerm_monitor_data_collection_rule_association" "ci" {
  name                    = "ContainerInsightsExtension"
  target_resource_id      = azurerm_kubernetes_cluster.aks.id
  data_collection_rule_id = azurerm_monitor_data_collection_rule.ci.id
}
```

Walk through it like you would in an interview:

| Setting | What it does | Interview question it answers |
|---|---|---|
| `private_cluster_enabled` + custom `private_dns_zone_id` | API server has only a private IP, name in our zone | "Have you created a private AKS? How?" |
| user-assigned control-plane identity + 2 role assignments **before** the cluster | AKS needs rights on the DNS zone and VNet during creation | "service principal vs managed identity", "system vs user-assigned" |
| `local_account_disabled` + `azure_rbac_enabled` | no static admin kubeconfig; every `kubectl` call is an Entra ID identity checked against Azure roles | "How do you authenticate to AKS?", "local accounts in Kubernetes" |
| `oidc_issuer_enabled` + `workload_identity_enabled` | pods can exchange their service-account token for an Entra token | "How does AKS connect to Key Vault?" |
| system pool with `only_critical_addons_enabled` | taint `CriticalAddonsOnly=true:NoSchedule`, so only system pods run there | taints/tolerations, node pools |
| user pool `auto_scaling_enabled`, 1–3 nodes | **Cluster Autoscaler** adds nodes when pods are Pending | "How do you increase nodes automatically?" |
| `network_plugin = "azure"` | Azure CNI: pods get VNet IPs. AGIC **needs** routable pod IPs | "Which network plugin do you use? Which component gives a pod its IP?" (the CNI plugin, called by kubelet) |
| `network_policy = "calico"` | enforces NetworkPolicy objects | "What are network plugins used for?" |
| `ingress_application_gateway` | AGIC add-on, managed by AKS | Ingress controller |
| `key_vault_secrets_provider` | Secrets Store CSI driver + Azure provider, rotation every 2 min | CSI, secrets in pods |
| `oms_agent` + DCR | Container Insights (logs + inventory to Log Analytics) | "How do AKS logs reach Azure Monitor?" |
| `vertical_pod_autoscaler_enabled` | VPA add-on (used in recommendation mode) | HPA vs VPA vs CA |
| `max_surge = "33%"` | during an upgrade, add 1 extra node before draining an old one | zero-downtime upgrades |
| kubelet identity → **AcrPull** | nodes pull images from ACR without passwords | "ImagePullBackOff 401" |
| AGIC identity → Contributor on App Gateway, Reader on RG, Network Contributor on its subnet | AGIC must update App Gateway through ARM | AGIC troubleshooting |
| GitHub app identity → Cluster User + RBAC Admin **on namespace `app`** | the pipeline can deploy into `app` and nowhere else | least privilege for CI/CD |

> The azurerm docs note that AGIC is a *non-critical* add-on. With `only_critical_addons_enabled` on the system pool, the AGIC pod needs a **user** node pool to run on. Our `user` pool is required, not optional.

### 8.4 PostgreSQL + private endpoint — `postgres.tf`

```hcl
resource "random_password" "pg" {
  length           = 24
  special          = true
  override_special = "!#%*-_=+"
}

resource "azurerm_postgresql_flexible_server" "pg" {
  name                          = "psql-${var.prefix}-${var.suffix}"
  resource_group_name           = local.rg
  location                      = local.location
  version                       = "16"
  sku_name                      = "B_Standard_B1ms" # cheapest burstable tier
  storage_mb                    = 32768
  backup_retention_days         = 7
  administrator_login           = "pgadmin"
  administrator_password        = random_password.pg.result
  public_network_access_enabled = false # only reachable through the private endpoint
  tags                          = var.tags

  lifecycle {
    ignore_changes = [zone] # Azure picks a zone; do not fight it
  }
}

resource "azurerm_postgresql_flexible_server_database" "app" {
  name      = "appdb"
  server_id = azurerm_postgresql_flexible_server.pg.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

resource "azurerm_private_endpoint" "pg" {
  name                = "pe-${local.name}-psql"
  resource_group_name = local.rg
  location            = local.location
  subnet_id           = local.f.snet_pe_id
  tags                = var.tags

  private_service_connection {
    name                           = "psc-psql"
    private_connection_resource_id = azurerm_postgresql_flexible_server.pg.id
    subresource_names              = ["postgresqlServer"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [local.f.private_dns_zone_ids["postgres"]]
  }
}

# The backend reads these from Key Vault through the CSI driver.
locals {
  db_secrets = {
    "db-url"      = "jdbc:postgresql://${azurerm_postgresql_flexible_server.pg.fqdn}:5432/${azurerm_postgresql_flexible_server_database.app.name}?sslmode=require"
    "db-username" = azurerm_postgresql_flexible_server.pg.administrator_login
    "db-password" = random_password.pg.result
  }
}

resource "azurerm_key_vault_secret" "db" {
  for_each     = local.db_secrets
  name         = each.key
  value        = each.value
  key_vault_id = local.f.key_vault_id
}
```

- `public_network_access_enabled = false` + a **private endpoint**: the server has no public reachability at all.
- The private endpoint's `private_dns_zone_group` writes the A record into `privatelink.postgres.database.azure.com`.
- The password is generated by Terraform and written straight into Key Vault. You never type it. It is in the Terraform **state**, which is why the state account is locked down (Entra-only, no keys, versioned).
- The other model is **VNet integration** (a delegated subnet). Know the difference: VNet integration puts the server *inside* a subnet you dedicate to it; a private endpoint puts only a NIC in your subnet, and the server stays a PaaS service.

### 8.5 Workload Identity for the backend — `workload-identity.tf`

```hcl
# Identity used by the backend pods to read Key Vault (no passwords stored anywhere).
resource "azurerm_user_assigned_identity" "backend" {
  name                = "id-${local.name}-backend"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags
}

# Trust: "a token from THIS cluster for THIS service account may act as this identity".
resource "azurerm_federated_identity_credential" "backend" {
  name                      = "fic-aks-backend"
  user_assigned_identity_id = azurerm_user_assigned_identity.backend.id
  audience                  = ["api://AzureADTokenExchange"]
  issuer                    = azurerm_kubernetes_cluster.aks.oidc_issuer_url
  subject                   = "system:serviceaccount:${var.app_namespace}:${var.backend_service_account}"
}

resource "azurerm_role_assignment" "backend_kv" {
  scope                = local.f.key_vault_id
  role_definition_name = "Key Vault Secrets User" # read-only
  principal_id         = azurerm_user_assigned_identity.backend.principal_id
  principal_type       = "ServicePrincipal"
}
```

The federated credential says: *"Tokens issued by **this AKS cluster's OIDC issuer** for the service account **`app/backend`** may act as this managed identity."* No secret is involved. If someone deploys a pod with a different service account or namespace, it gets nothing.

### 8.6 Diagnostics — `diagnostics.tf`

```hcl
# Send logs to Log Analytics so you can query them with KQL.
# "Dedicated" = one table per log type (AGWFirewallLogs, AKSAuditAdmin, ...) instead of AzureDiagnostics.
resource "azurerm_monitor_diagnostic_setting" "appgw" {
  name                           = "diag-appgw"
  target_resource_id             = azurerm_application_gateway.appgw.id
  log_analytics_workspace_id     = local.f.log_analytics_workspace_id
  log_analytics_destination_type = "Dedicated"

  enabled_log {
    category_group = "allLogs" # access log, performance log, firewall (WAF) log
  }

  enabled_metric {
    category = "AllMetrics"
  }
}

resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                           = "diag-aks"
  target_resource_id             = azurerm_kubernetes_cluster.aks.id
  log_analytics_workspace_id     = local.f.log_analytics_workspace_id
  log_analytics_destination_type = "Dedicated"

  # Only the useful, low-volume categories (the 1 GB/day cap is on).
  enabled_log {
    category = "kube-audit-admin" # who created/changed/deleted what (no read calls)
  }

  enabled_log {
    category = "cluster-autoscaler"
  }

  enabled_log {
    category = "guard" # Entra ID authentication decisions
  }
}
```

### 8.7 Alerts — `alerts.tf`

```hcl
# Metric alert: average node CPU above 80% for 15 minutes.
resource "azurerm_monitor_metric_alert" "node_cpu" {
  name                = "alert-${local.name}-node-cpu"
  resource_group_name = local.rg
  scopes              = [azurerm_kubernetes_cluster.aks.id]
  description         = "AKS node CPU above 80%"
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"
  tags                = var.tags

  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "node_cpu_usage_percentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = local.f.action_group_id
  }
}

# Log alert: any container in CrashLoopBackOff in the last 10 minutes.
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "crashloop" {
  name                 = "alert-${local.name}-crashloop"
  resource_group_name  = local.rg
  location             = local.location
  scopes               = [local.f.log_analytics_workspace_id]
  description          = "Pods in CrashLoopBackOff"
  severity             = 2
  evaluation_frequency = "PT5M"
  window_duration      = "PT10M"
  tags                 = var.tags

  criteria {
    query                   = <<-KQL
      KubePodInventory
      | where ContainerStatusReason == "CrashLoopBackOff"
      | summarize Restarts = count() by Namespace, Name
    KQL
    time_aggregation_method = "Count"
    operator                = "GreaterThan"
    threshold               = 0

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  action {
    action_groups = [local.f.action_group_id]
  }
}
```

### 8.8 Outputs and values

```hcl
output "aks_name" { value = azurerm_kubernetes_cluster.aks.name }
output "aks_private_fqdn" { value = azurerm_kubernetes_cluster.aks.private_fqdn }
output "appgw_public_ip" { value = azurerm_public_ip.appgw.ip_address }
output "postgres_fqdn" { value = azurerm_postgresql_flexible_server.pg.fqdn }
output "backend_identity_client_id" { value = azurerm_user_assigned_identity.backend.client_id }
output "tenant_id" { value = data.azurerm_client_config.current.tenant_id }
```

```hcl
subscription_id         = "00000000-0000-0000-0000-000000000000" # echo $SUBSCRIPTION_ID
resource_group_name     = "rg-your-existing-rg"                  # echo $RG
location                = "centralindia"                         # echo $LOCATION
prefix                  = "ttapp"
suffix                  = "sk01"          # echo $SUFFIX
tfstate_storage_account = "sttfstatesk01" # echo $TFSTATE_SA

# One version BELOW the newest from: az aks get-versions -l $LOCATION -o table
kubernetes_version = "1.33"

aks_admin_object_ids    = ["11111111-1111-1111-1111-111111111111"] # echo $MY_OBJECT_ID
github_app_principal_id = "33333333-3333-3333-3333-333333333333"   # echo $APP_PRINCIPAL_ID
```

Pick the Kubernetes version:

```bash
az aks get-versions -l $LOCATION -o table
```

Choose the **second-newest** minor version (for example, if 1.34 is newest, use 1.33). In Part 16 you'll upgrade it, which answers "how did you upgrade AKS, and what challenges did you face?"

### 8.9 Apply

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks/infra/platform
terraform init -backend-config=backend.hcl
terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan
terraform apply tfplan     # 15-25 minutes (AKS and App Gateway are slow)
terraform output
```

Run it from your **laptop** for this first time. Key Vault allows your IP, so Terraform can write the `db-*` secrets. Later the pipeline runs it from the runner, which reaches Key Vault through the private endpoint.

If something fails, read the error carefully and check [Appendix A](#appendix-a--blockers-and-the-easy-way-around). Typical first-run errors: quota, SKU not available in the region, a policy denying the node resource group, or role assignments not yet applied (just run `apply` again).

### 8.10 Verify from the jumpbox

```bash
ssh azureuser@$JUMPBOX_IP
az login --use-device-code        # if the session expired

# 1) Private API server: name resolves to a PRIVATE IP from the hub VNet
az aks show -g $RG -n $AKS_NAME --query "{fqdn:privateFqdn, k8s:kubernetesVersion, state:provisioningState}" -o table
nslookup $(az aks show -g $RG -n $AKS_NAME --query privateFqdn -o tsv)   # → 10.1.4.x

# 2) kubeconfig with Entra ID login (no local admin account exists)
az aks get-credentials -g $RG -n $AKS_NAME --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes -o wide          # node IPs are 10.1.4.x (snet-aks)
kubectl get pods -A -o wide        # pod IPs are ALSO 10.1.4.x: that is Azure CNI

# 3) Add-ons running
kubectl get pods -n kube-system | grep -E "ingress-appgw|secrets-store|ama-"
kubectl get pods -A | grep -iE "calico|tigera"
kubectl describe node -l kubernetes.azure.com/mode=system | grep Taints    # CriticalAddonsOnly

# 4) PostgreSQL over the private endpoint
nslookup $PG_NAME.postgres.database.azure.com      # CNAME privatelink → 10.1.1.x
export PGPASSWORD=$(az keyvault secret show --vault-name $KV_NAME -n db-password --query value -o tsv)
psql "host=$PG_NAME.postgres.database.azure.com port=5432 dbname=appdb user=pgadmin sslmode=require" -c "select version();"

# 5) App Gateway is running
az network application-gateway show -g $RG -n $APPGW_NAME --query "{state:operationalState, sku:sku.name}" -o table
```

From your **laptop**, try `nslookup <private fqdn>` and `psql ...`. Both fail. That is the expected, secure result: the API server and the database are only reachable from inside the VNets.

Create the namespace the app will live in (your admin role can; the pipeline's namespace-scoped role cannot create namespaces):

```bash
kubectl create namespace app
```

```bash
# on your laptop
git add infra/platform && git commit -m "Platform stack: private AKS, App Gateway WAF, PostgreSQL" && git push
```

---

## Part 9 — The Helm chart

Create `helm/three-tier-app/` with these files.

### 9.1 `Chart.yaml` and `values.yaml`

```yaml
apiVersion: v2
name: three-tier-app
description: React frontend + Spring Boot backend on AKS (database is Azure PostgreSQL)
type: application
version: 0.1.0      # chart version: bump when templates change
appVersion: "1.0.0" # informational; the real image tag is passed with --set
```

```yaml
image:
  registry: acrttappsk01.azurecr.io # overridden by the pipeline
  tag: latest                       # overridden by the pipeline with the Git commit SHA
  pullPolicy: IfNotPresent

# Filled by the pipeline from Azure (az identity show / az keyvault show / az account show).
azure:
  tenantId: ""
  keyVaultName: ""
  backendClientId: ""

frontend:
  repository: ttapp-frontend
  replicas: 2
  port: 8080
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits: { memory: 128Mi }

backend:
  repository: ttapp-backend
  replicas: 2
  port: 8080
  serviceAccountName: backend # must match the federated credential subject in Terraform
  javaToolOptions: "-XX:MaxRAMPercentage=75 -javaagent:/app/applicationinsights-agent.jar"
  resources:
    requests: { cpu: 250m, memory: 512Mi }
    limits: { memory: 768Mi }
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 70

# Key Vault secret name -> environment variable in the backend container.
keyVaultSecrets:
  db-url: DB_URL
  db-username: DB_USERNAME
  db-password: DB_PASSWORD
  appinsights-connection-string: APPLICATIONINSIGHTS_CONNECTION_STRING

ingress:
  enabled: true
  className: azure-application-gateway
  host: ""                 # empty = any host (use the App Gateway public IP)
  tls:
    enabled: false
    appgwSslCertificate: "" # name of the certificate uploaded to App Gateway (Part 10)

networkPolicy:
  enabled: true
  appGatewaySubnetCidr: 10.1.0.0/24
  privateEndpointSubnetCidr: 10.1.1.0/24
  monitoringNamespace: monitoring

serviceMonitor:
  enabled: false # turn on after kube-prometheus-stack is installed (Part 11)
```

### 9.2 Helpers — `templates/_helpers.tpl`

```text
{{- define "app.labels" -}}
app.kubernetes.io/part-of: {{ .Chart.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end }}

{{- define "app.backendName" -}}{{ .Release.Name }}-backend{{- end }}
{{- define "app.frontendName" -}}{{ .Release.Name }}-frontend{{- end }}

{{- define "app.podSecurityContext" -}}
runAsNonRoot: true
seccompProfile:
  type: RuntimeDefault
{{- end }}

{{- define "app.containerSecurityContext" -}}
allowPrivilegeEscalation: false
capabilities:
  drop: ["ALL"]
{{- end }}
```

### 9.3 Backend: service account, SecretProviderClass, Deployment, Service, HPA

`templates/backend-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.backend.serviceAccountName }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    # Workload Identity: tokens for this service account are exchanged for this Azure identity.
    azure.workload.identity/client-id: {{ required "azure.backendClientId is required" .Values.azure.backendClientId | quote }}
```

`templates/backend-secretproviderclass.yaml`:

```yaml
# Tells the Secrets Store CSI driver which Key Vault secrets to fetch, and with which identity.
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: {{ include "app.backendName" . }}-kv
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: {{ .Values.azure.backendClientId | quote }} # Workload Identity
    keyvaultName: {{ required "azure.keyVaultName is required" .Values.azure.keyVaultName | quote }}
    tenantId: {{ required "azure.tenantId is required" .Values.azure.tenantId | quote }}
    objects: |
      array:
      {{- range $secret, $env := .Values.keyVaultSecrets }}
        - |
          objectName: {{ $secret }}
          objectType: secret
      {{- end }}
  # Also copy the values into a normal Kubernetes Secret so we can use them as env vars.
  # The Secret only exists while at least one pod mounts the CSI volume.
  secretObjects:
    - secretName: {{ include "app.backendName" . }}-secrets
      type: Opaque
      data:
      {{- range $secret, $env := .Values.keyVaultSecrets }}
        - objectName: {{ $secret }}
          key: {{ $env }}
      {{- end }}
```

`templates/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: backend
spec:
  {{- if not .Values.backend.hpa.enabled }}
  replicas: {{ .Values.backend.replicas }}
  {{- end }}
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # one extra pod during rollout
      maxUnavailable: 0 # never go below the desired count -> zero downtime
  template:
    metadata:
      labels:
        app: backend
        azure.workload.identity/use: "true" # webhook injects the federated token into the pod
    spec:
      serviceAccountName: {{ .Values.backend.serviceAccountName }}
      terminationGracePeriodSeconds: 45
      securityContext:
        {{- include "app.podSecurityContext" . | nindent 8 }}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: backend
      containers:
        - name: backend
          image: "{{ .Values.image.registry }}/{{ .Values.backend.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.backend.port }}
          env:
            - name: APP_VERSION
              value: {{ .Values.image.tag | quote }}
            - name: JAVA_TOOL_OPTIONS
              value: {{ .Values.backend.javaToolOptions | quote }}
            - name: APPLICATIONINSIGHTS_ROLE_NAME # name shown in the App Insights application map
              value: backend
            {{- range $secret, $env := .Values.keyVaultSecrets }}
            - name: {{ $env }}
              valueFrom:
                secretKeyRef:
                  name: {{ include "app.backendName" $ }}-secrets
                  key: {{ $env }}
            {{- end }}
          # Slow JVM start: startup probe gives up to 150s before liveness kicks in.
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: http }
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              sleep:
                seconds: 10 # let Service endpoints / App Gateway stop sending traffic first
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
          securityContext:
            {{- include "app.containerSecurityContext" . | nindent 12 }}
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: kv-secrets
              mountPath: /mnt/secrets
              readOnly: true
      volumes:
        - name: tmp
          emptyDir: {}
        - name: kv-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: {{ include "app.backendName" . }}-kv
```

`templates/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: backend
spec:
  type: ClusterIP # internal only: reachable from the frontend pods, never from the internet
  selector:
    app: backend
  ports:
    - name: http
      port: 8080
      targetPort: http
```

`templates/backend-hpa.yaml`:

```yaml
{{- if .Values.backend.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "app.backendName" . }}
  minReplicas: {{ .Values.backend.hpa.minReplicas }}
  maxReplicas: {{ .Values.backend.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization # percent of the CPU *request*
          averageUtilization: {{ .Values.backend.hpa.targetCPUUtilizationPercentage }}
{{- end }}
```

### 9.4 Frontend: Deployment and Service

`templates/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.frontendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: frontend
spec:
  replicas: {{ .Values.frontend.replicas }}
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: frontend
    spec:
      terminationGracePeriodSeconds: 30
      securityContext:
        {{- include "app.podSecurityContext" . | nindent 8 }}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: frontend
      containers:
        - name: frontend
          image: "{{ .Values.image.registry }}/{{ .Values.frontend.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.frontend.port }}
          env:
            - name: BACKEND_URL # used by nginx proxy_pass
              value: "http://{{ include "app.backendName" . }}:8080"
          # AGIC copies this readiness probe into the App Gateway health probe.
          readinessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
          lifecycle:
            preStop:
              sleep:
                seconds: 10
          resources:
            {{- toYaml .Values.frontend.resources | nindent 12 }}
          securityContext:
            {{- include "app.containerSecurityContext" . | nindent 12 }}
            readOnlyRootFilesystem: true
          # nginx needs two writable folders: its temp/pid files and the rendered config.
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: tmp
          emptyDir: {}
        - name: nginx-conf
          emptyDir: {}
```

`templates/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.frontendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: frontend
spec:
  type: ClusterIP # AGIC sends traffic straight to pod IPs; the Service is used to find them
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```

### 9.5 Ingress, PDB, NetworkPolicy, ServiceMonitor

`templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    appgw.ingress.kubernetes.io/health-probe-path: /healthz
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "30"
    {{- if .Values.ingress.tls.enabled }}
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: {{ .Values.ingress.tls.appgwSslCertificate | quote }}
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - {{- with .Values.ingress.host }}
      host: {{ . }}
      {{- end }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "app.frontendName" . }}
                port:
                  name: http
{{- end }}
```

`templates/pdb.yaml`:

```yaml
# Keeps at least one pod of each tier running during node drains (upgrades, autoscaler scale-down).
{{- range $tier := list "frontend" "backend" }}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ $.Release.Name }}-{{ $tier }}
  labels:
    {{- include "app.labels" $ | nindent 4 }}
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {{ $tier }}
{{- end }}
```

`templates/networkpolicy.yaml`:

```yaml
{{- if .Values.networkPolicy.enabled }}
# 1) Deny everything in this namespace by default (zero trust).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# 2) Every pod may resolve DNS through CoreDNS.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
---
# 3) Frontend: only App Gateway may call it; it may only call the backend.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - ipBlock:
            cidr: {{ .Values.networkPolicy.appGatewaySubnetCidr }}
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - { protocol: TCP, port: 8080 }
---
# 4) Backend: called by frontend and Prometheus; may call PostgreSQL and HTTPS (App Insights).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.networkPolicy.monitoringNamespace }}
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - ipBlock:
            cidr: {{ .Values.networkPolicy.privateEndpointSubnetCidr }}
      ports:
        - { protocol: TCP, port: 5432 }
    - ports:
        - { protocol: TCP, port: 443 }
{{- end }}
```

`templates/servicemonitor.yaml`:

```yaml
{{- if .Values.serviceMonitor.enabled }}
# Tells Prometheus (kube-prometheus-stack) to scrape Spring Boot's /actuator/prometheus.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
{{- end }}
```

### 9.6 Kubernetes concepts in this chart (know each one)

| Concept | Where | Explain it like this |
|---|---|---|
| **startupProbe** | backend | "JVM needs up to ~60 s to start. The startup probe gives it up to 150 s; liveness and readiness don't run until it passes, so a slow start is never killed." |
| **livenessProbe** | both | "If it fails 3 times, kubelet **restarts** the container (deadlock, hung JVM). It must not check the database, or a DB outage would restart every pod." |
| **readinessProbe** | both | "If it fails, the pod is removed from Service endpoints and from the App Gateway pool, but **not restarted**. Our readiness includes the DB." |
| **App Gateway health probe** | Ingress annotation | "AGIC builds the App Gateway probe from the readiness probe / `health-probe-path`. If it fails: **502 Bad Gateway**." |
| **RollingUpdate `maxSurge: 1, maxUnavailable: 0`** | both | "A new pod must be Ready before an old one is removed: zero downtime." |
| **preStop sleep + graceful shutdown** | both | "On delete, the endpoint removal and SIGTERM happen in parallel. Sleeping 10 s first lets App Gateway and kube-proxy stop sending traffic; then Spring finishes in-flight requests." |
| **requests / limits** | both | "Requests are for scheduling (and HPA %), limits are the cap. Memory over the limit = **OOMKilled**. No CPU limit, so no CPU throttling." |
| **HPA** | backend | "Scales pods 2–5 on CPU above 70% of the *request*. Needs metrics-server (built into AKS)." |
| **PDB `minAvailable: 1`** | both | "A node drain (upgrade, autoscaler) can never evict the last pod of a tier." |
| **topologySpreadConstraints** | both | "Spread replicas across nodes, so one node failure doesn't take both." |
| **securityContext** | both | "Non-root, no privilege escalation, all Linux capabilities dropped, seccomp RuntimeDefault, read-only root filesystem (writable `emptyDir` only where needed)." |
| **ServiceAccount + label `azure.workload.identity/use`** | backend | "The webhook injects `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and the projected token file into the pod." |
| **SecretProviderClass + `secretObjects`** | backend | "The CSI driver mounts Key Vault secrets as files **and** syncs a Kubernetes Secret, which we use as env vars. The synced Secret exists only while a pod mounts the volume." |
| **NetworkPolicy** | namespace | "Default deny. Then: App Gateway subnet → frontend:8080, frontend → backend:8080, monitoring → backend:8080, backend → PE subnet:5432, everyone → CoreDNS:53." |
| **ClusterIP Services** | both | "Stable virtual IP + DNS name in front of changing pod IPs. Not reachable from outside the cluster." |

### 9.7 Lint and test the chart locally

```bash
cd ~/three-tier-aks
SET="azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x"
helm lint helm/three-tier-app --set $SET
helm template ttapp helm/three-tier-app -n app --set $SET > /tmp/rendered.yaml
less /tmp/rendered.yaml       # read what Helm really sends to Kubernetes

# Schema check against the Kubernetes API (CRDs like SecretProviderClass are skipped)
docker run --rm -v /tmp:/t ghcr.io/yannh/kubeconform:latest -strict -summary -ignore-missing-schemas /t/rendered.yaml

# Security misconfiguration check
docker run --rm -v "$PWD":/src -w /src aquasec/trivy:0.76.1 config --severity HIGH,CRITICAL \
  --ignorefile /src/.trivyignore --helm-set $SET /src/helm/three-tier-app
```

```bash
git add helm && git commit -m "Helm chart" && git push
```

---

## Part 10 — First deployment by hand

Deploy once **manually** before automating it, so you understand every step the pipeline will do.

### 10.1 Push images from your laptop

ACR is public (Basic SKU) and your user has Contributor, so you can push from your laptop.

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks
az acr login -n $ACR_NAME
TAG=manual-1
for app in backend frontend; do
  docker build -t $ACR_NAME.azurecr.io/ttapp-$app:$TAG $app
  docker push $ACR_NAME.azurecr.io/ttapp-$app:$TAG
done
az acr repository list -n $ACR_NAME -o table
az acr repository show-tags -n $ACR_NAME --repository ttapp-backend -o table
```

### 10.2 Install with Helm from the jumpbox

```bash
ssh azureuser@$JUMPBOX_IP
git clone https://github.com/$GH_OWNER/$GH_REPO.git && cd $GH_REPO

BACKEND_CLIENT_ID=$(az identity show -g $RG -n id-ttapp-dev-backend --query clientId -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

helm upgrade --install ttapp helm/three-tier-app -n app \
  --rollback-on-failure --wait --timeout 10m \
  --set image.registry=$ACR_NAME.azurecr.io --set image.tag=manual-1 \
  --set azure.tenantId=$TENANT_ID --set azure.keyVaultName=$KV_NAME \
  --set azure.backendClientId=$BACKEND_CLIENT_ID
```

While it runs, watch from a second SSH session:

```bash
kubectl get pods -n app -w
```

You'll see backend pods go `ContainerCreating` (the CSI driver fetches secrets) → `Running 0/1` (startup probe) → `1/1` (readiness passed).

### 10.3 Check every layer

```bash
kubectl get deploy,rs,pods,svc,ingress,hpa,pdb,networkpolicy -n app -o wide
kubectl get endpointslices -n app                       # pod IPs behind each Service

# Ingress ADDRESS = App Gateway public IP (AGIC fills it in, takes ~1 min)
APPGW_IP=$(kubectl get ingress ttapp -n app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://$APPGW_IP/api/info; echo
for i in $(seq 1 6); do curl -s http://$APPGW_IP/api/info; echo; done    # pod name changes: load balancing

# What AGIC wrote into App Gateway
az network application-gateway show -g $RG -n $APPGW_NAME \
  --query "{listeners:httpListeners[].name, pools:backendAddressPools[].{name:name, ips:backendAddresses[].ipAddress}, probes:probes[].{path:path}}" -o json
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{ip:address, health:health}" -o table
```

The backend pool IPs are exactly the **frontend pod IPs** from `kubectl get pods -n app -o wide`. That's AGIC + Azure CNI.

Open `http://<APPGW_IP>` in the browser and add some todos.

### 10.4 Check the secrets path

```bash
kubectl describe pod -n app -l app=backend | grep -A6 "Environment:"   # AZURE_CLIENT_ID etc. injected by webhook
kubectl get secretproviderclass -n app
kubectl get secret ttapp-backend-secrets -n app -o jsonpath='{.data.DB_USERNAME}' | base64 -d; echo   # pgadmin
kubectl get events -n app --sort-by=.lastTimestamp | tail -20
```

### 10.5 Debug a distroless container

```bash
kubectl exec -n app deploy/ttapp-backend -- sh          # fails: no shell in distroless

# The pod has runAsNonRoot: true, which also applies to debug containers,
# so run busybox as the same non-root user (65532) as the Java process.
echo '{"securityContext":{"runAsUser":65532,"runAsNonRoot":true}}' > /tmp/debug.json
kubectl debug -n app -it $(kubectl get pod -n app -l app=backend -o name | head -1) \
  --image=busybox:1.36 --target=backend --custom=/tmp/debug.json -- sh
# inside the debug container (shares the backend's process namespace):
ls /proc/1/root/mnt/secrets          # the files mounted by the CSI driver
wget -qO- localhost:8080/actuator/health
exit
```

### 10.6 Look at the database

```bash
export PGPASSWORD=$(az keyvault secret show --vault-name $KV_NAME -n db-password --query value -o tsv)
psql "host=$PG_NAME.postgres.database.azure.com port=5432 dbname=appdb user=pgadmin sslmode=require" \
  -c "select * from flyway_schema_history;" -c "select * from todo;"
```

The todos you added in the browser are now in PostgreSQL, so all three tiers work end to end.

---

## Part 11 — SonarQube Cloud

We use **SonarQube Cloud** (formerly SonarCloud). It is free for public repositories and needs no server. The checks and quality gates are the same as a self-hosted SonarQube server, so everything you learn applies to both. (The self-hosted option is in Appendix A.)

> SonarSource changes its UI from time to time. If a button has a slightly different name, look for the same idea.

1. Go to <https://sonarcloud.io> → **Log in with GitHub**.
2. **+ → Analyze new project → Import an organization** → choose your GitHub account → choose the **Free** plan. Note the **organization key** (for example `yourname`).
3. Select the `three-tier-aks` repository. It is a monorepo (two apps), so choose **monorepo / set up multiple projects** and create two projects:
   - key `<org>_backend`, name `three-tier backend`
   - key `<org>_frontend`, name `three-tier frontend`
4. For **each** project: **Administration → Analysis Method → turn OFF Automatic Analysis**. We analyse from CI, and the two methods cannot run together.
5. For each project, **Administration → New Code**: set "Previous version" (or "Number of days: 30").
6. **My Account → Security → Generate token**. Then in GitHub:
   - secret `SONAR_TOKEN` = the token
   - variable `SONAR_ORG` = your organization key

**What SonarQube checks (interview answer):**

| Category | Meaning | Example |
|---|---|---|
| Bugs (Reliability) | code that is likely wrong | null dereference, resource leak |
| Vulnerabilities (Security) | code that is exploitable | SQL built by string concatenation |
| Security Hotspots | code a human must review | hard-coded IP, weak crypto, CORS `*` |
| Code Smells (Maintainability) | hard-to-maintain code | long methods, duplicated blocks, unused imports |
| Coverage | % of lines run by tests | from `jacoco.xml` |
| Duplications | copy-pasted code | |

The **Quality Gate** ("Sonar way") is a pass/fail rule on **new code**: for example coverage ≥ 80%, no new bugs or vulnerabilities, and every hotspot reviewed. `-Dsonar.qualitygate.wait=true` makes the CI step **fail** when the gate fails, which blocks the merge.

> Your backend has only 3 tests. If the gate fails on coverage, that is a real result: add a test, or adjust the gate in SonarQube Cloud. Talk about this trade-off in interviews.

---

## Part 12 — GitHub Actions CI/CD

### 12.1 Application pipeline — `.github/workflows/app-ci-cd.yml`

```yaml
name: app-ci-cd

on:
  pull_request:
    branches: [main]
    paths: ["backend/**", "frontend/**", "helm/**", ".github/workflows/app-ci-cd.yml"]
  push:
    branches: [main]
    paths: ["backend/**", "frontend/**", "helm/**", ".github/workflows/app-ci-cd.yml"]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: app-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }} # never cancel a running deploy

env:
  TRIVY_IMAGE: aquasec/trivy:0.76.1
  GITLEAKS_IMAGE: ghcr.io/gitleaks/gitleaks:v8.30.1
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP }}
  ACR_NAME: ${{ vars.ACR_NAME }}
  AKS_NAME: ${{ vars.AKS_NAME }}
  RELEASE: ttapp
  NAMESPACE: app

jobs:
  # ---------------- CI: runs on every PR and every push ----------------
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # scan the whole Git history, not just the last commit
      - name: Gitleaks
        run: docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/repo" "$GITLEAKS_IMAGE" git /repo --redact --no-banner

  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # Sonar uses Git history for "new code"
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Build and unit test
        run: mvn -B verify
      - name: SonarQube Cloud scan + quality gate
        if: env.SONAR_TOKEN != ''
        run: >
          mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
          -Dsonar.host.url=https://sonarcloud.io
          -Dsonar.organization=${{ vars.SONAR_ORG }}
          -Dsonar.projectKey=${{ vars.SONAR_ORG }}_backend
          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
          -Dsonar.qualitygate.wait=true
      - name: Trivy dependency + secret scan
        # --offline-scan + the Maven cache: no extra calls to Maven Central (avoids HTTP 429).
        run: >
          docker run --rm -v "$PWD:/src" -v "$HOME/.m2:/root/.m2" "$TRIVY_IMAGE"
          fs --offline-scan --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed
          --exit-code 1 --skip-dirs /src/target /src

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v7
        with:
          node-version: "24"
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci --no-audit --no-fund
      - run: npm run build
      - name: SonarQube Cloud scan + quality gate
        if: env.SONAR_TOKEN != ''
        uses: SonarSource/sonarqube-scan-action@v8
        with:
          projectBaseDir: frontend
          args: >
            -Dsonar.host.url=https://sonarcloud.io
            -Dsonar.organization=${{ vars.SONAR_ORG }}
            -Dsonar.projectKey=${{ vars.SONAR_ORG }}_frontend
            -Dsonar.sources=src
            -Dsonar.qualitygate.wait=true
      - name: Trivy dependency + secret scan
        run: >
          docker run --rm -v "$PWD:/src" "$TRIVY_IMAGE"
          fs --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed
          --exit-code 1 --skip-dirs /src/node_modules /src

  helm-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: helm lint
        run: helm lint helm/three-tier-app --set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x
      - name: Trivy misconfiguration scan
        run: >
          docker run --rm -v "$PWD:/src" -w /src "$TRIVY_IMAGE"
          config --severity HIGH,CRITICAL --exit-code 1 --ignorefile /src/.trivyignore
          --helm-set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x
          /src/helm/three-tier-app

  # ---------------- Build, scan and push images (main branch only) ----------------
  build-push:
    needs: [secret-scan, backend, frontend, helm-scan]
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # needed to request the OIDC token for azure/login
    strategy:
      matrix:
        app: [frontend, backend]
    env:
      IMAGE: ${{ vars.ACR_NAME }}.azurecr.io/ttapp-${{ matrix.app }}:${{ github.sha }}
    steps:
      - uses: actions/checkout@v7
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_APP_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Log in to ACR
        run: az acr login --name "$ACR_NAME"
      - uses: docker/setup-buildx-action@v4
      - name: Build image (not pushed yet)
        uses: docker/build-push-action@v7
        with:
          context: ${{ matrix.app }}
          load: true # keep it in the local Docker daemon so Trivy can scan it
          push: false
          tags: ${{ env.IMAGE }}
          cache-from: type=gha,scope=${{ matrix.app }}
          cache-to: type=gha,mode=max,scope=${{ matrix.app }}
      - name: Trivy image scan (blocks the push on HIGH/CRITICAL)
        run: >
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock "$TRIVY_IMAGE"
          image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE"
      - name: Push image
        run: docker push "$IMAGE"

  # ---------------- Deploy to the private AKS cluster ----------------
  deploy-dev:
    needs: build-push
    runs-on: [self-hosted, linux, aks-deployer] # the runner VM inside the hub VNet
    environment: dev
    permissions:
      contents: read
      id-token: write
    env:
      KUBECONFIG: ${{ github.workspace }}/.kubeconfig
    steps:
      - uses: actions/checkout@v7
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_APP_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Read values from Azure
        run: |
          {
            echo "KV_NAME=$(az keyvault list -g "$RESOURCE_GROUP" --query '[0].name' -o tsv)"
            echo "BACKEND_CLIENT_ID=$(az identity show -g "$RESOURCE_GROUP" -n id-ttapp-dev-backend --query clientId -o tsv)"
            echo "TENANT_ID=$(az account show --query tenantId -o tsv)"
            echo "APPGW_IP=$(az network public-ip show -g "$RESOURCE_GROUP" -n pip-ttapp-dev-appgw --query ipAddress -o tsv)"
          } >> "$GITHUB_ENV"
      - name: Get kubeconfig (Entra ID auth)
        run: |
          az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --file "$KUBECONFIG" --overwrite-existing
          kubelogin convert-kubeconfig -l azurecli --kubeconfig "$KUBECONFIG"
      - name: Helm upgrade (auto-rollback on failure)
        run: |
          helm upgrade --install "$RELEASE" helm/three-tier-app \
            --namespace "$NAMESPACE" \
            --rollback-on-failure --wait --timeout 10m --history-max 10 \
            --set image.registry="$ACR_NAME.azurecr.io" \
            --set image.tag="${{ github.sha }}" \
            --set azure.tenantId="$TENANT_ID" \
            --set azure.keyVaultName="$KV_NAME" \
            --set azure.backendClientId="$BACKEND_CLIENT_ID" \
            --set serviceMonitor.enabled="${{ vars.SERVICE_MONITOR_ENABLED || 'false' }}" \
            --set ingress.tls.enabled="${{ vars.TLS_ENABLED || 'false' }}" \
            --set ingress.tls.appgwSslCertificate=ttapp-selfsigned
      - name: Smoke test through App Gateway
        run: |
          for i in $(seq 1 20); do
            if curl -fsS "http://$APPGW_IP/api/info"; then echo; echo "Smoke test passed"; exit 0; fi
            echo "waiting for App Gateway... ($i)"; sleep 15
          done
          echo "Smoke test failed"; exit 1
      - name: Debug info on failure
        if: failure()
        run: |
          kubectl -n "$NAMESPACE" get pods,svc,ingress -o wide || true
          kubectl -n "$NAMESPACE" get events --sort-by=.lastTimestamp | tail -30 || true
          helm -n "$NAMESPACE" history "$RELEASE" || true
```

### 12.2 Infrastructure pipeline — `.github/workflows/infra.yml`

````yaml
name: infra

on:
  pull_request:
    branches: [main]
    paths: ["infra/**", ".github/workflows/infra.yml"]
  workflow_dispatch:
    inputs:
      stack:
        description: Which Terraform stack
        type: choice
        options: [foundation, platform]
        default: platform
      action:
        description: What to do
        type: choice
        options: [plan, apply, destroy]
        default: plan

permissions:
  contents: read

concurrency:
  group: infra-${{ github.event.inputs.stack || 'pr' }} # never two applies on the same state
  cancel-in-progress: false

env:
  TF_VERSION: "1.17.0"
  TRIVY_IMAGE: aquasec/trivy:0.76.1
  # Terraform's azurerm provider + backend use these for OIDC login (no client secret).
  ARM_USE_OIDC: "true"
  ARM_USE_AZUREAD: "true"
  ARM_CLIENT_ID: ${{ vars.AZURE_INFRA_CLIENT_ID }}
  ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  TF_VAR_my_ip: ${{ secrets.TF_VAR_MY_IP }}
  TF_VAR_alert_email: ${{ secrets.TF_VAR_ALERT_EMAIL }}
  TF_IN_AUTOMATION: "true"

jobs:
  # Static checks: no Azure access needed, safe on GitHub-hosted runners.
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [foundation, platform]
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - run: terraform fmt -check -recursive
        working-directory: infra/${{ matrix.stack }}
      - run: terraform init -backend=false -input=false
        working-directory: infra/${{ matrix.stack }}
      - run: terraform validate
        working-directory: infra/${{ matrix.stack }}
      - name: Trivy IaC scan
        run: >
          docker run --rm -v "$PWD:/src" -w /src "$TRIVY_IMAGE"
          config --severity HIGH,CRITICAL --exit-code 1 --ignorefile /src/.trivyignore
          /src/infra/${{ matrix.stack }}

  # Plan on pull requests (same-repo PRs only: fork PRs must never reach the self-hosted runner).
  plan-pr:
    needs: validate
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: [self-hosted, linux, aks-deployer]
    permissions:
      contents: read
      id-token: write
    strategy:
      max-parallel: 1
      matrix:
        stack: [foundation, platform]
    defaults:
      run:
        working-directory: infra/${{ matrix.stack }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false
      - run: terraform init -input=false -backend-config=backend.hcl
      - name: terraform plan
        run: |
          terraform plan -input=false -lock-timeout=5m -var-file=dev.tfvars -no-color -out=tfplan | tee plan.txt
          {
            echo "### Plan for \`${{ matrix.stack }}\`"
            echo '```'
            grep -E '^(Plan:|No changes)' plan.txt || true
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

  # Manual run: plan / apply / destroy one stack. "dev" environment can require your approval.
  run:
    needs: validate
    if: github.event_name == 'workflow_dispatch'
    runs-on: [self-hosted, linux, aks-deployer]
    environment: dev
    permissions:
      contents: read
      id-token: write
    defaults:
      run:
        working-directory: infra/${{ inputs.stack }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false
      - run: terraform init -input=false -backend-config=backend.hcl
      - name: terraform plan
        if: inputs.action != 'destroy'
        run: terraform plan -input=false -lock-timeout=5m -var-file=dev.tfvars -out=tfplan
      - name: terraform apply
        if: inputs.action == 'apply'
        run: terraform apply -input=false -auto-approve tfplan
      - name: terraform destroy
        if: inputs.action == 'destroy'
        run: terraform destroy -input=false -auto-approve -lock-timeout=5m -var-file=dev.tfvars
````

### 12.3 What happens, stage by stage

```text
Pull request ─► secret-scan ─┐
             ─► backend ─────┤ (build, unit test, JaCoCo, SonarQube gate, Trivy fs)
             ─► frontend ────┤ (npm ci, build, SonarQube gate, Trivy fs)
             ─► helm-scan ───┘ (helm lint, Trivy config)
                               all must pass to merge (branch ruleset)

Push to main ─► same CI jobs ─► build-push (matrix: frontend, backend)
                                  azure/login (OIDC) → buildx (GHA cache) → Trivy image → push :<git sha>
                               ─► deploy-dev  [waits for YOUR approval: environment "dev"]
                                  self-hosted runner in hub VNet → az login (OIDC) → kubeconfig (kubelogin)
                                  → helm upgrade --rollback-on-failure → smoke test through App Gateway
```

Design decisions to explain:

- **Image tag = Git commit SHA**, never `latest`. Every running pod points to an exact commit, and rollback means "deploy the previous SHA".
- **Scan before push.** A vulnerable image never reaches the registry.
- **Build once, deploy many.** QA and Prod would deploy the **same** image and digest that passed dev, promoted rather than rebuilt (Appendix B).
- **`concurrency`.** PR runs cancel older runs of the same PR; deploys are never cancelled halfway.
- **`permissions`** is read-only by default; `id-token: write` only on the jobs that log in to Azure.
- **Fork safety.** Fork PRs never reach the self-hosted runner (`head.repo.full_name == github.repository`), plus the repo setting from Part 2.4.
- **Pin versions** (`aquasec/trivy:0.76.1`, `actions/checkout@v7`). For stronger supply-chain security, pin actions to a **commit SHA**. Tag hijacking has happened in real attacks; the `tj-actions/changed-files` compromise in March 2025 is the well-known example.
- **Ways a GitHub workflow can be triggered:** `push`, `pull_request`, `workflow_dispatch` (manual button), `schedule` (cron), `workflow_run` (after another workflow), `repository_dispatch` (API call), `release`. This answers "in how many ways can a pipeline run?"

### 12.4 Run it

```bash
cd ~/three-tier-aks
git add .github && git commit -m "CI/CD workflows" && git push
```

1. **Actions → app-ci-cd**: the CI jobs run on GitHub-hosted runners, then `build-push`.
2. `deploy-dev` shows **"Waiting for review"**. Click **Review deployments → Approve**.
3. The job runs on `jumpbox-runner`. Watch the Helm output and the smoke test.
4. On the jumpbox: `helm history ttapp -n app` shows revision 2 with the Git SHA image.

Now practise the **PR flow**:

```bash
git checkout -b feature/title-length
# change @Size(max = 200) to @Size(max = 100) in TodoController.java
git commit -am "Limit title to 100 chars" && git push -u origin feature/title-length
# open a PR on GitHub: CI runs, SonarQube posts its result, merge only when green
```

After the first run, set up the **branch ruleset** from Part 2.4 (the check names now exist).

### 12.5 Run Terraform from the pipeline

**Actions → infra → Run workflow → stack: `platform`, action: `plan`**. It runs on the self-hosted runner. Approve the `dev` environment, then read the plan in the log. It should show **No changes** if nothing has drifted.

From now on you can create and destroy the platform stack **from GitHub** (`apply` / `destroy`) instead of from your laptop. Interview line: *"Terraform runs only in the pipeline: PRs get a plan, and apply needs approval through the environment protection rule. State is in Azure Storage with blob-lease locking, and the pipeline logs in with OIDC."*

**Interview question — "the whole company shares one state file; how do multiple people work on it?"** Remote backend + **state locking** (blob lease) lets only one plan or apply run at a time; the other waits (`-lock-timeout=5m`) or fails with the lock ID. Better still: **split the state** by component and environment (as we did: foundation and platform), so teams rarely touch the same state.

---

## Part 13 — HTTPS and WAF

### 13.1 Self-signed certificate on App Gateway

There's no real domain, so use a self-signed certificate. `nip.io` gives a hostname for any IP (`1.2.3.4.nip.io` → 1.2.3.4).

```bash
# laptop
source ~/three-tier-env.sh
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
HOST="$APPGW_IP.nip.io"

openssl req -x509 -nodes -days 30 -newkey rsa:2048 -keyout tls.key -out tls.crt \
  -subj "/CN=$HOST" -addext "subjectAltName=DNS:$HOST"
openssl pkcs12 -export -out tls.pfx -inkey tls.key -in tls.crt -passout pass:LabOnly-123

az network application-gateway ssl-cert create -g $RG --gateway-name $APPGW_NAME \
  -n ttapp-selfsigned --cert-file tls.pfx --cert-password 'LabOnly-123'
rm tls.key tls.pfx      # never keep private keys around (and .gitignore blocks them)
```

Set the GitHub variable **`TLS_ENABLED` = `true`** and re-run the **app-ci-cd** workflow (Run workflow on `main`). The Ingress gets two annotations:

- `appgw-ssl-certificate: ttapp-selfsigned`: AGIC creates an HTTPS listener on 443 with that certificate.
- `ssl-redirect: "true"`: AGIC adds an HTTP→HTTPS redirect.

```bash
curl -sI http://$APPGW_IP/ | head -3          # 301 Moved Permanently → https://
curl -sk https://$APPGW_IP/api/info; echo     # -k because the certificate is self-signed
```

**TLS termination.** HTTPS ends at App Gateway; App Gateway → pod is HTTP inside the VNet. For end-to-end TLS you would also give the pods a certificate and set `appgw.ingress.kubernetes.io/backend-protocol: "https"`.

> The certificate lives on the App Gateway. When you destroy and recreate the platform stack, upload it again (or set `TLS_ENABLED=false`). The production way is a Key Vault certificate referenced by App Gateway through a managed identity, with auto-renewal. Then expiry is handled by Key Vault, which answers "how do you make sure an expiring certificate doesn't break the app?" See Appendix C.

### 13.2 Test the WAF

```bash
# Normal request: 200
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/api/todos"

# SQL injection pattern: blocked by the OWASP 3.2 managed rules → 403
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/api/todos?id=1%27%20OR%20%271%27=%271"

# XSS pattern → 403
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/?q=<script>alert(1)</script>"

# Our custom rule (User-Agent contains "BadBot") → 403
curl -s -o /dev/null -w "%{http_code}\n" -A "BadBot/1.0" "http://$APPGW_IP/"
```

See *why* it was blocked: **Log Analytics → Logs** (logs arrive in ~5 minutes):

```kusto
AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where Action == "Blocked" or Action == "Matched"
| project TimeGenerated, ClientIp, RequestUri, RuleId, Message, Action
| order by TimeGenerated desc
```

If a legitimate request is blocked (a **false positive**): find the `RuleId` here, then add an **exclusion** or disable that rule in the WAF policy, in Terraform. Run new rules in `Detection` mode first, then switch to `Prevention`.

**"How do you restrict users before requests reach the application?"** Layer by layer: WAF managed and custom rules (OWASP, bots, geo-match, rate limiting), then NSG on the App Gateway subnet, then App Gateway listener and host rules, then NetworkPolicy in the cluster, then app-level authentication.

---

## Part 14 — Monitoring with Azure Monitor

### 14.1 How the pieces fit (interview answer)

```text
AKS nodes/pods ──(ama-logs agent, DCR)──► Log Analytics: ContainerLogV2, KubePodInventory, KubeEvents, Perf...
AKS control plane ──(diagnostic setting)──► Log Analytics: AKSAuditAdmin, AKSControlPlane
App Gateway ──(diagnostic setting)──► Log Analytics: AGWAccessLogs, AGWFirewallLogs
Spring Boot ──(App Insights Java agent)──► Application Insights ──► same workspace: AppRequests, AppDependencies, AppExceptions
                                                    │
Metric alert (node CPU) / Log alert (CrashLoopBackOff KQL) ──► Action group ──► email
```

- **Azure Monitor** is the umbrella service.
- **Log Analytics** stores logs; you query them with **KQL**.
- **Container Insights** collects Kubernetes logs and inventory.
- **Application Insights** collects application telemetry: requests, dependencies (SQL calls), exceptions, the application map, and live metrics. The Java agent does this **without code changes**.

### 14.2 Generate some traffic first

```bash
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
for i in $(seq 1 300); do curl -s -o /dev/null http://$APPGW_IP/api/todos; curl -s -o /dev/null http://$APPGW_IP/api/nope; done
```

### 14.3 Container Insights

Portal → AKS cluster → **Monitoring → Insights**: Cluster, Nodes, Controllers and Containers tabs. Then **Logs**:

```kusto
// Errors in application logs (last hour)
ContainerLogV2
| where TimeGenerated > ago(1h) and PodNamespace == "app"
| where LogMessage has_any ("ERROR", "Exception")
| project TimeGenerated, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
```

```kusto
// Latest status and restart count of every pod in "app"
KubePodInventory
| where TimeGenerated > ago(30m) and Namespace == "app"
| summarize arg_max(TimeGenerated, PodStatus, ContainerRestartCount, Node = Computer) by Name
```

```kusto
// Kubernetes warning events (FailedScheduling, BackOff, FailedMount, Unhealthy...)
KubeEvents
| where TimeGenerated > ago(1h) and Namespace == "app" and KubeEventType == "Warning"
| project TimeGenerated, Name, Reason, Message
| order by TimeGenerated desc
```

```kusto
// Who changed what in the cluster (audit, no read calls)
AKSAuditAdmin
| where TimeGenerated > ago(1d)
| project TimeGenerated, User = tostring(User.username), Verb, ObjectRef = strcat(ObjectRef.resource, "/", ObjectRef.name)
| order by TimeGenerated desc
```

### 14.4 Application Insights

Portal → **appi-ttapp-dev**:

- **Application map**: `backend` → PostgreSQL (`appdb`), with call counts and latency.
- **Failures**: your `/api/nope` calls show as 404s; drill into one request.
- **Performance**: slowest operations and their SQL dependencies.
- **Live metrics**: real-time requests while you run the loop above.

The same data in KQL (workspace tables):

```kusto
AppRequests
| where TimeGenerated > ago(1h)
| summarize count(), avg(DurationMs), percentile(DurationMs, 95) by Name, ResultCode
| order by count_ desc
```

### 14.5 Alerts

Terraform created two alerts: **node CPU > 80%** (metric alert) and **CrashLoopBackOff** (log alert, KQL every 5 minutes). Look at them in Portal → Monitor → **Alerts → Alert rules**. You trigger the CrashLoop alert for real in [Lab 2](#lab-2--crashloopbackoff) and get an email.

**Metric vs log alerts:** metric alerts are near real-time and cheap, but only cover numbers Azure already collects. Log alerts run any KQL query, which is more flexible, but have a delay (evaluation frequency + ingestion latency) and cost per rule.

---

## Part 15 — Monitoring with Prometheus and Grafana

We install the open-source **kube-prometheus-stack** with Helm: Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics and node-exporter. The managed Azure version (Azure Monitor managed Prometheus + Azure Managed Grafana) is in Appendix C.

### 15.1 Values and alert rules

`k8s/monitoring/kube-prometheus-stack-values.yaml`:

```yaml
# AKS hides the control plane, so Prometheus cannot scrape these. Disabling them avoids "target down" noise.
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeEtcd:
  enabled: false
kubeProxy:
  enabled: false

prometheus:
  prometheusSpec:
    # Pick up ServiceMonitors / PrometheusRules from ALL namespaces, not only ones labelled for this release.
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    retention: 2d
    resources:
      requests: { cpu: 200m, memory: 512Mi }
      limits: { memory: 1536Mi }

alertmanager:
  alertmanagerSpec:
    resources:
      requests: { cpu: 20m, memory: 64Mi }
      limits: { memory: 128Mi }

grafana:
  persistence:
    enabled: false # lab only: dashboards you create in the UI are lost on restart
  resources:
    requests: { cpu: 50m, memory: 128Mi }
    limits: { memory: 256Mi }
```

`k8s/monitoring/backend-alerts.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backend-alerts
  namespace: monitoring
spec:
  groups:
    - name: backend
      rules:
        - alert: BackendHigh5xxRate
          expr: |
            sum(rate(http_server_requests_seconds_count{application="backend", status=~"5.."}[5m]))
              /
            sum(rate(http_server_requests_seconds_count{application="backend"}[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "More than 5% of backend requests return 5xx"

        - alert: BackendTargetDown
          expr: up{namespace="app", service=~".*backend.*"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Prometheus cannot scrape backend pod {{ $labels.pod }}"

        - alert: AppPodRestarting
          expr: increase(kube_pod_container_status_restarts_total{namespace="app"}[15m]) > 2
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} restarted more than twice in 15 minutes"
```

### 15.2 Install (from the jumpbox)

```bash
cd ~/three-tier-aks && git pull
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack     # note the CHART VERSION (92.x when this was written)

kubectl create namespace monitoring
helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring \
  --version <CHART VERSION> -f k8s/monitoring/kube-prometheus-stack-values.yaml --wait --timeout 15m
kubectl get pods -n monitoring
kubectl apply -f k8s/monitoring/backend-alerts.yaml
```

Always pin `--version` for third-party charts, so an upstream release never changes your cluster by surprise.

### 15.3 Let Prometheus scrape the backend

Set the GitHub variable **`SERVICE_MONITOR_ENABLED` = `true`** and re-run **app-ci-cd**. The chart now creates a `ServiceMonitor`. The Prometheus Operator turns it into scrape config for `/actuator/prometheus` on every backend pod. The NetworkPolicy already allows the `monitoring` namespace to reach the backend on 8080.

### 15.4 Open the UIs through an SSH tunnel

```bash
# laptop: one SSH session that forwards two ports
ssh -L 3000:localhost:3000 -L 9090:localhost:9090 azureuser@$JUMPBOX_IP

# on the jumpbox (inside that session)
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 >/dev/null &
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 >/dev/null &
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open <http://localhost:9090> (Prometheus) and <http://localhost:3000> (Grafana, user `admin` + the password above) on your laptop.

### 15.5 Prometheus: targets and PromQL

**Status → Targets**: `serviceMonitor/app/ttapp-backend/0` shows 2 targets **UP**. If it says DOWN or the target is missing, check the ServiceMonitor labels, the Service port name (`http`), and the NetworkPolicy.

Run these queries in the Prometheus UI (generate traffic first):

```promql
# Requests per second by endpoint and status
sum by (uri, status) (rate(http_server_requests_seconds_count{application="backend"}[1m]))

# p95 latency (works because percentiles-histogram is enabled)
histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket{application="backend"}[5m])))

# Error ratio (what the BackendHigh5xxRate alert uses)
sum(rate(http_server_requests_seconds_count{application="backend",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{application="backend"}[5m]))

# JVM heap and DB connection pool
sum by (pod) (jvm_memory_used_bytes{application="backend", area="heap"})
hikaricp_connections_active{application="backend"}

# Kubernetes state (kube-state-metrics)
kube_deployment_status_replicas_available{namespace="app"}
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="app", container!=""}[5m]))
```

**Alerts** tab: your three rules from `backend-alerts.yaml` are listed (green = not firing).

### 15.6 Grafana dashboards

1. **Dashboards → Browse**: the built-in "Kubernetes / Compute Resources / Namespace (Pods)" → namespace `app`.
2. **Dashboards → New → Import** → ID **4701** ("JVM (Micrometer)") → choose the Prometheus data source.
3. Build your own dashboard with 4 panels using the PromQL above: RPS, p95 latency, error %, and pods available. This is the **RED** method (Rate, Errors, Duration) for services. The **USE** method (Utilization, Saturation, Errors) is for nodes and resources.

### 15.7 When to use which (interview answer)

| Need | Tool |
|---|---|
| Pod and container logs, search, retention, audit | Log Analytics (Container Insights) |
| Request traces, dependency map, exceptions | Application Insights |
| Real-time metrics, custom app metrics, flexible PromQL, cheap long-running dashboards | Prometheus + Grafana |
| Alerts on Kubernetes or app metrics | Prometheus Alertmanager (or managed Prometheus alert rules) |
| Alerts on Azure resources (App Gateway, PostgreSQL, node CPU), log queries | Azure Monitor alerts + action groups |

Investigation order for "10% of requests are failing": App Gateway access logs and backend health (is it the edge?) → Prometheus error rate by `uri` and `pod` (one pod or all? one endpoint?) → App Insights failures (exception + SQL dependency) → pod logs and events (restarts, OOM, probes) → PostgreSQL metrics (connections, CPU).

---

## Part 16 — Kubernetes exercises

Run these on the jumpbox. Keep a second SSH session open with `kubectl get pods -n app -w` so you can watch the changes live.

```bash
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
```

### 16.1 Zero-downtime rolling update (prove it)

Terminal 1 sends a request every 0.2 s and counts the status codes:

```bash
while true; do curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/api/info; sleep 0.2; done | uniq -c
```

Terminal 2 triggers a rollout:

```bash
kubectl rollout restart deploy/ttapp-backend deploy/ttapp-frontend -n app
kubectl rollout status deploy/ttapp-backend -n app
```

Terminal 1 should show only `200`. What makes it work: readiness probes + `maxUnavailable: 0` + `preStop` sleep + Spring graceful shutdown + App Gateway connection draining. **Experiment:** remove the `preStop` block and repeat. You may see a few 502s, because App Gateway still sends traffic to a pod that is already shutting down.

### 16.2 HPA under load

```bash
kubectl get hpa -n app -w          # terminal 2

# terminal 1: 50 parallel clients
seq 1 30000 | xargs -P 50 -I{} curl -s -o /dev/null http://$APPGW_IP/api/todos
```

Watch `TARGETS` climb above 70% and `REPLICAS` go 2 → 3 → 4… Stop the load. After about 5 minutes (the scale-down stabilization window) it returns to 2.

```bash
kubectl top pods -n app
kubectl describe hpa ttapp-backend -n app      # Events: "New size: 4; reason: cpu resource utilization above target"
```

If CPU doesn't rise enough, lower the target to 30% (`--set backend.hpa.targetCPUUtilizationPercentage=30`) and redeploy.

### 16.3 Cluster Autoscaler

Create pods that **request** more CPU than the user pool has:

```bash
kubectl create deployment inflate --image=mcr.microsoft.com/oss/kubernetes/pause:3.6 --replicas=0
kubectl set resources deployment inflate --requests=cpu=500m,memory=256Mi
kubectl scale deployment inflate --replicas=8
kubectl get pods -l app=inflate          # some Pending
kubectl get events --sort-by=.lastTimestamp | grep -iE "TriggeredScaleUp|FailedScheduling" | tail
kubectl get nodes -w                     # a new "aks-user-..." node appears in 2-4 minutes
```

Clean up and watch it scale down (about 10 minutes of low usage):

```bash
kubectl delete deployment inflate
```

Look at the decisions in Log Analytics (diagnostic category `cluster-autoscaler`):

```kusto
AKSControlPlane | where Category == "cluster-autoscaler" | project TimeGenerated, Message | order by TimeGenerated desc | take 50
```

**HPA vs VPA vs Cluster Autoscaler:** HPA changes the **number of pods** from metrics. VPA changes **pod requests and limits** (it restarts pods to apply them). CA changes the **number of nodes** when pods can't be scheduled, or when nodes are under-used. HPA + CA is the usual pair. Don't let HPA and VPA both act on **CPU or memory for the same workload**, because they fight; VPA in "Off" (recommendation) mode is safe.

### 16.4 VPA recommendations

`k8s/vpa-frontend.yaml`:

```yaml
# VPA in recommendation-only mode: it suggests CPU/memory requests but changes nothing.
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: frontend-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ttapp-frontend
  updatePolicy:
    updateMode: "Off"
```

```bash
kubectl apply -f k8s/vpa-frontend.yaml
# wait a few minutes with some traffic, then:
kubectl describe vpa frontend-vpa -n app      # "Target" CPU/memory = recommended requests
```

Compare the recommendation with the `resources` in `values.yaml`. This is how you right-size requests, which feeds into cost optimization.

### 16.5 PodDisruptionBudget and node drain

```bash
NODE=$(kubectl get nodes -l agentpool=user -o jsonpath='{.items[0].metadata.name}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

With one user node, the evicted pods have nowhere to go. They go Pending, and the **Cluster Autoscaler adds a node**. The PDB (`minAvailable: 1`) makes the drain **wait**, evicting one pod at a time, so each tier keeps a running pod. Watch the "Cannot evict pod as it would violate the pod's disruption budget" messages. Then:

```bash
kubectl uncordon $NODE
```

This is exactly what happens during an AKS upgrade.

### 16.6 NetworkPolicy tests

```bash
# A random pod in "app" cannot reach the backend (default deny)
kubectl run np-test -n app --rm -it --restart=Never --image=busybox:1.36 \
  -- wget -qO- -T 3 http://ttapp-backend:8080/api/info
# → "download timed out"

# The same pod labelled app=frontend CAN (the policy allows frontend → backend)
kubectl run np-test2 -n app --rm -it --restart=Never --image=busybox:1.36 --labels app=frontend \
  -- wget -qO- -T 3 http://ttapp-backend:8080/api/info
# → {"pod":"ttapp-backend-...","version":"..."}

# The frontend cannot reach the internet (egress only to backend + DNS)
kubectl exec -n app deploy/ttapp-frontend -- wget -qO- -T 3 https://www.microsoft.com
# → times out
```

`kubectl get networkpolicy -n app` then `kubectl describe networkpolicy backend -n app` shows the rules. In an interview, draw the allowed arrows: AppGW subnet → frontend → backend → PostgreSQL.

### 16.7 Layer 4 internal load balancer (Service type LoadBalancer)

App Gateway is L7. Now build an **L4 internal** Azure Load Balancer to see the difference:

```bash
kubectl create deployment hello --image=nginxinc/nginx-unprivileged:1.31-alpine
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: hello-ilb
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"   # private IP from the VNet
spec:
  type: LoadBalancer
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
EOF
kubectl get svc hello-ilb -w           # EXTERNAL-IP becomes 10.1.4.x (not public)
curl -s http://<that IP> | head -5      # works from the jumpbox over peering
```

What happened: the cloud-controller-manager inside AKS created a **frontend IP + rule + health probe** on an internal Standard Load Balancer (`kubernetes-internal`, in the node resource group). Without the annotation it would get a **public** IP. If `EXTERNAL-IP` stays `<pending>`, the control-plane identity is missing **Network Contributor** on the subnet. (We granted it on the spoke VNet in `aks.tf`.)

```bash
kubectl delete svc hello-ilb && kubectl delete deployment hello
```

**Service types recap:** ClusterIP (internal virtual IP), NodePort (a port on every node, 30000–32767), LoadBalancer (Azure LB, public or internal), ExternalName (DNS CNAME), headless (`clusterIP: None`: DNS returns pod IPs directly, used by StatefulSets).

### 16.8 Helm rollback, history, and a specific chart version

```bash
helm history ttapp -n app                  # every revision, with chart version + status
helm get values ttapp -n app               # values used by the current revision
helm get manifest ttapp -n app | less      # exactly what was applied

helm rollback ttapp <REVISION> -n app --wait
helm history ttapp -n app                  # a NEW revision appears ("Rollback to N")
```

**Package the chart and store versions in ACR (OCI)**. This answers "how do you deploy one specific version out of three?"

```bash
cd ~/three-tier-aks
az acr login -n $ACR_NAME --expose-token --output tsv --query accessToken \
  | helm registry login $ACR_NAME.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin

helm package helm/three-tier-app                    # three-tier-app-0.1.0.tgz
helm push three-tier-app-0.1.0.tgz oci://$ACR_NAME.azurecr.io/helm
# bump version: 0.2.0 in Chart.yaml, package and push again; then:
helm upgrade --install ttapp oci://$ACR_NAME.azurecr.io/helm/three-tier-app --version 0.1.0 -n app --reuse-values
```

**Rollback strategy (interview answer):** application = `helm rollback` (or redeploy the previous Git SHA; images are immutable). Database = Flyway migrations are **forward-only and backward-compatible**: add columns, don't rename or drop in the same release (expand/contract). So the old app version still works against the new schema, and you roll back the app without touching the DB. For a destructive mistake, restore PostgreSQL to a point in time (16.10).

### 16.9 Secret rotation

Rotate the database password **without** downtime:

```bash
NEWPW="$(openssl rand -base64 18 | tr -d '/+=')Aa1!"
az postgres flexible-server update -g $RG -n $PG_NAME --admin-password "$NEWPW"
az keyvault secret set --vault-name $KV_NAME -n db-password --value "$NEWPW" -o none

# Within ~2 minutes the CSI driver updates the mounted file AND the synced Kubernetes Secret:
kubectl get secret ttapp-backend-secrets -n app -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

But the **environment variable inside running pods does not change**. Env vars are read once at start-up. Existing DB connections keep working; new connections would fail with the old password. So:

```bash
kubectl rollout restart deploy/ttapp-backend -n app      # rolling, zero downtime
```

**Interview answer — "will rotating a secret cause downtime?"** Not if you do it right. Either read the secret from the mounted **file** (the app re-reads it), or roll the pods after rotation (tools like Reloader automate this). For zero risk, use **two credentials**: create the new one, roll the pods, then remove the old one.

> The next `terraform apply` of the platform stack sets the password back to the one in state (drift). For this lab that is fine, because the stack is recreated anyway. In real life, either let Terraform own rotation, or use `ignore_changes`.

### 16.10 PostgreSQL backup and point-in-time restore

Flexible Server takes automatic backups (7 days retention, as set in Terraform).

```bash
az postgres flexible-server show -g $RG -n $PG_NAME --query "{earliestRestore:backup.earliestRestoreDate, retention:backup.backupRetentionDays}" -o table
# Restore to a NEW server (it never overwrites the original):
az postgres flexible-server restore -g $RG --name ${PG_NAME}-restored --source-server $PG_NAME \
  --restore-time "$(date -u -d '-15 min' +%Y-%m-%dT%H:%M:%SZ)"
# compare data, then delete it (it costs money):
az postgres flexible-server delete -g $RG -n ${PG_NAME}-restored --yes
```

> The restored server copies the source's network settings. You may need to add a private endpoint for it before you can connect from the jumpbox. Even just running the restore and seeing it succeed is a good interview answer.

### 16.11 Upgrade AKS (the full procedure)

**Before**:

```bash
az aks get-upgrades -g $RG -n $AKS_NAME -o table          # available target versions

# Are we calling deprecated APIs? (counter > 0 means yes)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Static check of what Helm renders (Pluto by Fairwinds)
curl -sL https://github.com/FairwindsOps/pluto/releases/download/v5.24.4/pluto_5.24.4_linux_amd64.tar.gz | tar xz pluto
helm template ttapp helm/three-tier-app --set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x | ./pluto detect -
./pluto detect-helm -o wide       # checks the deployed releases
```

Also check: the PDBs are not too strict (`minAvailable` equal to the replica count would block the drain forever); there is enough quota for surge nodes; and you've read the AKS release notes for breaking changes.

**Upgrade with Terraform** (control plane first, then node pools; nodes are surged and drained one by one, respecting PDBs):

```bash
# terminal 1: zero-downtime check from 16.1
# laptop: change kubernetes_version in infra/platform/dev.tfvars to the next minor version
terraform plan -var-file=dev.tfvars -out=tfplan    # in-place update of cluster + 2 node pools
terraform apply tfplan                              # 20-40 minutes
# jumpbox: watch nodes get replaced
kubectl get nodes -o wide -w
```

**Interview answer — challenges you can mention:** PDBs blocking the drain; deprecated APIs in Helm charts (fixed before upgrading, with Pluto); not enough IPs in the subnet for surge nodes with Azure CNI (that's why we sized it /22); add-on versions changing; a node image update rebooting nodes (`node_os_upgrade_channel`). Also, you can only upgrade **one minor version at a time**.

**Automate upgrades:** `automatic_upgrade_channel = "patch"` or `"stable"` + a **maintenance window**, with dev/QA upgraded first. The Terraform `kubernetes_version` must then be ignored or kept in sync.

---

## Part 17 — Break-and-fix lab

Each lab: **break → look at the symptoms → investigate → find the root cause → fix → write a 4-line RCA.** Do them one at a time, and make sure the app is healthy before the next one.

RCA template (write it in your notes):

```text
Impact:      what users saw, from when to when
Root cause:  the technical reason
Fix:         what restored service
Prevention:  what stops it happening again (alert, test, policy, automation)
```

### Lab 1 — ImagePullBackOff

```bash
# break
kubectl -n app set image deploy/ttapp-frontend frontend=$ACR_NAME.azurecr.io/ttapp-frontend:does-not-exist
# investigate
kubectl get pods -n app
kubectl describe pod -n app <new frontend pod> | sed -n '/Events/,$p'   # "manifest unknown" / "not found"
curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/            # still 200!
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

Notice: **no outage**. With `maxUnavailable: 0`, the old pods keep serving while the new one fails. Variant: a **401 Unauthorized** pull error means the kubelet identity lost **AcrPull**. Check with `az role assignment list --scope <acr id>`.

### Lab 2 — CrashLoopBackOff

```bash
# break: JVM cannot start with a 1 MB heap
kubectl -n app set env deploy/ttapp-backend JAVA_TOOL_OPTIONS="-Xmx1m"
# investigate
kubectl get pods -n app -w                                       # Error → CrashLoopBackOff, RESTARTS grows
kubectl logs -n app <crashing pod> --previous                     # "Too small maximum heap"
kubectl describe pod -n app <crashing pod> | grep -A5 "Last State"
# wait 5-10 min: the CrashLoopBackOff log alert sends you an EMAIL
# fix
kubectl rollout undo deploy/ttapp-backend -n app
```

`--previous` is the key flag: the current container has just restarted, so the error is in the **previous** container's log. CrashLoopBackOff causes in real life: bad config or env var, missing secret, the app can't reach a dependency at start-up, wrong command or entrypoint, failing liveness probe.

### Lab 3 — OOMKilled

```bash
# break: JVM grabs 400 MB immediately, the limit is 256 MB
kubectl -n app set resources deploy/ttapp-backend --limits=memory=256Mi
kubectl -n app set env deploy/ttapp-backend JAVA_TOOL_OPTIONS="-Xms400m -XX:+AlwaysPreTouch"
# investigate
kubectl get pods -n app -w
kubectl describe pod -n app <pod> | grep -A4 "Last State"         # Reason: OOMKilled, Exit Code: 137
kubectl top pods -n app
# fix (two changes were made, so roll back two revisions)
kubectl rollout history deploy/ttapp-backend -n app
kubectl rollout undo deploy/ttapp-backend -n app --to-revision=<the revision before the two changes>
```

Exit code **137** = 128 + 9 (SIGKILL from the kernel OOM killer). Real fixes: right-size the limit (VPA recommendation, Grafana memory graph), set the JVM heap as a **percentage** of the container limit (`MaxRAMPercentage`, as we do), find memory leaks (heap dump, the `jvm_memory_used_bytes` trend).

### Lab 4 — Pod stuck in Pending

```bash
# break: no node has 3 free CPUs (Standard_D2s_v5 = 2 vCPU)
kubectl -n app set resources deploy/ttapp-frontend --requests=cpu=3
# investigate
kubectl get pods -n app                                            # Pending
kubectl describe pod -n app <pending pod> | sed -n '/Events/,$p'   # "0/2 nodes are available: Insufficient cpu"
                                                                   # + "pod didn't trigger scale-up" (no node size fits)
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

Other Pending reasons to know: taints with no matching toleration (try a `nodeSelector` that matches only the system pool), a `nodeSelector` that matches no node, a PVC not bound, the max pods per node reached, or not enough IPs in the subnet (Azure CNI).

### Lab 5 — App Gateway returns 502 Bad Gateway

```bash
# break
kubectl -n app scale deploy/ttapp-frontend --replicas=0
# investigate
curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/            # 502
kubectl get endpointslices -n app -l kubernetes.io/service-name=ttapp-frontend   # no endpoints
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{ip:address, health:health}" -o table
# fix
kubectl -n app scale deploy/ttapp-frontend --replicas=2
```

**502/504 even though all pods are Running.** The usual causes: the App Gateway health probe fails (wrong path or port, or readiness fails); a NetworkPolicy or NSG blocks the App Gateway subnet → pod traffic; AGIC lost permissions so the backend pool is stale (check the AGIC pod logs: `kubectl logs -n kube-system -l app=ingress-appgw`); or the backend is too slow (504, request timeout).

### Lab 6 — NetworkPolicy blocks frontend → backend

```bash
# break: remove "frontend" from the backend policy's allowed sources
kubectl -n app patch networkpolicy backend --type=json -p='[{"op":"remove","path":"/spec/ingress/0/from/0"}]'
# investigate
curl -s -m 70 -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/api/info    # 504 after nginx's 60 s timeout
kubectl exec -n app deploy/ttapp-frontend -- wget -qO- -T 3 http://ttapp-backend:8080/actuator/health  # timed out
kubectl get networkpolicy backend -n app -o yaml
# fix: re-run the app-ci-cd workflow, or helm upgrade with the same values (Helm restores the policy)
```

Lesson: **pod Running + probes OK + "timeout" between pods = check NetworkPolicy first.**

### Lab 7 — PostgreSQL blocked by NSG

```bash
# break
az network nsg rule update -g $RG --nsg-name nsg-ttapp-dev-pe -n Allow-Postgres-From-AKS-And-Jumpbox --access Deny
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl get pods -n app                 # new backend pods 0/1 (readiness includes DB) → rollout stuck; old pods still serve
kubectl logs -n app <new backend pod> | grep -iE "connection|timeout" | head
kubectl run dbtest --rm -it --restart=Never --image=busybox:1.36 -- sh -c \
  "nslookup $PG_NAME.postgres.database.azure.com; nc -zv -w 3 $PG_NAME.postgres.database.azure.com 5432"
#   DNS → 10.1.1.x (fine)   nc → timed out (blocked)  ⇒ network, not DNS
az network nsg rule list -g $RG --nsg-name nsg-ttapp-dev-pe -o table
# fix
az network nsg rule update -g $RG --nsg-name nsg-ttapp-dev-pe -n Allow-Postgres-From-AKS-And-Jumpbox --access Allow
```

Then run **infra → foundation → plan**. Terraform shows **no drift**, because you put it back by hand. Break it again and run the plan **before** fixing: Terraform shows the drift and `apply` repairs it. That's your "how do you detect manual changes?" story.

**"App in AKS cannot connect to PostgreSQL" checklist:** DNS (private zone linked? resolves to 10.1.1.x?) → network (NSG, NetworkPolicy egress, `nc -zv`) → server (status, public access, firewall) → credentials (Key Vault secret, synced Secret, env var) → TLS (`sslmode=require`) → connection limits (max connections, Hikari pool).

### Lab 8 — Private DNS zone link removed

```bash
# break
az network private-dns link vnet delete -g $RG -z privatelink.postgres.database.azure.com -n link-postgres-spoke --yes
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- nslookup $PG_NAME.postgres.database.azure.com
#   → resolves to a PUBLIC IP now → connection refused (public access disabled)
# fix: Terraform recreates the link
#   GitHub: infra → foundation → apply   (or locally: terraform apply -var-file=dev.tfvars)
```

This is the classic "worked yesterday, now can't connect after someone changed DNS or networking" production issue.

### Lab 9 — Key Vault access removed (secrets can't mount)

```bash
# break
BACKEND_PID=$(az identity show -g $RG -n id-ttapp-dev-backend --query principalId -o tsv)
az role assignment delete --assignee $BACKEND_PID --role "Key Vault Secrets User" \
  --scope $(az keyvault show -n $KV_NAME --query id -o tsv)
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl get pods -n app                                 # new pod stuck in ContainerCreating
kubectl describe pod -n app <stuck pod> | sed -n '/Events/,$p'   # FailedMount ... 403 Forbidden ... does not have secrets get permission
# fix: infra → platform → apply (Terraform recreates the role assignment), then the pod mounts within ~2 min
```

### Lab 10 — Readiness probe misconfigured

```bash
# break
kubectl -n app patch deploy ttapp-frontend --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/wrong"}]'
# investigate
kubectl get pods -n app                                 # new pod Running but 0/1 READY, rollout stuck
kubectl describe pod -n app <new pod> | grep -i unhealthy   # Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl rollout status deploy/ttapp-frontend -n app --timeout=60s    # times out
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

In the pipeline, the same situation makes `helm upgrade --rollback-on-failure` time out and **roll back automatically**. That's your "automatic rollback" story.

### Lab 11 — Terraform state lock

```bash
# laptop, terminal 1
cd ~/three-tier-aks/infra/platform && terraform apply -var-file=dev.tfvars   # leave it at the "Enter a value" prompt
# laptop, terminal 2
terraform plan -var-file=dev.tfvars
#   Error acquiring the state lock ... Lock Info: ID, Path, Operation: OperationTypeApply, Who, Created
az storage blob show --account-name $TFSTATE_SA -c tfstate -n platform.tfstate --auth-mode login \
  --query "properties.lease" -o json      # state: leased
# terminal 1: answer "no" → lock released
```

`plan` **and** `apply` both take the lock (unless `-lock=false`), so the lock details appear in both. If a crashed run leaves a stale lock, use `terraform force-unlock <LOCK_ID>`, but only after you are **sure** nobody else is running.

### Lab 12 — Pipeline identity has no rights outside its namespace (RBAC)

```bash
# 1) See exactly what the app pipeline identity is allowed to do
az role assignment list --assignee $APP_PRINCIPAL_ID --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table
#   Azure Kubernetes Service RBAC Admin   .../managedClusters/aks-ttapp-dev/namespaces/app
#   Azure Kubernetes Service Cluster User Role   .../managedClusters/aks-ttapp-dev
#   AcrPush, Reader ...
```

2) Prove it. In `app-ci-cd.yml`, add a temporary step to `deploy-dev`, right after "Get kubeconfig":

```yaml
      - name: RBAC test (remove after)
        run: kubectl get pods -n kube-system
```

Push to `main` and approve the deploy. The step fails with **Forbidden**: `... cannot list resource "pods" in API group "" in the namespace "kube-system"`. Remove the step and push again.

Interview line: *"The CI/CD identity can only deploy into its own namespace. Cluster-wide changes need a human admin role, and every action is in the audit log (AKSAuditAdmin)."*

---

## Part 18 — Daily start/stop, cost and teardown

### 18.1 Start of a lab session

```bash
source ~/three-tier-env.sh
az vm start -g $RG -n vm-ttapp-dev-jumpbox                # runner comes online by itself
# GitHub → Actions → infra → Run workflow: stack=platform, action=apply   (approve) ~20 min
#   or from the laptop: cd infra/platform && terraform apply -var-file=dev.tfvars
# jumpbox: kubectl create namespace app   (new cluster each time)
# GitHub → Actions → app-ci-cd → Run workflow (main)        (approve)
# optional: Part 15.2 (Prometheus/Grafana), Part 13.1 (certificate)
```

### 18.2 End of a lab session (do not skip)

```bash
# GitHub → Actions → infra → Run workflow: stack=platform, action=destroy   (approve) ~10-15 min
az vm deallocate -g $RG -n vm-ttapp-dev-jumpbox           # "stopped" alone still bills compute; deallocate does not
```

Check that nothing expensive is left:

```bash
az resource list -g $RG --query "[].{type:type, name:name}" -o table
```

### 18.3 Rough cost (pay-as-you-go, USD; varies by region; check the Azure Pricing Calculator)

| Resource | While platform is up | Idle (platform destroyed, VM deallocated) |
|---|---|---|
| App Gateway WAF_v2 (fixed part) | ≈ $0.45/h | 0 |
| 2 × D2s_v5 AKS nodes (+ disks) | ≈ $0.20/h | 0 |
| PostgreSQL B1ms + 32 GB | ≈ $0.02/h | 0 |
| Public IPs, load balancer, private endpoints | ≈ $0.05/h | ≈ $0.01/h (KV endpoint, jumpbox IP) |
| Jumpbox B2s | ≈ $0.05/h when running | disk only |
| ACR Basic, Key Vault, DNS zones, Log Analytics (1 GB/day cap) | small | small |
| **Total** | **≈ $0.8/h** | **< $1/day** |

A 4-hour session costs about **$3–4**. Left running for a month, it would be **≈ $600**. Create a **budget alert**: Portal → your resource group → **Cost Management → Budgets → Add** (for example $30/month, email at 50/80/100%).

Interview line on cost optimization: *"Dev infrastructure is created on demand with Terraform and destroyed after use. The VM has auto-shutdown, Log Analytics has a daily cap, App Gateway autoscales from 0 capacity units, nodes are right-sized with VPA recommendations, and the cluster autoscaler removes idle nodes."*

### 18.4 Full teardown (when you are completely done)

```bash
# 1) platform: GitHub infra workflow destroy, or locally:
cd ~/three-tier-aks/infra/platform && terraform destroy -var-file=dev.tfvars
# 2) foundation
cd ../foundation && terraform destroy -var-file=dev.tfvars
# 3) bootstrap resources
az identity delete -g $RG -n id-gh-infra
az identity delete -g $RG -n id-gh-app
az storage account delete -g $RG -n $TFSTATE_SA --yes
# 4) GitHub: remove the self-hosted runner (Settings → Actions → Runners)
```

The Key Vault stays **soft-deleted for 7 days** (purging needs subscription rights) and is then removed automatically. Its name can't be reused until then.

---

## Part 19 — Interview questions → where you did it

| Question (from your interview notes) | Where in this lab |
|---|---|
| Explain three-tier architecture / how frontend connects to backend | Part 0, Part 6.3 |
| Which network plugin do you use? Which component gives a pod its IP? | Part 3.2, Part 8.3, Part 8.10 |
| What are network plugins used for? | Part 3.2, Part 16.6 |
| Have you created a private AKS cluster? How? | Part 3.4, Part 8.3, Part 8.10 |
| How does the pipeline authenticate to a private AKS? | Part 2.3, Part 5.3, Part 12 |
| How do you authenticate/connect to AKS? Local accounts? | Part 8.3, Part 8.10 |
| Private endpoint between the AKS backend and the database | Part 3.3, Part 8.4, Lab 7, Lab 8 |
| VNet + subnet + private endpoint: how does the request flow? | Part 0, Part 3.3 |
| Frontend, backend, DB in one VNet or three? | Part 3.1 (subnets + NSGs + NetworkPolicy instead of separate VNets) |
| Overlapping CIDR / address issues | Part 3.1 |
| VNet-to-VNet connectivity | Part 4.5 (peering), Part 5.2 |
| Types of load balancers; which have you used? L4 vs L7 | Part 3.2, Part 3.7, Part 16.7 |
| Health probes in a load balancer | Part 9.6, Part 16.7, Lab 5 |
| What is an Ingress controller? Main configuration of an Ingress | Part 3.2, Part 9.5 |
| Types of Services; how do you expose an app? | Part 16.7, Part 9 |
| Headless service, sidecar, probes | Part 9.6, Part 16.7 |
| Restrict users before requests reach the app (WAF) | Part 13.2 |
| HTTP to HTTPS for frontend and backend | Part 13.1 |
| Certificate expiry without impact | Part 13.1 note, Appendix C |
| How does AKS connect to Key Vault? How do secrets reach the pods? | Part 8.5, Part 9.3, Part 10.4 |
| Service principal vs managed identity; system vs user-assigned | Part 2.3, Part 8.3 |
| Entra Workload ID with a user-assigned identity (detailed steps) | Part 8.5, Part 9.3 |
| If a Secret changes while pods run, what happens? Downtime on rotation? | Part 16.9 |
| Secrets in Docker images / in pipelines | Part 7, Part 2.3 (OIDC), Part 12 |
| Write a sample deployment.yaml | Part 9.3 |
| Multistage Dockerfile, CMD vs ENTRYPOINT, reduce build time | Part 7 |
| SonarQube checks and quality gates | Part 11 |
| How Trivy scans images / what scans in the build stage | Part 7.5, Part 12 |
| DevSecOps for frontend, backend, database | Parts 7, 11, 12, 13 |
| Commercial tools instead of SonarQube/Trivy | Appendix C |
| Write a pipeline for this stack | Part 12.1 |
| Ways a GitHub Actions workflow can run | Part 12.3 |
| Manual approval in the pipeline | Part 2.4 (environment `dev`) |
| Hosted vs self-hosted runners/agents | Part 5.3 |
| Store images in ACR; promote the same image | Part 10.1, Part 12, Appendix B |
| Helm commands for deploy and rollback; deploy a specific version | Part 16.8 |
| Rollback strategy (including database) | Part 16.8 |
| Deployment strategy / zero-downtime | Part 16.1, Part 9.6 |
| HPA, VPA, Cluster Autoscaler; real scenarios; HPA + VPA together | Parts 16.2–16.4 |
| PDB and upgrades; AKS upgrade + challenges; deprecated APIs | Parts 16.5, 16.11 |
| Pod not scheduled / Pending | Lab 4 |
| CrashLoopBackOff | Lab 2 |
| OOMKilled | Lab 3 |
| ImagePullBackOff | Lab 1 |
| 502/504 with all pods Running | Lab 5, Lab 6 |
| App can't connect to PostgreSQL | Lab 7, Lab 8 |
| 10% of traffic failing: investigate | Part 15.7 |
| Integrate Prometheus and Grafana with AKS | Part 15 |
| AKS logs to Azure Monitor; Azure Monitor vs App Insights vs Log Analytics | Part 14 |
| Terraform authentication to Azure | Part 2.3, Part 12.2 |
| Remote state, locking, lock details in plan or apply | Part 2.1, Lab 11 |
| One state file shared by many people | Part 12.5 |
| Modules / multiple environments | Part 0 (two stacks), Appendix B |
| Terraform drift / manual changes | Lab 7, Lab 8 |
| Secure the Terraform state file | Part 2.1 |
| PostgreSQL backup and restore | Part 16.10 |
| Provide RCA | Part 17 (RCA template) |
| Cost optimization in Azure | Part 18.3 |

---

## Appendix A — Blockers and the easy way around

| Blocker | Symptom | Easy way around |
|---|---|---|
| Resource provider not registered | `MissingSubscriptionRegistration` | Ask the admin to run `az provider register -n <name>` once. For `Microsoft.DevTestLab`, just delete the VM shutdown schedule resource |
| Can't read provider status | `cannot read` in Part 1.4 | Continue; only act if an apply fails |
| vCPU quota | `QuotaExceeded` / `OperationNotAllowed` | Use smaller sizes (`Standard_B2s_v2`/`Standard_B4ms`), max 2 user nodes, or another region (the resource group region decides) |
| SKU not available in region | `SkuNotAvailable` for D2s_v5 or B1ms | Try `Standard_D2as_v5`, `Standard_D2s_v4`; for PostgreSQL `B_Standard_B2s`; or check `az vm list-skus -l $LOCATION --size Standard_D2 -o table` |
| Policy denies public IPs | jumpbox or App Gateway IP creation fails | Jumpbox: remove its public IP and use **`az aks command invoke`** for kubectl/helm (runs through the Azure API, no network path needed). Deploy job: GitHub-hosted runner + `az aks command invoke -g $RG -n $AKS_NAME --command "helm upgrade ..." --file helm`. App Gateway needs a public frontend IP; ask for a policy exemption for this RG |
| Policy requires tags | node resource group creation fails | Add the required tags to `var.tags`. If the policy targets the AKS-created node RG, ask the admin for an exemption |
| Role assignment refused | `AuthorizationFailed` on `roleAssignments/write` | Your User Access Administrator may have a **condition** that limits which roles you can assign. Ask which roles are allowed, or ask the admin to create those 3–4 assignments once |
| `az ad signed-in-user show` blocked | Graph permission error | Copy your Object ID from Portal → Microsoft Entra ID → Users → you |
| Key Vault 403 from laptop | `Forbidden ... client address not authorized` | Your public IP changed: `export TF_VAR_my_ip=$(curl -s ifconfig.me)` and re-apply the foundation |
| Need a private GitHub repo | SonarQube Cloud free is public-only; environment approvals on private repos need a paid plan | Run **SonarQube Community Build** on the jumpbox: `docker run -d --name sonarqube -p 9000:9000 sonarqube:community`, create a token at `http://localhost:9000` (via SSH tunnel), run the Sonar steps on the self-hosted runner with `-Dsonar.host.url=http://localhost:9000`. Skip environment approvals or use `workflow_dispatch` as the "approval" |
| Self-hosted runners not allowed | | GitHub-hosted runner + `az aks command invoke` for deploy; Terraform from your laptop (Key Vault firewall allows your IP) |
| App Gateway takes too long or costs too much | | Use NGINX Ingress with an internal LB + `kubectl port-forward` from the jumpbox. You lose the WAF part, but keep everything else |
| AGIC pod not running | Ingress has no ADDRESS | `kubectl get pods -n kube-system -l app=ingress-appgw`; check the user node pool exists and the AGIC role assignments (Part 8.3) |

---

## Appendix B — Adding QA and Prod later

1. **State**: same code, a new state key per environment (`key = "qa/platform.tfstate"`) and `qa.tfvars` / `prod.tfvars` (different CIDRs, SKUs, node counts). Production: a separate subscription or resource group, its own VNet ranges that don't overlap dev.
2. **Modules**: move `foundation` and `platform` into `infra/modules/`, and create thin `infra/envs/{dev,qa,prod}` roots that call them with different variables.
3. **GitHub environments** `qa` and `prod`, each with its own approvers, branch rules and variables (`AKS_NAME`, `RESOURCE_GROUP`), plus a federated credential per environment (`environment:qa`, `environment:prod`) on separate identities.
4. **Promote, don't rebuild**: deploy the same image SHA to QA, then Prod. If Prod has its own ACR, copy the image: `az acr import -n <prodacr> --source <devacr>.azurecr.io/ttapp-backend:<sha> --image ttapp-backend:<sha>`. Better still, deploy by **digest** (`image@sha256:...`).
5. **Pipeline**: `deploy-dev → deploy-qa (auto tests) → deploy-prod (approval)` as dependent jobs; the same Helm chart with `values-<env>.yaml`.

---

## Appendix C — Optional upgrades

| Upgrade | What you learn |
|---|---|
| **ACR Premium + private endpoint** (`privatelink.azurecr.io`) | Fully private registry; builds must then also run on the self-hosted runner |
| **NAT Gateway** (`outbound_type = "userAssignedNATGateway"`) | Fixed egress IP and more SNAT ports |
| **Azure Firewall + route table** (`outbound_type = "userDefinedRouting"`) | Central egress filtering (FQDN rules for AKS required endpoints), hub-spoke with forced tunnelling |
| **Azure Bastion** in the hub | No public IP on the jumpbox (needs `AzureBastionSubnet`) |
| **Azure Monitor managed Prometheus + Azure Managed Grafana** | Managed alternative to Part 15 (needs `Microsoft.Monitor` and `Microsoft.Dashboard` providers) |
| **Key Vault certificate on App Gateway** (user-assigned identity on App Gateway + `ssl_certificate { key_vault_secret_id }`) | Automatic certificate renewal, no expiry outages |
| **cert-manager + Let's Encrypt** (with a real domain) | Free, auto-renewed public certificates |
| **Argo CD (GitOps)** | Pull-based deploy: the cluster syncs from Git (see your `azure-kubernetes-service/project2-helm-argocd-gitops` repo folder) |
| **Cilium** (`network_data_plane = "cilium"`, overlay) | eBPF dataplane, L7 network policies, Hubble flow logs |
| **Microsoft Defender for Containers / Azure Policy for AKS** | Runtime threat detection; enforce "no privileged pods", "only ACR images" |
| **Commercial scanners** | Snyk, Checkmarx, Veracode, Prisma Cloud, Aqua (commercial Trivy), Mend, GitHub Advanced Security: they replace or extend SonarQube/Trivy/Gitleaks |
| **Terraform modules + Terratest/`terraform test`** | Reusable, tested infrastructure code |

---

## Appendix D — Command cheat sheet

```bash
# ---------- Azure / AKS ----------
az aks get-credentials -g $RG -n $AKS_NAME --overwrite-existing && kubelogin convert-kubeconfig -l azurecli
az aks show -g $RG -n $AKS_NAME -o table
az aks get-upgrades -g $RG -n $AKS_NAME -o table
az aks nodepool list -g $RG --cluster-name $AKS_NAME -o table
az aks command invoke -g $RG -n $AKS_NAME --command "kubectl get pods -A"     # private cluster without network access

# ---------- kubectl ----------
kubectl get pods -n app -o wide -w
kubectl describe pod <pod> -n app                 # Events at the bottom = the answer, most of the time
kubectl logs <pod> -n app --previous              # logs of the crashed container
kubectl get events -n app --sort-by=.lastTimestamp
kubectl top pods -n app ; kubectl top nodes
kubectl rollout status|history|undo|restart deploy/<name> -n app
kubectl get endpointslices -n app
kubectl auth can-i --list -n app
kubectl debug -n app -it <pod> --image=busybox:1.36 --target=<container> --custom=/tmp/debug.json -- sh

# ---------- Helm ----------
helm upgrade --install <rel> <chart> -n app --rollback-on-failure --wait --timeout 10m -f values.yaml --set k=v
helm list -n app ; helm history <rel> -n app ; helm get values <rel> -n app ; helm get manifest <rel> -n app
helm rollback <rel> <rev> -n app --wait
helm template <rel> <chart> --set k=v | less ; helm lint <chart>

# ---------- Terraform ----------
terraform init -backend-config=backend.hcl
terraform fmt -recursive && terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan && terraform apply tfplan
terraform state list ; terraform state show <addr>
terraform import <addr> <azure id> ; terraform force-unlock <lock id>
terraform destroy -var-file=dev.tfvars

# ---------- Networking checks ----------
nslookup <name>                                   # private (10.x) or public answer?
nc -zv -w 3 <host> <port>
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME -o json
az network nsg rule list -g $RG --nsg-name <nsg> -o table
az network private-dns link vnet list -g $RG -z <zone> -o table
az network private-endpoint list -g $RG -o table
```

> When a topic from this lab comes up in an interview, note it in the interview tracker (`interview-questions` repo → `real-interview-questions/README.md`) with "did it hands-on: my-projects/handson Part N".
