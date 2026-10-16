# 05 — Managed Identities, Custom Roles and Service Connections

Pipelines authenticate to Azure through **user-assigned managed identities (UAMI)** with **workload identity federation** — no client secrets. Permissions are narrowed with **custom roles**, and the scope of those roles was tested before use.

## Creating a service connection from a managed identity (federated credentials)

1. **Create the UAMI** — Portal → Managed Identities → Create (subscription, RG, region, name). Note the **Client ID** and **Tenant ID**.
2. **Grant it access** — target resource / RG → Access control (IAM) → Add role assignment → e.g. Contributor or a custom role → assign to the identity.
3. **Create the service connection** — Azure DevOps → Project settings → Service connections → New → **Azure Resource Manager** → Identity type **Managed identity**:
   - Managed identity details: subscription, resource group, identity.
   - Azure scope: Subscription (optionally a resource group).
   - Name, and *Grant access permission to all pipelines*.

Azure DevOps creates the federated credential on the identity; the pipeline gets short-lived tokens with no stored secret.

> Note: creating an ARM service connection with RG scope also creates a **Contributor** role assignment for the identity on that RG in the background.

## Custom role scope testing

Bicep stacks created a storage account, role definition, UAMI and role assignment; a service connection was built from the UAMI and release pipeline stages tried different operations.

```bash
az deployment group what-if --name iam-scenario-1 --resource-group <rg> --template-file scenario1.bicep
az stack group create --name iam-scenario-1 --resource-group <rg> --template-file scenario1.bicep \
  --deny-settings-mode denyDelete --action-on-unmanage deleteAll
az role assignment list --assignee <client-or-object-id> --all --output table
```

### Scenario 1 — assignable scope = a single storage account

| Test | Result |
|---|---|
| Create new storage account in same RG | ❌ `AuthorizationFailed` on `Microsoft.Storage/storageAccounts/write` |
| Create Key Vault | ❌ `AuthorizationFailed` on `Microsoft.KeyVault/vaults/write` |
| Create storage account in a different RG | ❌ |
| Update existing storage account (enable blob soft delete, 7 days) | ✅ |

### Scenario 2 — wildcards in assignable scopes

Scopes like `.../providers/Microsoft.Storage/storageAccounts/*` or `.../providers/Microsoft.Storage`.
Result: stack failed with *"The role … is not available for assignment at the requested scope."* **Wildcards are not allowed in `assignableScopes`.**

### Scenario 3 — assignable scope = resource group

| Test | Result |
|---|---|
| Create storage account in RG | ✅ |
| Create Key Vault in RG | ✅ |
| Create storage account in another RG | ❌ |
| Update existing storage account | ✅ |

### Scenarios 4–5 — multiple specific resources (and via modules)

Scoped to one storage account and one key vault: creating **new** resources fails; updating **those** resources (soft delete, SKU change Standard → Premium) succeeds.

### Changing a role's scope

Narrowing scope fails with `RoleScopeBeingRemovedContainsAssignments` if assignments exist at the old scope. Delete them first:

```bash
az role assignment list --role <custom-role> --scope <old-scope>
az role assignment delete --role <custom-role> --scope <old-scope> --assignee <principal-id>
```

### Conclusion

- **Resource scope** — only actions on that exact resource; no new resources of any type in the RG; no side effects on other resources.
- **Resource group scope** — create and manage any resource in the RG; nothing outside it.
- The production IAM stack therefore uses **RG-scoped custom roles**, one identity + service connection per environment RG.

### Example custom role (deployment identity)

```json
{
  "roleName": "custom-role-sc",
  "description": "Custom role for Azure DevOps service connection to deploy infra resources",
  "assignableScopes": ["/subscriptions/<sub-id>/resourceGroups/<rg>"],
  "permissions": [{
    "actions": [
      "Microsoft.Resources/deployments/*",
      "Microsoft.Authorization/roleAssignments/read",
      "Microsoft.Authorization/roleAssignments/write",
      "Microsoft.ManagedIdentity/userAssignedIdentities/*",
      "Microsoft.Storage/storageAccounts/*",
      "Microsoft.KeyVault/vaults/*"
    ],
    "notActions": [], "dataActions": [], "notDataActions": []
  }]
}
```

## Automating service connection creation — what was tried

| Method | Approach | Outcome |
|---|---|---|
| 1. REST API + curl | `uami.bicep` (storage, KV, role, UAMI, assignment) → `serviceConnection.json` with `"scheme": "WorkloadIdentityFederation"` → `POST _apis/serviceendpoint/endpoints` with PAT → `fedCred.bicep` for federated credential | Service connection created, but pipeline auth failed — federated credential subject did not match |
| 2. `az devops service-endpoint create` | CLI with UAMI client id | `unrecognized arguments: --authorization-scheme WorkloadIdentityFederation` — CLI only supports SP + secret or SP + certificate |
| 3. REST API + shell loop | Loop over a list of names, create only if missing, then federated credential via Bicep | Pipeline failed with `AADSTS700016` (application not found in directory) — wrong issuer/subject pairing |
| 4. Replay portal API calls (Postman) | Captured the portal's network calls and replayed them | Some endpoints returned 404; federated credential not created |
| 5. **Bicep `deploymentScripts`** | Bicep deployment script runs a shell script that creates the endpoint via `az devops service-endpoint create --service-endpoint-configuration payload.json`, then returns `issuer` + `subject`; Bicep creates the federated identity credential from those outputs | ✅ Service connection created and validated |

### Working deploymentScript pattern

```bash
#!/bin/bash
set -x
exec > >(tee stdout.log) 2> >(tee stderr.log >&2)

az config set extension.use_dynamic_install=yes_without_prompt
az extension add --name azure-devops

SUBSCRIPTION_ID=$(az account list --query "[?isDefault].id" -o tsv)
SUBSCRIPTION_NAME=$(az account show --query name -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
ISSUER="https://login.microsoftonline.com/${TENANT_ID}/v2.0"

# AZDO_PAT is passed in securely at runtime (secure parameter / pipeline secret) — never hard-coded
[ -z "$AZDO_PAT" ] && { echo "AZDO_PAT not set"; exit 1; }
echo "$AZDO_PAT" | az devops login --organization "$ORG_URL"
PROJECT_ID=$(az devops project show --organization "$ORG_URL" --project "$PROJECT" --query id -o tsv)
az devops configure --defaults organization="$ORG_URL" project="$PROJECT"

SERVICE_PRINCIPAL_ID=$(az identity show --name "$UAMI_NAME" --resource-group "$UAMI_RG" --query clientId -o tsv)

cat <<EOF > payload.json
{
  "data": { "subscriptionId": "$SUBSCRIPTION_ID", "subscriptionName": "$SUBSCRIPTION_NAME",
            "environment": "AzureCloud", "scopeLevel": "Subscription", "creationMode": "Manual" },
  "name": "$SC_NAME", "type": "AzureRM", "url": "https://management.azure.com/",
  "authorization": { "parameters": { "tenantid": "$TENANT_ID", "serviceprincipalid": "$SERVICE_PRINCIPAL_ID" },
                     "scheme": "WorkloadIdentityFederation" },
  "isShared": false, "isReady": true,
  "serviceEndpointProjectReferences": [ { "projectReference": { "id": "$PROJECT_ID", "name": "$PROJECT" }, "name": "$SC_NAME" } ]
}
EOF

az devops service-endpoint create --service-endpoint-configuration ./payload.json
SC_ID=$(az devops service-endpoint list --query "[?name=='$SC_NAME'].id" -o tsv)
SUBJECT="/eid1/c/pub/t/<encoded-tenant>/a/<encoded-app>/sc/<org-id>/${SC_ID}"

# Outputs consumed by Bicep to create the federated identity credential
echo "{\"subject\": \"$SUBJECT\", \"issuer\": \"$ISSUER\"}" > "${AZ_SCRIPTS_OUTPUT_PATH:-/tmp/output.json}"
```

```bash
az deployment group what-if --name sandbox-sc --resource-group <rg> \
  --template-file azure/bicep/stacks/wms/serviceConnection/main.bicep \
  --parameters azure/bicep/parameters/wms/sandbox/serviceConnection/params.bicepparam
az stack group create --name sandbox-sc --resource-group <rg> \
  --template-file azure/bicep/stacks/wms/serviceConnection/main.bicep \
  --parameters azure/bicep/parameters/wms/sandbox/serviceConnection/params.bicepparam \
  --deny-settings-mode none --action-on-unmanage deleteAll --yes
az stack group delete --name sandbox-sc --resource-group <rg> --action-on-unmanage deleteAll --yes
```

### Debugging deploymentScripts

A deployment script runs in a container backed by a temporary storage account + file share. Logs live there:

```bash
KEY=$(az storage account keys list -g <rg> --account-name <script-sa> --query "[0].value" -o tsv)
az storage file list --account-name <script-sa> --share-name <share> --path azscriptinput --account-key "$KEY" -o table
az storage file download --account-name <script-sa> --share-name <share> \
  --path azscriptinput/stderr.log --account-key "$KEY" --dest ./stderr.log
```

The `azscriptinput/` folder holds `DeploymentScript.sh`, `userscript.sh`, `stdout.log`, `stderr.log`; `azscriptoutput/` holds outputs. A resource-scoped role on the identity caused `AuthorizationFailed` on `userAssignedIdentities/read` inside the script — the script identity needs RG-level read on the UAMI.

**Lessons:** pass the PAT as a secure runtime parameter; never print or commit it; rotate any token that ends up in logs.

## Kubernetes service connection types in Azure DevOps

| | KubeConfig | Service Account | Azure Subscription |
|---|---|---|---|
| Auth | kubeconfig with embedded creds | SA token | Entra token via ARM |
| Best for | Quick tests / PoC | Non-Azure clusters (EKS, GKE, on-prem) | **AKS (recommended)** |
| Granularity | Depends on kubeconfig user | Fine-grained Kubernetes RBAC | Azure RBAC + Entra |
| Token expiry | Certs/tokens may expire | Long-lived unless rotated | Auto-managed |
| Setup | Medium | Medium–high | Low (dropdowns) |
| Auto-rotate | No | No | Yes |

```bash
# KubeConfig
az aks get-credentials --resource-group <rg> --name <cluster> --file kubeconfig.yaml

# Service Account
kubectl create namespace test
kubectl create serviceaccount test-sa -n test
kubectl create clusterrolebinding test-sa-binding --clusterrole=cluster-admin --serviceaccount=test:test-sa
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
kubectl get secret test-sa-token -n test -o json
```

Each type was validated with a release stage running a **kubectl** task (`get pods -n default`). Creating these connections does not create extra Azure resources; pipeline and cross-project access is controlled from the connection's security page.

## Final implementation in the IaC repo

### A. Fully declarative: one Bicep stack

**Files:** `azure/bicep/stacks/wms/serviceConnection/main.bicep`, `azure/bicep/modules/serviceConnection/serviceConnection.bicep`, `azure/bicep/modules/serviceConnection/serviceConnection.sh`, `azure/bicep/modules/iam/identity/federatedCredential.bicep`, `azure/bicep/parameters/wms/sandbox/serviceConnection/params.bicepparam`

One `az stack group create` does all of this:

1. Creates a **user-assigned managed identity**.
2. Creates a **custom role** scoped to the resource group and **assigns** it to the identity.
3. Runs an **Azure Deployment Script** (`Microsoft.Resources/deploymentScripts`, AzureCLI kind) as that identity, which:
   - logs into Azure DevOps with a PAT (`az devops login`),
   - builds the payload (`scheme: WorkloadIdentityFederation`, `type: AzureRM`),
   - calls `az devops service-endpoint create`,
   - reads back the new service connection ID,
   - computes the federation **issuer** (`https://login.microsoftonline.com/<tenant>/v2.0`) and **subject** (`/eid1/c/pub/t/<orgHash>/a/<projectHash>/sc/<orgId>/<scId>`),
   - writes them to `$AZ_SCRIPTS_OUTPUT_PATH`.
4. The **federated identity credential** module takes `deploymentScripts.outputs.issuer` and `.subject` and binds Azure DevOps to the identity.

```bicep
// params.bicepparam — the PAT never goes into git
param azdoPat = readEnvironmentVariable('AZDO_PAT', '')

// main.bicep — chaining script outputs into the next module
module fedCred '../../../modules/iam/identity/federatedCredential.bicep' = {
  name: 'fed-cred'
  params: {
    identityName: uami.outputs.name
    issuer: serviceConnection.outputs.issuer
    subject: serviceConnection.outputs.subject
    audiences: ['api://AzureADTokenExchange']
  }
}
```

**Talking points**

- **No client secrets.** Azure DevOps gets a short-lived OIDC token and exchanges it for an Entra ID token. Nothing expires or needs rotating.
- **PAT only at deploy time,** read from an environment variable in the `.bicepparam` file.
- **Output chaining** removes the manual "copy issuer/subject from the portal" step.
- **Repeatable per environment** — only the param file changes (`commonParams.tags.environment`).

### B. Bulk creation through the REST API (shell)

**Files:** `azureDevOps/serviceConnection/serviceConnection.sh`, `serviceConnection.json`, `serviceConnectionNames.txt`, `README.md`

- Reads names from `serviceConnectionNames.txt`.
- For each name, **checks whether it already exists** (GET `_apis/serviceendpoint/endpoints` + `jq`) and skips it if so — idempotent.
- Looks up the identity `principalId` (`az identity show`), fills the JSON template with `envsubst`, and POSTs it to the REST API.
- Documented limitation: the REST API could not authenticate with federated credentials created outside the API. That is why approach A creates the connection first and the federated credential second.

### C. Multi-connection federated credentials

`azure/bicep/modules/federatedCredentials/federatedCredentials.bicep` loops `for i in range(0, length(serviceConnectionId))` and creates one federated credential per (identity, service connection) pair — a bulk version of step 4.

### Interview Q&A

**Service principal + secret vs managed identity + WIF?**
Secrets expire, leak and need rotation. WIF stores no secret and its tokens are short-lived.

**What are issuer, subject and audience?**
Issuer — who mints the token (Entra, for Azure DevOps). Subject — which service connection. Audience — `api://AzureADTokenExchange`.

**Why a Deployment Script?**
Bicep can't call the Azure DevOps API natively. A Deployment Script runs CLI code inside the ARM deployment and returns outputs.

## Custom roles and identities in Bicep

**Files:** `modules/iam/roles/definition.bicep`, `modules/iam/roles/assignment.bicep`, `modules/iam/identity/userAssigned.bicep`, stacks under `stacks/wms/iam/` and `stacks/wms/resourceGroup/`

- **Deterministic GUIDs** — `guid(subscriptionId, roleDefinitionName)` for definitions and `guid(rg, roleDefId, principalId)` for assignments, so redeploys are idempotent with no duplicate-assignment errors.
- **One assignment module, many scopes** — `resourceScope.resourceType` picks RG, ACR, Key Vault, Storage Account or Function App. Supports built-in role IDs and custom roles (`is_custom_role_definition`).
- **`iac-managed-identity` stack** — the identity behind the pipeline service connection, with a custom role limited to the providers IaC needs (AKS, ACR, VNet, PostgreSQL, Key Vault…) and `notActions` for subscription cancel/enable.
- **Sandbox access stacks** — a subscription-scope stack (`targetScope = 'subscription'`) creates a sandbox RG and a custom role assigned to individual engineers through a `for assignment in items(roleAssignments)` loop. Granting access becomes a pull request, not a portal click.
- **Separate identities** for cluster operations (`aks-managed-identity`) and RBAC automation (`aks-rbac-managed-identity`).

```bicep
resource assignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(resourceGroup().id, roleDefinitionId, principalId)
  properties: {
    roleDefinitionId: roleDefinitionId
    principalId: principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Talking point:** "We replaced broad Owner/Contributor grants with custom roles assigned in code. Every grant is reviewable, repeatable and auditable in git."
