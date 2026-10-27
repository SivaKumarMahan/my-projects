# Part 1 — Prerequisites and permission checks

[← Part 0 — What you will build](00-what-you-will-build.md) | [Index](../README.md) | [Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities →](02-bootstrap-state-repo-oidc.md)

---

## 1.1 Tools on your laptop

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

## 1.2 One file with all your names

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

## 1.3 Log in and check your role

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

## 1.4 Resource providers (the only possible admin request)

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

## 1.5 Quota and policy (quick check)

```bash
# vCPU quota in your region (may fail without subscription read access; that is fine)
az vm list-usage -l "$LOCATION" -o table 2>/dev/null | grep -iE "Total Regional vCPUs|DSv5|BS Family"

# Azure Policy assignments that apply to your RG (deny rules can block public IPs, regions, SKUs)
az policy assignment list --resource-group "$RG" --query "[].displayName" -o tsv
```

The lab needs about **10 vCPUs** at peak: 1 system node and up to 3 user nodes, each `Standard_D2s_v5` (2 vCPU), plus a `Standard_B2s` jumpbox. If a policy denies **public IPs**, read [Appendix A](appendix-a-blockers.md) first.

---

[← Part 0 — What you will build](00-what-you-will-build.md) | [Index](../README.md) | [Part 2 — Bootstrap: state storage, GitHub repo, OIDC identities →](02-bootstrap-state-repo-oidc.md)
