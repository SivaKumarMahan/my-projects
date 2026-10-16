# 18 — Bicep Engineering Practices and the Deployment Script Module

## Reusable Deployment Script module

**File:** `azure/bicep/modules/deploymentScript/deploymentScript.bicep`

One module covers every "run Azure CLI during a deployment" need:

| Use | Covered in |
|---|---|
| Azure DevOps service connection creation | [05](05-identity-custom-roles-service-connections.md) |
| AKS bootstrap (SecretProviderClass) | [06](06-aks-version-upgrades.md) |
| Function App code deployment | [16](16-keyvault-cross-region-secret-sync.md) |

- Runs as a **user-assigned managed identity** — no secrets.
- Script content comes from `loadTextContent()`; inputs are environment variables.
- Outputs are read back safely: `contains(script.properties.outputs, 'issuer') ? script.properties.outputs.issuer : ''`.
- `cleanupPreference` and `retentionInterval` control the cost of the temporary container instance and storage account.
- A duplicated copy of the module was later consolidated into this single version.

**Debugging story:** `cleanupPreference` behaved differently across API versions of `Microsoft.Resources/deploymentScripts`. The fix and the behaviour are documented in the service connection stack README under *Limitations*.

## Strongly typed parameters

**Folder:** `azure/bicep/types/`

User-defined types with `@export()` and `import {}`. Every module takes a typed param object (`AksParams`, `RoleAssignmentParams`, `FederatedCredentialParams`…), so bad parameters fail at **compile time** instead of halfway through a deployment.

```bicep
// types/aks.bicep
@export()
type AksParams = {
  name: string
  aks_version: string
  enableRBAC: bool?
  disableLocalAccounts: bool?
  agentPools: AgentPool[]
}

// stacks/wms/aks/main.bicep
import { AksParams } from '../../../types/aks.bicep'
param aksParams AksParams
```

## Parameter inheritance

`.bicepparam` files use `extends '../base/base.bicepparam'`. Shared values live once; each environment overrides only what differs — no environment-specific hard-coding.

Environments: `sandbox`, `qlab-01`, `performance`, `production`, `production-west2`, plus integration-layer stacks that reuse the same modules.

## Helpers

- User-defined functions in `azure/bicep/utils/helpers.bicep`, e.g. `can_acr_be_attached()`, `is_value_valid()`.
- Tag helper `generateTags()` in `types/tags.bicep` so every resource gets consistent tags.

## Linter and scanning

- **`bicepconfig.json`** with ~30 analyzer rules, including `outputs-should-not-contain-secrets`, `secure-secrets-in-params`, `no-hardcoded-env-urls`, `use-recent-api-versions` (730 days).
- **`.checkov.yaml`** runs Checkov on the `arm` and `bicep` frameworks (also wired into pre-commit, see [15](15-pre-commit-hooks-and-shift-left-validation.md)).
- **Bicep test scaffold** (`tests/stacks/aks-network/test.bicep`) using the experimental `test` keyword.
- Every stack has a **README** with its what-if / create / delete commands and the environment → resource group map.

## Interview Q&A

**How do you stop bad parameters reaching Azure?**
User-defined types catch shape errors at compile time; the linter catches unsafe patterns; what-if shows the effect before apply.

**How do you avoid copy-pasting parameters per environment?**
A base `.bicepparam` with `extends`, and per-environment files that only override differences.

**Why a Deployment Script instead of a pipeline step?**
It keeps the CLI work inside the ARM deployment, so its outputs can feed later resources (e.g. issuer/subject into a federated credential) in the same stack.
