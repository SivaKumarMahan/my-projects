# 04 — Reusable Bicep Modules via a Registry

Plan to build, version and publish Bicep modules to an OCI registry so every stack consumes the same tested building blocks instead of copying local files.

## Registry options evaluated

| Registry | Notes | When to consider |
|---|---|---|
| **Azure Container Registry (ACR)** | Native Bicep / Azure DevOps / GitHub Actions integration, Entra ID auth, Azure RBAC, private endpoints | **Recommended** for Azure-native setups |
| GitHub Packages (GHCR) | OCI artifacts; auth with GitHub PAT | Org is all-in on GitHub / Actions |
| GitLab Package Registry | OCI container registry per project/group; PAT, deploy or job tokens | Org is all-in on GitLab CI |
| JFrog Artifactory | Universal artifact manager; API key / scoped token; complex, premium | Enterprises needing one central artifact store |
| Public Bicep Registry (Azure Verified Modules) | Microsoft-maintained, referenced with `br/public:` | Generic modules; not customizable, may lag |
| Harbor (open source) | Full-featured, OCI-compliant, self-hosted | Self-hosted, cost-sensitive |
| Docker Distribution | Minimal registry, no UI | Very simple needs |

```bash
az bicep publish --file ./modules/storage.bicep --target "br:ghcr.io/<org>/<repo>/bicep/modules/storage:v1.0"
az bicep publish --file ./modules/storage.bicep --target "br:registry.gitlab.com/<group>/<project>/bicep/storage:v1.0"
az bicep publish --file ./modules/storage.bicep --target "br:<instance>.jfrog.io/bicep-modules/storage:v1.0"
```

### Approximate pricing

| Registry | Visibility | Price |
|---|---|---|
| ACR | Private only | Basic ~$5/mo (10 GB), Standard ~$20/mo (100 GB), Premium ~$40/mo (500 GB) |
| GHCR | Public & private | Free quota by plan (500 MB / 1 GB transfer on free orgs), then per-GB |
| GitLab | Public & private | Included; 5 GB per namespace on free tier |
| JFrog SaaS | Public & private | From ~$98/mo |
| Docker Hub | Public & private | Free (rate-limited) / Pro ~$5/mo |
| Harbor | Private | Free (infra cost) or hosted plans |
| AVM | Public | Free |

## How a Bicep module is stored (OCI manifest)

- **config** — custom Bicep media type, SHA-256 `digest`, `size: 0` (config is empty for modules).
- **layers** — exactly one layer: the compiled ARM JSON, with its own media type, digest and size.
- The digest lets the client verify the module was not corrupted or tampered with.

## Build and publish

```bash
# Validate by compiling to ARM JSON
az bicep build --file ./modules/network.bicep --outdir ./compiled

# Publish to ACR
az bicep publish \
  --file ./modules/network.bicep \
  --target br:myregistry.azurecr.io/bicep/modules/network:v2
```

Target format: `br:<login-server>/<repository-path>:<tag>`

- `br:` — Bicep registry scheme.
- `<login-server>` — `*.azurecr.io` (commercial), `*.azurecr.us` (government), `*.azurecr.cn` (China).
- repository path — created if missing; organise as `bicep/modules/<name>` or `bicep/types/<name>`.
- tag — module version; use immutable tags for repeatable deployments.

`az bicep build` takes one file at a time, so a small loop script is used to build and publish all modules.

## Consuming modules

```bicep
module stg 'br:myregistry.azurecr.io/bicep/modules/storage:v1' = {
  name: 'storage'
  params: { ... }
}
```

During `az deployment group create` / `az stack group create`, the CLI sees the `br:` reference, pulls the artifact into the local cache, and continues. No extra pipeline commands are needed.

## Pipeline integration options

1. **Inline publishing** — the deploy script runs `az bicep publish` before `az deployment/stack create`.
2. **Staged publishing** — a first stage in the release pipeline publishes modules; later stages (what-if, deploy) consume them.
3. **Decoupled publishing** — modules are independent artifacts with their own CI pipeline per module; infrastructure pipelines only consume versions.

Impact on the existing workflow: scripts are unchanged (still what-if + deploy against `main.bicep`); only module definitions move from local files to the registry.

## Permissions

| Identity | Role |
|---|---|
| Publisher pipeline (SP / managed identity) | **AcrPush** (push + pull, cannot delete registry) |
| Deploy pipeline | **AcrPull** |
| Team members | Assign AcrPull / AcrPush to **Entra ID security groups**, then manage access via group membership (least privilege, no per-user assignments) |
