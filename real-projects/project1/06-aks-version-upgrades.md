# 06 — AKS Kubernetes Version Upgrades

Upgrades were run across seven clusters (sandbox, qlab, performance, dev and three production clusters), moving through 1.28 → 1.29 → 1.30 → 1.31 → 1.32/1.33 as versions approached end of support.

## Upgrade history

| Wave | Clusters | From → To | Approach |
|---|---|---|---|
| 1 | Dev cluster | 1.28.9 → 1.29.0 | Portal |
| 2 (non-prod) | dev, performance | 1.29.0 → 1.30.12 → 1.31.9 | Two stages |
| 2 (non-prod) | qlab, sandbox | 1.30.0 → 1.31.9 | One stage |
| 2 (prod) | wmsproductionaks | 1.28.15 → 1.30.12 → 1.31.9 | Two stages |
| 2 (prod) | aimsplusprodaks | 1.30.0 → 1.31.9 | One stage |
| 2 (prod) | wmsproductioncentral | 1.29.0 → 1.30.12 → 1.31.9 | Two stages |
| 3 (non-prod) | dev, qlab, performance | 1.31.9 → 1.32.10 | 1.32 EOL approaching |
| 3 (sandbox) | sandboxaks | 1.32.x → 1.33.0 | Bicep tests |
| 4 (prod) | aimsplusprodaks, wmsproductioncentral | 1.31.9 → 1.33.10 | Control plane + blue-green node pools |
| 5 (prod) | wmsproductionaks | 1.31.9 → 1.33.10 | Control plane + blue-green node pools |

Production windows were booked in low-traffic hours (e.g. 05:00–08:00 EDT), one cluster per day, with an estimated 30–45 minutes per cluster.

## Standard upgrade process (communicated to stakeholders)

1. Upgrade the **control plane** first (in stages if crossing more than one minor version — stop at the last patch of each minor, e.g. 1.30.12 before 1.31).
2. Add **new node pools** at the control-plane version.
3. **Cordon** old node pools (or taint `NoSchedule`) so no new pods land there; existing pods keep running.
4. **Drain** old nodes so workloads reschedule onto new pools. Old pods are only terminated once replacements are healthy.
5. Developers run **sanity tests** against endpoints.
6. After sign-off, **delete** old node pools.

Pre-check: confirm no breaking API removals between the current and target versions (Kubernetes deprecation guide / AKS release notes).

## Runbook — portal

1. Portal → Kubernetes services → cluster → Overview (note version, network plugin, region).
2. Settings → **Upgrades** → *Upgrade version*.
3. **Approach 1 (recommended):** select target version → scope **Upgrade control plane only** → Save. Then upgrade node pools separately (in place, or blue-green below).
4. **Approach 2:** scope **Upgrade control plane + all node pools** — use when separate upgrades are not offered.
5. Validate node pool versions afterwards.

### In-place node pool upgrade

Node pool → *Upgrade Kubernetes* → choose version → Save. AKS does a rolling update: cordon + drain a node, replace it with a new-version node, reschedule pods, repeat (max surge controls how many extra nodes are added).

### Blue-green node pool upgrade (production)

Create new pools at the target version with the same shape as the old ones:

| Setting | Value |
|---|---|
| Mode | User (app pools) / System (system pool) |
| OS SKU | Ubuntu Linux |
| Zones | 1, 2, 3 |
| Spot | Disabled |
| Size | Match old pool (e.g. D8s_v3 for heavy pool, D2as_v4 app pool, D2s_v3 system pool) |
| Scaling | Match old pool (manual 8 nodes for heavy pool; autoscale 1–8 app; 2–4 system) |
| Max pods | 110 |
| Public IP per node | Disabled |
| Max surge | 10% |
| Node drain timeout | 30 minutes |
| Labels / taints | `type=user` / `type=system`; system pool `CriticalAddonsOnly=true:NoSchedule` where used |

Then cordon and drain old nodes **one by one**, watching pods move in **k9s**, run sanity checks, and delete the old pools from the portal.

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide -A
```

## Four methods compared

| Method | Steps | Time (sandbox) |
|---|---|---|
| 1. Portal | Upgrades → target version → control plane + node pools → rolling upgrade | ~15–16 min |
| 2. Bicep (atomic) | Change `aks_version` → what-if → deploy (single PUT; RP upgrades CP then pools) | ~18 min |
| 3. Portal + manual blue-green | CP only → new pools in portal → cordon/drain → delete old pools | ~20–25 min |
| 4. Bicep manual blue-green | CP version in Bicep → add new pools in params → cordon/drain → remove old pools → CLI delete old system pool | ~40–45 min |

| | Portal | Bicep | Blue-green (portal) | Blue-green (Bicep) |
|---|---|---|---|---|
| Control | Low | Medium | High | Medium |
| Speed | Fastest | Fast | Medium | Slow |
| Workload safety | Moderate (rolling) | Moderate (rolling) | Highest (zero-downtime potential) | Moderate |
| Complexity | Simple | Simple | Medium | Complex |

**Recommendation:** dev/test → Portal or Bicep atomic. Production → blue-green (3 or 4); it is slower but lets you validate the new version on a few nodes before deleting the old ones.

## Bicep upgrade findings and limitations

### Inline pools upgrade cleanly

Keeping all system and user pools inside `agentPoolProfiles` of the `managedClusters` resource and changing `aks_version` upgraded control plane and pools in one atomic operation (tested 1.33.0 → 1.33.5 with sample nginx/redis/httpd/tomcat deployments running).

```bicep
// params
aks_version = '1.33.5'
```

### Separate agentPools resources conflict

Declaring pools as separate `Microsoft.ContainerService/managedClusters/agentPools` resources generates several PUTs. The AKS resource provider locks the cluster while updating, and a pool request during the `Updating` state fails with:

> `NotAllAgentPoolOrchestratorVersionSpecifiedAndUnchanged` — all agent pools' OrchestratorVersion must be all specified or all unspecified… For agent pool specific change, please use per agent pool operations.

**Fix:** define all pools inline in `agentPoolProfiles` for version changes.

### Adding a pool to an existing cluster

Adding a new object to `agentPoolProfiles` on an existing cluster fails:

> A new agent pool was introduced. Adding agent pools to an existing cluster is not allowed through managed cluster operations.

**Fix:** add new pools (including System pools) as child `agentPools` resources with `mode: 'System'`.

### Deleting a system pool

Removing a system pool from Bicep reports **Succeeded**, but the pool stays — an AKS safety guard against removing the last system pool (CoreDNS, metrics-server). Delete it explicitly:

```bash
az aks nodepool delete --resource-group <rg> --cluster-name <cluster> --name <old-system-pool> --no-wait
```

### Bicep blue-green walkthrough (sandbox)

1. Deploy cluster at 1.33.0; bump control plane to 1.33.2 and redeploy — only the control plane moves.
2. Add `systempool0`, `userpool0`, `userpool1` to params; what-if confirms three new pools; deploy.
3. Cordon old nodes; drain old nodes.
4. Remove old user pools from params and redeploy — deleted.
5. Delete the old system pool with `az aks nodepool delete`.

## Upgrade template (reusable)

```
Requirement: AKS version upgrade from ____ to ____ (control plane + agent pools)
Current version: ____     Target version: ____
Node pools: System pool (version ____), User pool(s) (version ____)
Window: <date> <time + timezone>     Estimated duration: 30–45 min
Approach: CP only → new pools → cordon/drain → sanity → delete old pools
Rollback: keep old pools until sign-off; uncordon old nodes if new pools misbehave
```

## Cluster tracking sheet columns

S.No · Cluster · Resource group · Location · Current version · Target version · Environment · System pools · User pools · Upgrade done (Y/N) · Upgraded via automation (Y/N) · Variable group · Service connection

## AKS cluster as code

**Files:** `azure/bicep/stacks/wms/aks/main.bicep`, `modules/aks/controlPlane.bicep`, `modules/aks/agentPool.bicep`, `modules/aks/bootstrap.bicep`, `parameters/wms/production/aks/params.bicepparam`

- **Sequential node pool deployment** — each pool depends on the previous one, so user pools are created or updated **one at a time**, never all at once. This is what makes node pool changes controlled and zero-downtime.

```bicep
module pools 'agentPool.bicep' = [for (pool, i) in aksParams.agentPools: {
  name: 'pool-${pool.name}'
  params: { pool: pool, clusterName: controlPlane.outputs.name }
  dependsOn: i == 0 ? [controlPlane] : [pools[i - 1]]
}]
```

- **System vs user pools** (`mode: System/User`), node labels, **zones 1/2/3**, autoscaling (`minCount/maxCount`), optional per-pool subnet.
- **`autoUpgradeProfile`** — `upgradeChannel: none` (Kubernetes version changes only through code) and `nodeOSUpgradeChannel: NodeImage` (OS patches automatic).
- **Add-ons** — Key Vault Secrets Store CSI driver, Container Insights (to Log Analytics), Defender for Containers (`securityProfile.defender`), Web App Routing (managed NGINX ingress), Blob/Disk/File CSI and snapshot controller.
- **Identity and access** — system-assigned cluster identity, optional managed AAD integration (`adminGroupObjectIDs`, `enableAzureRBAC`), `disableLocalAccounts` toggle, OIDC issuer toggle.
- **Kubelet identity role assignments created automatically** — **AcrPull** on ACR (only when `attach_acr`, checked by the helper `can_acr_be_attached`) and a Key Vault role so pods can read secrets.
- **Optional bootstrap** — a UAMI with a narrow custom role (`listClusterUserCredential` only) plus a Deployment Script that applies a `SecretProviderClass`, mounting Key Vault secrets (DB credentials, SMTP…) into pods.
- **Pinned `aks_version`** in params — an upgrade is a one-line change → what-if → stack update.
