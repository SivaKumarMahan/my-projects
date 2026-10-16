# Real Projects — Warehouse Management System (WMS) on Azure

Write-ups from a real DevOps engagement: a cloud-native, containerized Warehouse Management System running on Azure Kubernetes Service (AKS), with infrastructure written in Bicep and delivered through Azure DevOps pipelines.

## Project at a glance

| Area | What was used |
|---|---|
| Application | 1 frontend (React) + 6 backend microservices (Python / FastAPI) + RabbitMQ |
| Compute | AKS (multiple clusters across Central US, West US 2, North Central US), Azure Container Apps, App Service Web Apps |
| IaC | Azure Bicep with Deployment Stacks (`az stack group create`) |
| CI/CD | Azure DevOps — YAML build pipelines, classic release pipelines, task groups, variable groups |
| Packaging | Raw Kubernetes manifests → Helm charts |
| Data | Azure Database for PostgreSQL Flexible Server (primary + read replica) over private endpoints |
| Networking | VNet, subnets, NSGs, VNet peering, private DNS zones, VPN gateway (IPsec site-to-site to on-prem) |
| Security | Key Vault + Secrets Store CSI driver, Managed Identities, custom Azure roles, Entra ID + Kubernetes RBAC |
| Observability | Azure Monitor / Container Insights, Log Analytics, Managed Prometheus, Grafana, metric alert rules |
| Edge | Cloudflare (global load balancing, CDN caching, cache purge) |

## Documents

| # | File | Topic |
|---|---|---|
| 01 | [01-architecture-and-environments.md](01-architecture-and-environments.md) | Architecture, components, environments, resource inventory |
| 02 | [02-iac-bicep-deployment-stacks.md](02-iac-bicep-deployment-stacks.md) | Bicep stacks, release pipelines for IaC, stack vs deployment, Terraform vs Bicep |
| 03 | [03-bicep-testing-and-validation.md](03-bicep-testing-and-validation.md) | Bicep test matrix, AKS version tests through Bicep, results |
| 04 | [04-bicep-module-registry.md](04-bicep-module-registry.md) | Publishing reusable Bicep modules to a registry (ACR) |
| 05 | [05-identity-custom-roles-service-connections.md](05-identity-custom-roles-service-connections.md) | Managed identities, custom role scope testing, service connections, automation attempts |
| 06 | [06-aks-version-upgrades.md](06-aks-version-upgrades.md) | AKS upgrade strategies, runbooks, node pool blue-green, Bicep limitations |
| 07 | [07-aks-rbac-and-authentication.md](07-aks-rbac-and-authentication.md) | AKS auth modes, RBAC Helm chart, RBAC test cases, kubelogin token lifecycle |
| 08 | [08-aks-networking-models.md](08-aks-networking-models.md) | Kubenet vs Azure CNI (and CNI variants) |
| 09 | [09-helm-and-kubernetes-release-management.md](09-helm-and-kubernetes-release-management.md) | Helm chart design, manifests vs Helm, resource management plan |
| 10 | [10-application-build-and-release-flow.md](10-application-build-and-release-flow.md) | App CI/CD flow, Kubernetes objects, multi-region variable groups |
| 11 | [11-production-incidents-and-troubleshooting.md](11-production-incidents-and-troubleshooting.md) | Pod evictions / memory pressure, cleanup and scaling, init container decoupling |
| 12 | [12-rabbitmq-database-and-global-load-balancing.md](12-rabbitmq-database-and-global-load-balancing.md) | RabbitMQ to AKS, DB into AKS network, global load balancer |
| 13 | [13-webapp-deployment-slots.md](13-webapp-deployment-slots.md) | App Service slots, slot swap, sticky vs non-sticky settings |
| 14 | [14-cloudflare-cache-purge.md](14-cloudflare-cache-purge.md) | Cache purge task group and cache behaviour testing |
| 15 | [15-pre-commit-hooks-and-shift-left-validation.md](15-pre-commit-hooks-and-shift-left-validation.md) | Pre-commit hooks: Bicep lint, Helm render + kubeconform + Checkov, developer tooling |
| 16 | [16-keyvault-cross-region-secret-sync.md](16-keyvault-cross-region-secret-sync.md) | Event-driven Key Vault sync with Event Grid + Azure Function, KQL alerting as code |
| 17 | [17-networking-and-postgresql-iac.md](17-networking-and-postgresql-iac.md) | VNet/subnets, cross-RG peering, private DNS, PostgreSQL replica and virtual endpoints |
| 18 | [18-bicep-engineering-practices.md](18-bicep-engineering-practices.md) | Deployment Script module, typed params, param inheritance, linter, helpers |
| 19 | [19-interview-quick-reference.md](19-interview-quick-reference.md) | Resume claim → evidence map, gap check, quick-fire Q&A |
| 20 | [20-bicep-testcases-lab-walkthrough.md](20-bicep-testcases-lab-walkthrough.md) | Bicep test case lab: merged vs split templates, AKS version tests, commands, functional tests |
