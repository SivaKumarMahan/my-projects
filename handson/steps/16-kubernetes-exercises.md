# Part 16 — Kubernetes exercises

[← Part 15 — Monitoring with Prometheus and Grafana](15-prometheus-and-grafana.md) | [Index](../README.md) | [Part 17 — Break-and-fix lab →](17-break-and-fix-lab.md)

---

Run these on the jumpbox. Keep a second SSH session open with `kubectl get pods -n app -w` so you can watch the changes live.

```bash
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
```

## 16.1 Zero-downtime rolling update (prove it)

Terminal 1 sends a request every 0.2 s and counts the status codes:

```bash
while true; do curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/api/info; sleep 0.2; done | uniq -c
```

Terminal 2 triggers a rollout:

```bash
kubectl rollout restart deploy/ttapp-backend deploy/ttapp-frontend -n app
kubectl rollout status deploy/ttapp-backend -n app
```

Terminal 1 should show only `200`. What makes it work: readiness probes + `maxUnavailable: 0` + `preStop` sleep + Spring graceful shutdown + App Gateway connection draining. **Experiment:** remove the `preStop` block and repeat. You may see a few 502s, because App Gateway still sends traffic to a pod that is already shutting down.

## 16.2 HPA under load

```bash
kubectl get hpa -n app -w          # terminal 2

# terminal 1: 50 parallel clients
seq 1 30000 | xargs -P 50 -I{} curl -s -o /dev/null http://$APPGW_IP/api/todos
```

Watch `TARGETS` climb above 70% and `REPLICAS` go 2 → 3 → 4… Stop the load. After about 5 minutes (the scale-down stabilization window) it returns to 2.

```bash
kubectl top pods -n app
kubectl describe hpa ttapp-backend -n app      # Events: "New size: 4; reason: cpu resource utilization above target"
```

If CPU doesn't rise enough, lower the target to 30% (`--set backend.hpa.targetCPUUtilizationPercentage=30`) and redeploy.

## 16.3 Cluster Autoscaler

Create pods that **request** more CPU than the user pool has:

```bash
kubectl create deployment inflate --image=mcr.microsoft.com/oss/kubernetes/pause:3.6 --replicas=0
kubectl set resources deployment inflate --requests=cpu=500m,memory=256Mi
kubectl scale deployment inflate --replicas=8
kubectl get pods -l app=inflate          # some Pending
kubectl get events --sort-by=.lastTimestamp | grep -iE "TriggeredScaleUp|FailedScheduling" | tail
kubectl get nodes -w                     # a new "aks-user-..." node appears in 2-4 minutes
```

Clean up and watch it scale down (about 10 minutes of low usage):

```bash
kubectl delete deployment inflate
```

Look at the decisions in Log Analytics (diagnostic category `cluster-autoscaler`):

```kusto
AKSControlPlane | where Category == "cluster-autoscaler" | project TimeGenerated, Message | order by TimeGenerated desc | take 50
```

**HPA vs VPA vs Cluster Autoscaler:** HPA changes the **number of pods** from metrics. VPA changes **pod requests and limits** (it restarts pods to apply them). CA changes the **number of nodes** when pods can't be scheduled, or when nodes are under-used. HPA + CA is the usual pair. Don't let HPA and VPA both act on **CPU or memory for the same workload**, because they fight; VPA in "Off" (recommendation) mode is safe.

## 16.4 VPA recommendations

`k8s/vpa-frontend.yaml`:

```yaml
# VPA in recommendation-only mode: it suggests CPU/memory requests but changes nothing.
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: frontend-vpa
  namespace: app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ttapp-frontend
  updatePolicy:
    updateMode: "Off"
```

```bash
kubectl apply -f k8s/vpa-frontend.yaml
# wait a few minutes with some traffic, then:
kubectl describe vpa frontend-vpa -n app      # "Target" CPU/memory = recommended requests
```

Compare the recommendation with the `resources` in `values.yaml`. This is how you right-size requests, which feeds into cost optimization.

## 16.5 PodDisruptionBudget and node drain

```bash
NODE=$(kubectl get nodes -l agentpool=user -o jsonpath='{.items[0].metadata.name}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=10m
```

With one user node, the evicted pods have nowhere to go. They go Pending, and the **Cluster Autoscaler adds a node**. The PDB (`minAvailable: 1`) makes the drain **wait**, evicting one pod at a time, so each tier keeps a running pod. Watch the "Cannot evict pod as it would violate the pod's disruption budget" messages. Then:

```bash
kubectl uncordon $NODE
```

This is exactly what happens during an AKS upgrade.

## 16.6 NetworkPolicy tests

```bash
# A random pod in "app" cannot reach the backend (default deny)
kubectl run np-test -n app --rm -it --restart=Never --image=busybox:1.36 \
  -- wget -qO- -T 3 http://ttapp-backend:8080/api/info
# → "download timed out"

# The same pod labelled app=frontend CAN (the policy allows frontend → backend)
kubectl run np-test2 -n app --rm -it --restart=Never --image=busybox:1.36 --labels app=frontend \
  -- wget -qO- -T 3 http://ttapp-backend:8080/api/info
# → {"pod":"ttapp-backend-...","version":"..."}

# The frontend cannot reach the internet (egress only to backend + DNS)
kubectl exec -n app deploy/ttapp-frontend -- wget -qO- -T 3 https://www.microsoft.com
# → times out
```

`kubectl get networkpolicy -n app` then `kubectl describe networkpolicy backend -n app` shows the rules. In an interview, draw the allowed arrows: AppGW subnet → frontend → backend → PostgreSQL.

## 16.7 Layer 4 internal load balancer (Service type LoadBalancer)

App Gateway is L7. Now build an **L4 internal** Azure Load Balancer to see the difference:

```bash
kubectl create deployment hello --image=nginxinc/nginx-unprivileged:1.31-alpine
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: hello-ilb
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"   # private IP from the VNet
spec:
  type: LoadBalancer
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 8080
EOF
kubectl get svc hello-ilb -w           # EXTERNAL-IP becomes 10.1.4.x (not public)
curl -s http://<that IP> | head -5      # works from the jumpbox over peering
```

What happened: the cloud-controller-manager inside AKS created a **frontend IP + rule + health probe** on an internal Standard Load Balancer (`kubernetes-internal`, in the node resource group). Without the annotation it would get a **public** IP. If `EXTERNAL-IP` stays `<pending>`, the control-plane identity is missing **Network Contributor** on the subnet. (We granted it on the spoke VNet in `aks.tf`.)

```bash
kubectl delete svc hello-ilb && kubectl delete deployment hello
```

**Service types recap:** ClusterIP (internal virtual IP), NodePort (a port on every node, 30000–32767), LoadBalancer (Azure LB, public or internal), ExternalName (DNS CNAME), headless (`clusterIP: None`: DNS returns pod IPs directly, used by StatefulSets).

## 16.8 Helm rollback, history, and a specific chart version

```bash
helm history ttapp -n app                  # every revision, with chart version + status
helm get values ttapp -n app               # values used by the current revision
helm get manifest ttapp -n app | less      # exactly what was applied

helm rollback ttapp <REVISION> -n app --wait
helm history ttapp -n app                  # a NEW revision appears ("Rollback to N")
```

**Package the chart and store versions in ACR (OCI)**. This answers "how do you deploy one specific version out of three?"

```bash
cd ~/three-tier-aks
az acr login -n $ACR_NAME --expose-token --output tsv --query accessToken \
  | helm registry login $ACR_NAME.azurecr.io --username 00000000-0000-0000-0000-000000000000 --password-stdin

helm package helm/three-tier-app                    # three-tier-app-0.1.0.tgz
helm push three-tier-app-0.1.0.tgz oci://$ACR_NAME.azurecr.io/helm
# bump version: 0.2.0 in Chart.yaml, package and push again; then:
helm upgrade --install ttapp oci://$ACR_NAME.azurecr.io/helm/three-tier-app --version 0.1.0 -n app --reuse-values
```

**Rollback strategy (interview answer):** application = `helm rollback` (or redeploy the previous Git SHA; images are immutable). Database = Flyway migrations are **forward-only and backward-compatible**: add columns, don't rename or drop in the same release (expand/contract). So the old app version still works against the new schema, and you roll back the app without touching the DB. For a destructive mistake, restore PostgreSQL to a point in time (16.10).

## 16.9 Secret rotation

Rotate the database password **without** downtime:

```bash
NEWPW="$(openssl rand -base64 18 | tr -d '/+=')Aa1!"
az postgres flexible-server update -g $RG -n $PG_NAME --admin-password "$NEWPW"
az keyvault secret set --vault-name $KV_NAME -n db-password --value "$NEWPW" -o none

# Within ~2 minutes the CSI driver updates the mounted file AND the synced Kubernetes Secret:
kubectl get secret ttapp-backend-secrets -n app -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
```

But the **environment variable inside running pods does not change**. Env vars are read once at start-up. Existing DB connections keep working; new connections would fail with the old password. So:

```bash
kubectl rollout restart deploy/ttapp-backend -n app      # rolling, zero downtime
```

**Interview answer — "will rotating a secret cause downtime?"** Not if you do it right. Either read the secret from the mounted **file** (the app re-reads it), or roll the pods after rotation (tools like Reloader automate this). For zero risk, use **two credentials**: create the new one, roll the pods, then remove the old one.

> The next `terraform apply` of the platform stack sets the password back to the one in state (drift). For this lab that is fine, because the stack is recreated anyway. In real life, either let Terraform own rotation, or use `ignore_changes`.

## 16.10 PostgreSQL backup and point-in-time restore

Flexible Server takes automatic backups (7 days retention, as set in Terraform).

```bash
az postgres flexible-server show -g $RG -n $PG_NAME --query "{earliestRestore:backup.earliestRestoreDate, retention:backup.backupRetentionDays}" -o table
# Restore to a NEW server (it never overwrites the original):
az postgres flexible-server restore -g $RG --name ${PG_NAME}-restored --source-server $PG_NAME \
  --restore-time "$(date -u -d '-15 min' +%Y-%m-%dT%H:%M:%SZ)"
# compare data, then delete it (it costs money):
az postgres flexible-server delete -g $RG -n ${PG_NAME}-restored --yes
```

> The restored server copies the source's network settings. You may need to add a private endpoint for it before you can connect from the jumpbox. Even just running the restore and seeing it succeed is a good interview answer.

## 16.11 Upgrade AKS (the full procedure)

**Before**:

```bash
az aks get-upgrades -g $RG -n $AKS_NAME -o table          # available target versions

# Are we calling deprecated APIs? (counter > 0 means yes)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Static check of what Helm renders (Pluto by Fairwinds)
curl -sL https://github.com/FairwindsOps/pluto/releases/download/v5.24.4/pluto_5.24.4_linux_amd64.tar.gz | tar xz pluto
helm template ttapp helm/three-tier-app --set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x | ./pluto detect -
./pluto detect-helm -o wide       # checks the deployed releases
```

Also check: the PDBs are not too strict (`minAvailable` equal to the replica count would block the drain forever); there is enough quota for surge nodes; and you've read the AKS release notes for breaking changes.

**Upgrade with Terraform** (control plane first, then node pools; nodes are surged and drained one by one, respecting PDBs):

```bash
# terminal 1: zero-downtime check from 16.1
# laptop: change kubernetes_version in infra/platform/dev.tfvars to the next minor version
terraform plan -var-file=dev.tfvars -out=tfplan    # in-place update of cluster + 2 node pools
terraform apply tfplan                              # 20-40 minutes
# jumpbox: watch nodes get replaced
kubectl get nodes -o wide -w
```

**Interview answer — challenges you can mention:** PDBs blocking the drain; deprecated APIs in Helm charts (fixed before upgrading, with Pluto); not enough IPs in the subnet for surge nodes with Azure CNI (that's why we sized it /22); add-on versions changing; a node image update rebooting nodes (`node_os_upgrade_channel`). Also, you can only upgrade **one minor version at a time**.

**Automate upgrades:** `automatic_upgrade_channel = "patch"` or `"stable"` + a **maintenance window**, with dev/QA upgraded first. The Terraform `kubernetes_version` must then be ignored or kept in sync.

---

[← Part 15 — Monitoring with Prometheus and Grafana](15-prometheus-and-grafana.md) | [Index](../README.md) | [Part 17 — Break-and-fix lab →](17-break-and-fix-lab.md)
