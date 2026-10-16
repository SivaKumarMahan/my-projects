# 11 — Production Incidents and Troubleshooting

## Incident 1 — Mass pod evictions from memory pressure (Central US production cluster)

### Symptoms

High volume of evicted pods in the `default` namespace, mainly `wms-fulfillment-services-prod` and `wms-package-planning-prod`.

Events:

```
Pod was rejected: The node had condition: [MemoryPressure].
The node was low on resource: memory. Threshold quantity: 100Mi, available: 29028Ki.
Container package-planning-consumer was using 482960Ki, request is 0, has larger consumption of memory.
The node was low on resource: memory. Threshold quantity: 100Mi, available: 98172Ki.
Container fulfillment-queue-consumer was using 506876Ki, request is 0, has larger consumption of memory.
```

### Investigation

`kubectl top nodes` showed nodes above capacity — 115%, 111% and 108% memory. The kubelet raised `MemoryPressure` and started evicting pods to protect the nodes.

Top consumers:

- `aimsplus-wms-production` — ~2.2 GiB (2295Mi) per pod
- `wms-package-planning-prod` — 480–500 MiB per instance
- `wms-fulfillment-services-prod` — 480–500 MiB per instance

Comparison with the stable West US 2 cluster:

| Cluster | Pool | VM size | Nodes | vCPU | Memory | Scaling |
|---|---|---|---|---|---|---|
| Central (failing) | nodes0 | D8s_v3 | 1 | 8 | 32 GiB | Autoscale |
| | nodes1 | D2as_v4 | 1 | 2 | 8 GiB | Autoscale |
| | nodes2 | D2s_v3 | 2 | 2 | 8 GiB | Autoscale |
| West US 2 (stable) | beastpool | D8s_v3 | **8** | 8 | 32 GiB | **Manual** |
| | apppool | D2as_v4 | 1 | 2 | 8 GiB | Autoscale |
| | agentpool | D2s_v3 | 2 | 2 | 8 GiB | Autoscale |

### Root cause

- **No memory requests** on the consumers (`request is 0`). The scheduler and cluster autoscaler only see *requests*, so they believed nodes were nearly empty while they were actually full.
- **Autoscaler starting at 1 node** — without requests it never triggered a scale-out; nodes overfilled, hit MemoryPressure and evicted en masse.
- The stable region avoided this only because it had 8 large fixed nodes.

### Fix

**1. Cleanup** — 233 of 275 pods purged:

| Status | Count |
|---|---|
| Evicted | 183 |
| Error | 34 |
| ContainerStatusUnknown | 14 |
| OOMKilled | 2 |
| Remaining active | 42 |

```bash
for s in Evicted Error ContainerStatusUnknown OOMKilled; do
  echo "$s: $(kubectl get pods -n default | grep -c "$s")"
  kubectl get pods -n default | grep "$s" | awk '{print $1}' | xargs -r kubectl delete pod -n default
done
```

(Equivalent: `kubectl delete pods -n default --field-selector=status.phase=Failed`.)

**2. Capacity** — heavy pool scaled from 1 to **4** nodes; autoscaler range changed from 1–8 to **4–8**.

**3. Recommendations**

- Set memory requests of at least **500Mi** on consumer deployments (and limits) so the scheduler and autoscaler see real demand.
- Keep the minimum node count at 2–3+ for headroom on traffic spikes.

### Outcome

No pods in Evicted/Error/ContainerStatusUnknown/OOMKilled after the change. Node memory utilization dropped from **55–113%** to **22–64%**; workloads stable.

---

## Incident 2 — Init container failures blocking the main API

### Problem

The main backend Deployment had an init container called **`upgrade`** that ran database migrations before the API container started. When it failed or crashed, pods never became ready, and the service went down.

| Environment | Cluster | Deployment | Init container |
|---|---|---|---|
| performance | wmsperformancewestus2 | aimsplus-wms-performance | upgrade |
| production | wmsproductioncentral | aimsplus-wms-performance | upgrade |
| production-DR | West US 2 | aimsplus-wms-performance | upgrade |

### Findings

- Only `manifests/deployment.yml` is used by the release pipelines; `devdeployment.yml` is not referenced by any pipeline.
- All five backend services are deployed from that shared manifest, so a migration failure affected every rollout.

### Solution

- Remove the init container from `deployment.yml`.
- Move migrations into a separate `upgrade.yaml` workload with **its own release pipeline** per environment (`wms-upgrade-sandbox`, `wms-upgrade-production`).
- Production upgrade stage configuration:
  - link the production variable group to the stage
  - enable the `kubectl apply` task
  - attach the production Kubernetes service connection
  - in the kubectl task's *Secrets* section, select the production ACR for the image pull secret

Result: migrations run, and can be retried, on their own; a failed migration no longer takes the API pods down.

---

## Investigation — environment variables changing after slot swap

See [13-webapp-deployment-slots.md](13-webapp-deployment-slots.md). Root cause: settings not marked as *deployment slot setting* (non-sticky) move with the code during a swap.

---

## Troubleshooting cheat sheet used on this project

```bash
kubectl top nodes
kubectl top pods -n default --sort-by=memory
kubectl describe node <node> | grep -A5 Conditions
kubectl get events -n default --sort-by=.lastTimestamp | grep -i evict
kubectl get pods -n default -o wide | grep -v Running
kubectl describe pod <pod> -n default        # init container status, OOMKilled reason
kubectl logs <pod> -c upgrade -n default     # init container logs
az aks nodepool scale -g <rg> --cluster-name <cluster> -n <pool> --node-count 4
az aks nodepool update -g <rg> --cluster-name <cluster> -n <pool> --update-cluster-autoscaler --min-count 4 --max-count 8
```
