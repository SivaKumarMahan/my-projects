# 10 — Application Build and Release Flow

## Repositories

| Repo | Contents |
|---|---|
| `wms` (backend) | Python services, `pipelines/prod.yml`, `manifests/deployment.yml`, `manifests/devdeployment.yml`, `manifests/service.yml`, HPA manifest |
| `wms-ui` (frontend) | React app, `pipelines/prod.yml`, `manifests/deployment.yml`, `manifests/service.yml` |
| `wms-iac` | Bicep stacks, parameters, Helm charts |

`main` is the production branch.

## End-to-end flow

```
Developer commit → main (or PR to main)
        │  auto trigger
        ▼
Build pipeline (YAML CI)
  ├─ Docker build
  ├─ Docker push → Prod ACR
  ├─ Docker push → DR ACR
  └─ Publish manifests (deployment.yml, service.yml) as pipeline artifact
        │
        ▼
Release pipeline (classic CD, manual trigger, approvals)
  ├─ Replace tokens (__…__ / {{…}}) from the variable group
  ├─ Deploy service.yml  (kubectl task, Kubernetes service connection)
  └─ kubectl apply deployment.yml  (imagePullSecrets for ACR)
        │
        ▼
AKS (default namespace) — rolling update, zero downtime
```

### Build pipelines

| Pipeline | Builds |
|---|---|
| wms-production | Backend image `aimsplus-wms` |
| wms-ui-production | Frontend image |

### Release pipelines

| App | Release | Variable group |
|---|---|---|
| Backend | wms - performance | Wms - Performance |
| Backend | wms - production | Wms - Production |
| Backend | wms - production - DR | Wms - Production - DR |
| Backend | wms-upgrade-sandbox / wms-upgrade-production | Upgrade (migration) job — see [11](11-production-incidents-and-troubleshooting.md) |
| Frontend | wms-ui-performance | wms-ui-performance |
| Frontend | wms-ui-production | wms-ui-production |
| Frontend | wms-ui-production - DR | wms-ui-production-DR (+ Cloudflare purge stage) |

One backend release deploys all five backend Deployments and Services from the shared manifest.

### Task groups

**Backend:** replace tokens in `deployment.yml` → replace tokens in `deployment.yml` + `service.yml` → deploy service (kubectl, Kubernetes service connection) → `kubectl apply` deployment (ACR credentials stored as Kubernetes secrets, referenced through `imagePullSecrets`; Azure subscription used for ACR integration). Backend variable group ≈ 60 variables.

**Frontend:** replace tag tokens → replace tokens → deploy app → deploy service. Frontend variable group ≈ 8 variables.

## Production Kubernetes objects

| Kind | Backend | Frontend |
|---|---|---|
| Deployments | aimsplus-wms-production, wms-conveyor-services-prod, wms-fulfillment-services-prod, wms-print-services-prod, wms-receipt-services-prod, wms-package-planning-prod | wms-ui-prod |
| Services | one per deployment | wms-ui-prod |
| Ingress | wms-prod-ingress | wms-ui-prod-ingress |
| HPA | wms-prod-autoscaler | — |
| CronJob | ship-fulfillment-sweep-prod | — |
| Secrets | image pull secrets, TLS secret | image pull secret |

Image settings per environment are injected as `IMAGEPULLSECRETNAME`, `IMAGE_NAME`, `IMAGE_REPO` (prod and DR/performance pull from different registries).

## Multi-region production release

A single release pipeline with two stages:

| Stage | Role | Original variable group |
|---|---|---|
| west2 | Primary region | Wms - Production |
| central | Failover region | Wms - Production - DR |

The variable groups were refactored into three to remove duplication:

| Variable group | Contents |
|---|---|
| wms prod multiregion - commons | Everything identical across regions (secrets like SMTP and shipping tokens; some intentionally empty strings such as `NAME`, `ROOT_PATH`) |
| wms prod multiregion - west2 | Region-specific values |
| wms prod multiregion - central | Region-specific values |

Only four variables actually differ by region: `APPLICATIONINSIGHTS_CONNECTION_STRING`, `RABBITMQ_HOST`, `RABBITMQ_PASSWORD`, `TRACKPAD_SERVICE_WSDL`. *Commons* is linked at release scope; each regional group is scoped to its stage.

## Repository housekeeping — stale branch cleanup

1. Clone the repo as a backup: `git clone <repo-url> && cd <repo>`.
2. List branches: `git branch -a`.
3. Agree the list of merged / obsolete feature branches with the team.
4. In Azure DevOps → Repos → **Branches**, **lock** the branches to keep (main, release, active features).
5. For each obsolete branch: ⋯ → **Delete branch** → confirm.
