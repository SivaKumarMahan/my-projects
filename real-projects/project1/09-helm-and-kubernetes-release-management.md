# 09 — Helm and Kubernetes Release Management

The application started with static manifests applied through `kubectl apply` with token replacement. It was moved to Helm for templating, versioned releases and rollback.

## Plan of action

### Release management with Helm

**Objectives:** replace `kubectl apply` with Helm; template and package resources; use Helm's release history, rollback and versioning; keep IaC consistency; deploy through Azure DevOps release pipelines.

1. **Chart creation** — inventory every manifest applied today; build charts per app type and infra component; template Deployments, Services, Ingress, HPA and ConfigMaps; per-environment values files plus a common values file to remove duplication.
2. **Testing** — deploy to sandbox; compare rendered manifests with the live ones; run rollback tests.
3. **CI/CD integration** — new pipelines using `helm upgrade --install`; task groups for repeated steps; environment and common variable groups scoped to stages; script-based Azure CLI tasks to run Helm.
4. **Documentation and KT** — Helm usage, configuration and rollback procedures; handover sessions for dev and ops.

### Resource management

**Objectives:** better scheduling, fewer resource conflicts, controlled pod placement, priorities for critical workloads, right-sized requests/limits.

1. **Affinity** — run application pods on dedicated user node pools (HA and isolation from system pods) using node affinity on pool labels.
2. **Taints and tolerations** — keep system-critical pods (e.g. NGINX ingress controller) on system pools; taint system nodes and add tolerations where needed.
3. **PriorityClasses** — high (mission-critical services), mid (standard services), low (batch, non-essential); assign one to every deployment.
4. **Requests / limits** — audit current usage, group apps into High / Medium / Low tiers, define requests/limits per tier, update every deployment.

## WMS Helm chart

```
wms/
├── Chart.yaml
├── values.yaml                     # base defaults (namespace: default)
├── .helmignore
├── templates/
│   ├── NOTES.txt
│   ├── resources/                  # thin wrappers
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── hpa.yaml
│   └── tpl/                        # reusable partials
│       ├── _deployment.tpl
│       ├── _service.tpl
│       ├── _ingress.tpl
│       ├── _hpa.tpl
│       └── _wms.tpl
└── values/
    ├── env/{sandbox,performance,production,qlab}/values.yaml
    └── wms/<MICROSERVICE>/values.yaml
```

- **resources/** files call the **tpl/** partials, so logic is written once and reused by every service.
- `values.yaml` groups settings under a `manifests` key: `manifests.deployment`, `manifests.service`, `manifests.ingress`.
- Templates read values with `{{ .Values.<key> }}`.
- `NOTES.txt` prints post-install hints; `.helmignore` excludes files from packaging.

### Services deployed by the chart

| Service | Resources |
|---|---|
| aimsplus-wms (main API) | Deployment, Service, Ingress |
| wms-conveyor-services | Deployment, Service, Ingress |
| wms-fulfillment-services | Deployment, Service, Ingress |
| wms-print-services | Deployment, Service, Ingress |
| wms-receipt-services | Deployment, Service, Ingress |
| rabbitmq | Deployment, Service, Ingress |
| wms-ui (frontend) | Deployment, Service, Ingress, HPA |

### Deploy command and value precedence

```bash
helm upgrade --install --kube-context=<CLUSTER> <RELEASE_NAME> ./wms \
  -f ./wms/values.yaml \
  -f ./wms/values/env/<ENVIRONMENT>/values.yaml \
  -f ./wms/values/wms/<MICROSERVICE>/values.yaml
```

Lowest → highest precedence: `values.yaml` → `env/<ENV>/values.yaml` → `wms/<MICROSERVICE>/values.yaml`.

A later step extracted Ingress and Service templates into a **Helm library chart** so other charts (RBAC, RabbitMQ) reuse the same definitions. A shell wrapper (`helm/releases/scripts/wms/global/script.sh`) is what the release pipeline calls.

## What the original manifests contained

**Backend `deployment.yml`** (one Deployment per service):

- Rolling update strategy; pod metadata, ports, environment variables
- **Secrets Store CSI driver** volume mounting secrets from Azure Key Vault
- `preStop` hook sleeping 15 s for graceful shutdown
- Liveness and readiness probes
- Startup command running FastAPI with Uvicorn (API) or Python consumer scripts, e.g. `python -m scripts.fulfillment_consumer`
- Resource requests/limits, `imagePullSecrets` for ACR
- `hostAliases` mapping IPs to hostnames inside the container
- **Init container** (`upgrade`) running DB migrations before the main container

**Backend `service.yml`** — ClusterIP Services per microservice, selectors on app labels.

**Backend HPA** — scale out when average memory per pod reaches 4 GiB.

**Frontend** — single Deployment on port 3000 (`npm run serve`), rolling updates, probes, `preStop` 15 s, CPU limit 333m, ACR pull secret, ClusterIP Service on 3000.

Templated placeholders replaced at release time: `__SERVICE_NAME__`, `__SERVICE_POSTFIX__` (e.g. `prod`), `__IMAGE_REPO__`, `__IMAGE_NAME__`, tags, plus app settings such as `__ENV__`, `__ROOT_PATH__`, `__RODB__`, `__RODBPORT__`, `__MAX_REPORT_QUERY_LIMIT__`, `__MAX_DOWNLOAD_LIMIT__` and third-party API keys/URLs.

## Kubernetes manifests vs Helm

| Feature | Manifests | Helm |
|---|---|---|
| Definition | One YAML per resource | Package of templates + values |
| Complexity | Repetitive for large apps | Bundles resources, templating |
| Templating | None | `values.yaml` |
| Reuse across envs | Separate files per env | One chart, many values files |
| Updates | Manual | `helm upgrade` |
| Versioning | Manual (Git) | Built-in release history |
| Scaling | Edit `replicas` | Change values + upgrade |
| CI/CD | Needs scripting | Pipeline-friendly |
| Rollback | Re-apply old YAML | `helm rollback` |
| Debugging | Easy, what you see is applied | Rendered output can be tricky (`helm template`) |
| Dependencies | kubectl only | Helm binary |
| Control | Fine-grained | Higher abstraction |

**Use manifests** for simple, static or one-off deployments, when you need precise control, or with GitOps tools (Argo CD / Flux) applying plain YAML.
**Use Helm** for many resources, frequent releases, the same app in several environments, reusing community charts (NGINX, PostgreSQL, Redis), and modular declarative management — the WMS case.

## Helm release automation script

**File:** `helm/releases/scripts/wms/global/script.sh`

| Flag | Action |
|---|---|
| `-tw` | `helm template` the **wms** app chart |
| `-uw` | `helm upgrade --install` the **wms** app chart |
| `-tr` | `helm template` the **RBAC** chart |
| `-ur` | `helm upgrade --install` the **RBAC** chart |

The release pipeline runs `helm template` first (rendered manifests appear in the logs as a dry run), then `helm upgrade --install` (idempotent). Values use the same three layers: base → environment → microservice / resource.

## Current wms chart settings

- One generic chart (Deployment, Service, Ingress through `tpl` partials) shared by multiple services (`rabbitmq`, `wms-ui`…) through per-service values files.
- **Zero-downtime rollout** — `RollingUpdate` with `maxSurge: 1`, `maxUnavailable: 0`.
- Liveness and readiness probes; CPU and memory requests and limits.
- **Graceful shutdown** — `preStop: sleep 15`, so the pod leaves the load balancer before it stops.
- Secrets through `secretKeyRef`, never in values; ACR image with `imagePullSecrets`.
- **Ingress** on AKS Web App Routing (`webapprouting.kubernetes.azure.com` ingress class) with a TLS secret and NGINX annotations for timeouts, body size and regex rewrites.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
lifecycle:
  preStop:
    exec: { command: ["sh", "-c", "sleep 15"] }
```

Charts are validated before commit with `values.schema.json`, `helm template`, kubeconform and Checkov — see [15](15-pre-commit-hooks-and-shift-left-validation.md).
