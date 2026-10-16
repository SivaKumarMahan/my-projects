# 02 — Infrastructure as Code with Bicep Deployment Stacks

All Azure resources for the WMS platform are defined in Bicep and deployed as **Deployment Stacks** through Azure DevOps release pipelines.

## Repository layout

```
azure/bicep/
├── stacks/wms/<STACK_NAME>/main.bicep                 # one stack per component
├── parameters/wms/<ENVIRONMENT>/<STACK_NAME>/params.bicepparam
└── modules/...                                        # shared modules
helm/                                                  # Helm charts (wms, RBAC, rabbitmq)
```

Stacks: `aks`, `aks-acr`, `aks-network`, `log-analytics-workspace`, `keyvault`, `application-insights`, `postgres-db`, `rabbitmq` (container app), `iam` (managed identity + custom role), `rg`, `role`, `identity`.

## Deployment flow

1. User triggers the Azure DevOps release pipeline. The Azure repo holds Bicep code organised in stacks.
2. User deploys a stage. The stage runs an **Azure CLI task**.
3. The CLI task runs `az stack group create`, which deploys the Bicep stack into the target resource group.
4. Resources are provisioned in the resource group and are tracked by the stack.

```
Actor → Release pipeline (stage per stack) → Azure CLI task → az stack group create → Resource Group
```

## Stack prerequisites (deployment order)

| Stack | Prerequisite |
|---|---|
| AKS-ACR | none |
| Log Analytics Workspace | none |
| Key Vault | none |
| AKS Network | none |
| Application Insights | Log Analytics Workspace |
| RabbitMQ Container App | ACR |
| AKS | VNet, ACR, Log Analytics Workspace |
| Postgres DB | AKS Network |

## Step-by-step setup

### 1. IAM stack first (one per environment)

The IAM stack creates the **user-assigned managed identity** and the **custom role** that the pipeline will use. It is created once manually with admin rights:

```bash
az stack group create \
  --name sandbox-iam \
  --resource-group devops-sandbox \
  --template-file azure/bicep/stacks/wms/iam/iac-managed-identity/main.bicep \
  --parameters azure/bicep/parameters/wms/sandbox/iam/iac-managed-identity/params.bicepparam \
  --deny-settings-mode denyDelete \
  --action-on-unmanage deleteAll
```

### 2. One service connection per resource group

Create an Azure Resource Manager service connection from the IAM managed identity for each resource group (sandbox, performance, production…).

### 3. Variable group

Pipelines → Library → + Variable group → add variables → enable *Allow access to all pipelines*.

### 4. Task group for Bicep deployment

| Task | Type | Script |
|---|---|---|
| Check subscription | Azure CLI | `az account show` |
| Preview changes | Azure CLI | `az deployment group what-if ...` |
| Create stack | Azure CLI | `az stack group create ...` |

```bash
# what-if
az deployment group what-if \
  --name <ENVIRONMENT>-<STACK_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --template-file azure/bicep/stacks/wms/<STACK_NAME>/main.bicep \
  --parameters azure/bicep/parameters/wms/<ENVIRONMENT>/<STACK_NAME>/params.bicepparam

# create / update stack
az stack group create \
  --name <ENVIRONMENT>-<STACK_NAME> \
  --resource-group <RESOURCE_GROUP> \
  --template-file azure/bicep/stacks/wms/<STACK_NAME>/main.bicep \
  --parameters azure/bicep/parameters/wms/<ENVIRONMENT>/<STACK_NAME>/params.bicepparam \
  --deny-settings-mode none \
  --action-on-unmanage deleteAll
```

Short-flag form used on production AKS:

```bash
az stack group create -n wms-aks \
  --resource-group <PROD_RG> \
  --template-file azure/bicep/stacks/wms/aks/main.bicep \
  --parameters azure/bicep/parameters/wms/production/aks/params.bicepparam \
  --dm none --aou deleteResources
```

### 5. Release pipeline

- Pipelines → Releases → New → **Empty job**, stored under an `IaC/wms` folder.
- One release pipeline per environment, one **stage per stack** (AKS, Key Vault, Postgres…).
- Trigger: **Manual only**, with **pre-deployment approvals**.
- Artifact: Azure Repos, the IaC repository and branch.
- Agent: Azure Pipelines, `ubuntu-latest`.
- Task: Azure CLI, Shell, *Script path* → `create.sh` (wraps what-if + stack create).
- Link variable groups and scope them to stages.

### 6. Deploy and validate

Create release → choose commit → deploy stage → approve. Check the stage logs, then check **Resource group → Settings → Deployment stacks** in the portal to confirm managed resources.

## Bringing manually-created resources under IaC

1. Inventory manual resources per resource group — private endpoints/links for PostgreSQL, VNet peerings, the database itself.
2. Write Bicep: private endpoint in the `postgres-db` stack, VNet peering in the `aks-network` stack.
3. Reuse the same variable group and task group.
4. Deploy through the release pipeline and validate.

## Subscription-scope stacks (RG, role definition, identity)

```bash
# Resource group (subscription scope)
az deployment sub what-if --name <stack>-rg --location <location> \
  --template-file azure/bicep/stacks/wms/rg/main.bicep \
  --parameters azure/bicep/parameters/wms/production/rg/params.bicepparam
az stack sub create --name <stack>-rg --location <location> \
  --template-file azure/bicep/stacks/wms/rg/main.bicep \
  --parameters azure/bicep/parameters/wms/production/rg/params.bicepparam \
  --deny-settings-mode none --action-on-unmanage deleteAll
az stack sub show --name <stack>-rg --output table

# Role definition / managed identity (resource-group scope)
az stack group create --name <stack>-role --resource-group <rg> \
  --template-file azure/bicep/stacks/wms/role/main.bicep \
  --parameters azure/bicep/parameters/wms/production/role/params.bicepparam \
  --deny-settings-mode none --action-on-unmanage deleteAll
az stack group show --name <stack>-role --resource-group <rg> --output table
```

## `az deployment create` vs `az stack create`

| Aspect | Deployment | Stack |
|---|---|---|
| Purpose | One-time / incremental | Lifecycle-managed |
| Resource tracking | Not tracked afterwards | Continuously tracked |
| Cleanup on removal | Manual | Automatic (`--action-on-unmanage`) |
| State | Stateless | Stateful |
| Drift control | No | Yes (deny settings) |
| Best for | Application-level IaC | Platform / foundation IaC |

### Why Microsoft Graph resources fail in a stack

- **Deployment flow:** Bicep → ARM → ARM sees `Microsoft.Graph/applications@v1.0` → delegates to Microsoft Graph with your Entra permissions → app registration and service principal created → done. No ownership tracking, so it succeeds.
- **Stack flow:** Bicep → stack engine → ownership/lifecycle validation → ARM providers only. Graph resources do not support ARM ownership metadata or delete/detach semantics, so the stack blocks them before ARM reaches Graph. The error surfaces as *"Insufficient privileges to complete the operation"* even when Entra permissions are valid.
- **Takeaway:** Entra objects (groups, app registrations) are created outside stacks — manually, with a plain deployment, or with `az rest` against Graph.

## Native Bicep CLI tasks in Azure DevOps

| Task | Purpose |
|---|---|
| Install Bicep CLI | Installs latest Bicep CLI on the agent |
| Run Bicep CLI build | `.bicep` → ARM JSON (default dir, directory, single file or stdout) |
| Run Bicep CLI decompile | ARM JSON → `.bicep` |

Evaluated and **not adopted**: the tasks only build/decompile; the stack-based flow (what-if + `az stack group create` through scripts) already covers compilation.

## Terraform vs Bicep (decision record)

| Feature | Terraform | Bicep |
|---|---|---|
| Cloud support | Multi-cloud | Azure only |
| State | `terraform.tfstate` required | No state file |
| Modularity | Mature module ecosystem (Registry) | Modules, less mature |
| New Azure features | Can lag | Day-zero support |
| Community | Large (since 2014) | Smaller (since 2020) |
| Secrets | Vault, Key Vault | Key Vault |
| Integration | Azure, AWS, GCP, on-prem | Deep Azure CLI / DevOps integration |
| Pricing | Free → Enterprise | Free |

**Choose Terraform** for multi-cloud, stronger state/drift detection, heavy module reuse, or an existing Terraform team.
**Choose Bicep** when fully on Azure, wanting new features immediately, simpler syntax and no state file — **this project chose Bicep**, combined with Deployment Stacks to get lifecycle tracking and drift protection.

## Generic release script for all stacks

**Files:** `azure/bicep/releases/scripts/wms/global/script.sh`, `azure/bicep/stacks/wms/stackCommmands.txt`

One script serves every stack and environment. Inputs come from Azure DevOps variable groups: `ENVIRONMENT`, `RESOURCE_GROUP`, `STACK_NAME`, `DENY_SETTINGS_MODE`, `ACTION_ON_UNMANAGED`.

- **Convention-based paths:** `stacks/wms/$STACK_NAME/main.bicep` and `parameters/wms/$ENVIRONMENT/$STACK_NAME/params.bicepparam`.
- **Flags:**

| Flag | Action |
|---|---|
| `-bw` | `az deployment group what-if` (plan) |
| `-bc` | `az stack group create` (apply) |
| `-de` | `az stack group delete` |

- `bicep_fix()` reinstalls the Bicep CLI on hosted agents (`az config set bicep.use_binary_from_path=false`) to work around version mismatches between the agent image and the templates.

```bash
case "$1" in
  -bw) az deployment group what-if -g "$RESOURCE_GROUP" -n "$ENVIRONMENT-$STACK_NAME" \
         -f "stacks/wms/$STACK_NAME/main.bicep" -p "parameters/wms/$ENVIRONMENT/$STACK_NAME/params.bicepparam" ;;
  -bc) az stack group create -g "$RESOURCE_GROUP" -n "$ENVIRONMENT-$STACK_NAME" \
         -f "stacks/wms/$STACK_NAME/main.bicep" -p "parameters/wms/$ENVIRONMENT/$STACK_NAME/params.bicepparam" \
         --deny-settings-mode "$DENY_SETTINGS_MODE" --action-on-unmanage "$ACTION_ON_UNMANAGED" --yes ;;
  -de) az stack group delete -g "$RESOURCE_GROUP" -n "$ENVIRONMENT-$STACK_NAME" \
         --action-on-unmanage "$ACTION_ON_UNMANAGED" --yes ;;
esac
```

### Why Deployment Stacks instead of `az deployment group create`

- A stack **tracks the resources it manages**. With `--action-on-unmanage deleteAll`, a resource removed from code is **deleted from Azure** — drift cleanup similar to Terraform state.
- `--deny-settings-mode denyDelete` **blocks portal and manual deletion** of stack-managed resources — a guardrail.
- `what-if` gives a Terraform-plan-like preview for PR reviews and approval gates.

| Option | Values |
|---|---|
| `--deny-settings-mode` | `none`, `denyDelete`, `denyWriteAndDelete` |
| `--action-on-unmanage` | `detachAll`, `deleteResources`, `deleteAll` |
