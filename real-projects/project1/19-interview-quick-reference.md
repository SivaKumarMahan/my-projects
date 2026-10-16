# 19 — Interview Quick Reference

## Resume claim → where the evidence is

| Claim | Evidence in this project |
|---|---|
| Reusable, modular Bicep with dynamic parameterization, no per-environment hard-coding | [18](18-bicep-engineering-practices.md): typed params, `modules/`, `extends base.bicepparam`, 5+ environments |
| Zero-downtime Kubernetes upgrades, node pools rolled out in a controlled way | [06](06-aks-version-upgrades.md): sequential node pool `dependsOn`, pinned `aks_version`, blue-green pools; [09](09-helm-and-kubernetes-release-management.md): `maxUnavailable: 0` |
| Helm charts, standardised container delivery, ACR | [09](09-helm-and-kubernetes-release-management.md); AcrPull automation in [06](06-aks-version-upgrades.md) |
| Helm-based RBAC automation, standardised Kubernetes access | [07](07-aks-rbac-and-authentication.md): RBAC chart; [15](15-pre-commit-hooks-and-shift-left-validation.md): `values.schema.json` |
| Secure networking: VNets, private DNS, peering | [17](17-networking-and-postgresql-iac.md), [01](01-architecture-and-environments.md) |
| RBAC, managed identities, Key Vault, custom roles automated with Bicep | [05](05-identity-custom-roles-service-connections.md), [16](16-keyvault-cross-region-secret-sync.md) |
| Azure DevOps service connection creation automated with Bicep | [05](05-identity-custom-roles-service-connections.md) |
| Azure Functions, event-driven processing, Storage | [16](16-keyvault-cross-region-secret-sync.md) |
| Azure Monitor, Log Analytics, App Insights, KQL alerting | [16](16-keyvault-cross-region-secret-sync.md), Container Insights in [06](06-aks-version-upgrades.md) |
| DevSecOps: Checkov, quality gates, policy as code | [15](15-pre-commit-hooks-and-shift-left-validation.md), linter in [18](18-bicep-engineering-practices.md) |
| Shell / Python automation | [05](05-identity-custom-roles-service-connections.md), [02](02-iac-bicep-deployment-stacks.md), [09](09-helm-and-kubernetes-release-management.md), Python hooks in [15](15-pre-commit-hooks-and-shift-left-validation.md), Python function in [16](16-keyvault-cross-region-secret-sync.md) |
| Git branching, PR reviews | [15](15-pre-commit-hooks-and-shift-left-validation.md), [10](10-application-build-and-release-flow.md) |
| PostgreSQL provisioning with secure connectivity | [17](17-networking-and-postgresql-iac.md), [12](12-rabbitmq-database-and-global-load-balancing.md) |
| Production troubleshooting | [11](11-production-incidents-and-troubleshooting.md), [13](13-webapp-deployment-slots.md) |

> **Gap check:** the resume also mentions **Go** for Helm RBAC automation, **Trivy** and **SonarQube**. None of these appear in this project's IaC repo. Here the RBAC automation is Helm templates plus shell, and scanning is Checkov plus kubeconform. Either have that other work ready to describe, or describe this work as "Helm templates with Checkov/kubeconform".

## Quick-fire Q&A

**Bicep vs Terraform?**
Bicep is Azure-native with day-zero support for new APIs and no state file (ARM is the source of truth). Deployment Stacks add Terraform-like managed-resource tracking and deletion. Terraform is multi-cloud with a large provider ecosystem, but you manage remote state and locking.

**What is `what-if`?**
An ARM preview of create / modify / delete changes — the equivalent of `terraform plan`. Used as a PR or approval gate.

**`--deny-settings-mode` options?**
`none`, `denyDelete`, `denyWriteAndDelete`. They protect stack-managed resources from out-of-band changes.

**`--action-on-unmanage` options?**
`detachAll` (leave resources in place), `deleteResources`, `deleteAll` (resources and resource groups) for anything removed from the template.

**How do secrets flow through the IaC?**
PAT from an environment variable at deploy time (`readEnvironmentVariable`); DB password via `getSecret()` from Key Vault into a `@secure()` param; pods get secrets through the CSI `SecretProviderClass`; the Function uses managed identity with `DefaultAzureCredential`; linter rules block secrets in outputs.

**How do you make role assignments idempotent?**
Deterministic names: `guid(scope, roleDefinitionId, principalId)`.

**How do you validate Helm charts before deploying?**
`values.schema.json` → `helm template` → kubeconform (API schema) → Checkov (security), run in pre-commit across every environment × service in `helm/charts.yaml`.

**How does Azure DevOps authenticate to Azure without secrets?**
Workload Identity Federation. Azure DevOps issues an OIDC token whose subject identifies the service connection; Entra trusts it through a federated credential on the UAMI and returns an access token.

**Event Grid delivery guarantees?**
At-least-once, with retries and exponential backoff, then dead-lettering. Handlers must be idempotent.

**How do you upgrade AKS with zero downtime?**
Control plane first, then node pools one at a time (surge nodes, cordon/drain, or blue-green pools). PodDisruptionBudgets and `maxUnavailable: 0` keep replicas serving.

**How do pods get Key Vault secrets without storing them in Kubernetes?**
Secrets Store CSI driver plus a `SecretProviderClass`, authenticated with the kubelet / workload managed identity.
