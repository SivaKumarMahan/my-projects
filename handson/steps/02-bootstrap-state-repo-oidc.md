# Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities

[← Part 1 — Prerequisites and permission checks](01-prerequisites-and-permissions.md) | [Index](../README.md) | [Part 3 — Networking explained (read before Terraform) →](03-networking-explained.md)

---

Terraform cannot store its state in a storage account that Terraform itself creates (chicken-and-egg), and the pipeline identities must exist before any pipeline runs. So these few things are created with the Azure CLI. This is also a good interview point: "the backend and pipeline identities are bootstrapped once; everything else is code".

## 2.1 Terraform state storage account

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

**State locking.** The azurerm backend locks the state with a **blob lease**. While `plan` or `apply` runs, the blob shows `leaseState: leased`. You will see this in [Part 17](17-break-and-fix-lab.md#lab-11--terraform-state-lock).

## 2.2 Create the GitHub repository

1. On GitHub, create a **public** repository named `three-tier-aks`. Public keeps SonarQube Cloud and environment approvals free. Never commit secrets; Gitleaks will check that for you.
2. Clone it:

```bash
cd ~ && git clone https://github.com/$GH_OWNER/$GH_REPO.git && cd $GH_REPO
mkdir -p .github/workflows backend frontend helm infra/foundation infra/platform k8s/monitoring
```

## 2.3 Two managed identities for GitHub Actions (OIDC)

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

## 2.4 GitHub settings

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
| `SONAR_ORG` | set in [Part 11](11-sonarqube-cloud.md) |
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

[← Part 1 — Prerequisites and permission checks](01-prerequisites-and-permissions.md) | [Index](../README.md) | [Part 3 — Networking explained (read before Terraform) →](03-networking-explained.md)
