# Part 17 — Break-and-fix lab

[← Part 16 — Kubernetes exercises](16-kubernetes-exercises.md) | [Index](../README.md) | [Part 18 — Daily start/stop, cost and teardown →](18-start-stop-cost-teardown.md)

---

Each lab: **break → look at the symptoms → investigate → find the root cause → fix → write a 4-line RCA.** Do them one at a time, and make sure the app is healthy before the next one.

RCA template (write it in your notes):

```text
Impact:      what users saw, from when to when
Root cause:  the technical reason
Fix:         what restored service
Prevention:  what stops it happening again (alert, test, policy, automation)
```

## Lab 1 — ImagePullBackOff

```bash
# break
kubectl -n app set image deploy/ttapp-frontend frontend=$ACR_NAME.azurecr.io/ttapp-frontend:does-not-exist
# investigate
kubectl get pods -n app
kubectl describe pod -n app <new frontend pod> | sed -n '/Events/,$p'   # "manifest unknown" / "not found"
curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/            # still 200!
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

Notice: **no outage**. With `maxUnavailable: 0`, the old pods keep serving while the new one fails. Variant: a **401 Unauthorized** pull error means the kubelet identity lost **AcrPull**. Check with `az role assignment list --scope <acr id>`.

## Lab 2 — CrashLoopBackOff

```bash
# break: JVM cannot start with a 1 MB heap
kubectl -n app set env deploy/ttapp-backend JAVA_TOOL_OPTIONS="-Xmx1m"
# investigate
kubectl get pods -n app -w                                       # Error → CrashLoopBackOff, RESTARTS grows
kubectl logs -n app <crashing pod> --previous                     # "Too small maximum heap"
kubectl describe pod -n app <crashing pod> | grep -A5 "Last State"
# wait 5-10 min: the CrashLoopBackOff log alert sends you an EMAIL
# fix
kubectl rollout undo deploy/ttapp-backend -n app
```

`--previous` is the key flag: the current container has just restarted, so the error is in the **previous** container's log. CrashLoopBackOff causes in real life: bad config or env var, missing secret, the app can't reach a dependency at start-up, wrong command or entrypoint, failing liveness probe.

## Lab 3 — OOMKilled

```bash
# break: JVM grabs 400 MB immediately, the limit is 256 MB
kubectl -n app set resources deploy/ttapp-backend --limits=memory=256Mi
kubectl -n app set env deploy/ttapp-backend JAVA_TOOL_OPTIONS="-Xms400m -XX:+AlwaysPreTouch"
# investigate
kubectl get pods -n app -w
kubectl describe pod -n app <pod> | grep -A4 "Last State"         # Reason: OOMKilled, Exit Code: 137
kubectl top pods -n app
# fix (two changes were made, so roll back two revisions)
kubectl rollout history deploy/ttapp-backend -n app
kubectl rollout undo deploy/ttapp-backend -n app --to-revision=<the revision before the two changes>
```

Exit code **137** = 128 + 9 (SIGKILL from the kernel OOM killer). Real fixes: right-size the limit (VPA recommendation, Grafana memory graph), set the JVM heap as a **percentage** of the container limit (`MaxRAMPercentage`, as we do), find memory leaks (heap dump, the `jvm_memory_used_bytes` trend).

## Lab 4 — Pod stuck in Pending

```bash
# break: no node has 3 free CPUs (Standard_D2s_v5 = 2 vCPU)
kubectl -n app set resources deploy/ttapp-frontend --requests=cpu=3
# investigate
kubectl get pods -n app                                            # Pending
kubectl describe pod -n app <pending pod> | sed -n '/Events/,$p'   # "0/2 nodes are available: Insufficient cpu"
                                                                   # + "pod didn't trigger scale-up" (no node size fits)
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

Other Pending reasons to know: taints with no matching toleration (try a `nodeSelector` that matches only the system pool), a `nodeSelector` that matches no node, a PVC not bound, the max pods per node reached, or not enough IPs in the subnet (Azure CNI).

## Lab 5 — App Gateway returns 502 Bad Gateway

```bash
# break
kubectl -n app scale deploy/ttapp-frontend --replicas=0
# investigate
curl -s -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/            # 502
kubectl get endpointslices -n app -l kubernetes.io/service-name=ttapp-frontend   # no endpoints
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{ip:address, health:health}" -o table
# fix
kubectl -n app scale deploy/ttapp-frontend --replicas=2
```

**502/504 even though all pods are Running.** The usual causes: the App Gateway health probe fails (wrong path or port, or readiness fails); a NetworkPolicy or NSG blocks the App Gateway subnet → pod traffic; AGIC lost permissions so the backend pool is stale (check the AGIC pod logs: `kubectl logs -n kube-system -l app=ingress-appgw`); or the backend is too slow (504, request timeout).

## Lab 6 — NetworkPolicy blocks frontend → backend

```bash
# break: remove "frontend" from the backend policy's allowed sources
kubectl -n app patch networkpolicy backend --type=json -p='[{"op":"remove","path":"/spec/ingress/0/from/0"}]'
# investigate
curl -s -m 70 -o /dev/null -w "%{http_code}\n" http://$APPGW_IP/api/info    # 504 after nginx's 60 s timeout
kubectl exec -n app deploy/ttapp-frontend -- wget -qO- -T 3 http://ttapp-backend:8080/actuator/health  # timed out
kubectl get networkpolicy backend -n app -o yaml
# fix: re-run the app-ci-cd workflow, or helm upgrade with the same values (Helm restores the policy)
```

Lesson: **pod Running + probes OK + "timeout" between pods = check NetworkPolicy first.**

## Lab 7 — PostgreSQL blocked by NSG

```bash
# break
az network nsg rule update -g $RG --nsg-name nsg-ttapp-dev-pe -n Allow-Postgres-From-AKS-And-Jumpbox --access Deny
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl get pods -n app                 # new backend pods 0/1 (readiness includes DB) → rollout stuck; old pods still serve
kubectl logs -n app <new backend pod> | grep -iE "connection|timeout" | head
kubectl run dbtest --rm -it --restart=Never --image=busybox:1.36 -- sh -c \
  "nslookup $PG_NAME.postgres.database.azure.com; nc -zv -w 3 $PG_NAME.postgres.database.azure.com 5432"
#   DNS → 10.1.1.x (fine)   nc → timed out (blocked)  ⇒ network, not DNS
az network nsg rule list -g $RG --nsg-name nsg-ttapp-dev-pe -o table
# fix
az network nsg rule update -g $RG --nsg-name nsg-ttapp-dev-pe -n Allow-Postgres-From-AKS-And-Jumpbox --access Allow
```

Then run **infra → foundation → plan**. Terraform shows **no drift**, because you put it back by hand. Break it again and run the plan **before** fixing: Terraform shows the drift and `apply` repairs it. That's your "how do you detect manual changes?" story.

**"App in AKS cannot connect to PostgreSQL" checklist:** DNS (private zone linked? resolves to 10.1.1.x?) → network (NSG, NetworkPolicy egress, `nc -zv`) → server (status, public access, firewall) → credentials (Key Vault secret, synced Secret, env var) → TLS (`sslmode=require`) → connection limits (max connections, Hikari pool).

## Lab 8 — Private DNS zone link removed

```bash
# break
az network private-dns link vnet delete -g $RG -z privatelink.postgres.database.azure.com -n link-postgres-spoke --yes
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- nslookup $PG_NAME.postgres.database.azure.com
#   → resolves to a PUBLIC IP now → connection refused (public access disabled)
# fix: Terraform recreates the link
#   GitHub: infra → foundation → apply   (or locally: terraform apply -var-file=dev.tfvars)
```

This is the classic "worked yesterday, now can't connect after someone changed DNS or networking" production issue.

## Lab 9 — Key Vault access removed (secrets can't mount)

```bash
# break
BACKEND_PID=$(az identity show -g $RG -n id-ttapp-dev-backend --query principalId -o tsv)
az role assignment delete --assignee $BACKEND_PID --role "Key Vault Secrets User" \
  --scope $(az keyvault show -n $KV_NAME --query id -o tsv)
kubectl rollout restart deploy/ttapp-backend -n app
# investigate
kubectl get pods -n app                                 # new pod stuck in ContainerCreating
kubectl describe pod -n app <stuck pod> | sed -n '/Events/,$p'   # FailedMount ... 403 Forbidden ... does not have secrets get permission
# fix: infra → platform → apply (Terraform recreates the role assignment), then the pod mounts within ~2 min
```

## Lab 10 — Readiness probe misconfigured

```bash
# break
kubectl -n app patch deploy ttapp-frontend --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/wrong"}]'
# investigate
kubectl get pods -n app                                 # new pod Running but 0/1 READY, rollout stuck
kubectl describe pod -n app <new pod> | grep -i unhealthy   # Readiness probe failed: HTTP probe failed with statuscode: 404
kubectl rollout status deploy/ttapp-frontend -n app --timeout=60s    # times out
# fix
kubectl rollout undo deploy/ttapp-frontend -n app
```

In the pipeline, the same situation makes `helm upgrade --rollback-on-failure` time out and **roll back automatically**. That's your "automatic rollback" story.

## Lab 11 — Terraform state lock

```bash
# laptop, terminal 1
cd ~/three-tier-aks/infra/platform && terraform apply -var-file=dev.tfvars   # leave it at the "Enter a value" prompt
# laptop, terminal 2
terraform plan -var-file=dev.tfvars
#   Error acquiring the state lock ... Lock Info: ID, Path, Operation: OperationTypeApply, Who, Created
az storage blob show --account-name $TFSTATE_SA -c tfstate -n platform.tfstate --auth-mode login \
  --query "properties.lease" -o json      # state: leased
# terminal 1: answer "no" → lock released
```

`plan` **and** `apply` both take the lock (unless `-lock=false`), so the lock details appear in both. If a crashed run leaves a stale lock, use `terraform force-unlock <LOCK_ID>`, but only after you are **sure** nobody else is running.

## Lab 12 — Pipeline identity has no rights outside its namespace (RBAC)

```bash
# 1) See exactly what the app pipeline identity is allowed to do
az role assignment list --assignee $APP_PRINCIPAL_ID --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table
#   Azure Kubernetes Service RBAC Admin   .../managedClusters/aks-ttapp-dev/namespaces/app
#   Azure Kubernetes Service Cluster User Role   .../managedClusters/aks-ttapp-dev
#   AcrPush, Reader ...
```

2) Prove it. In `app-ci-cd.yml`, add a temporary step to `deploy-dev`, right after "Get kubeconfig":

```yaml
      - name: RBAC test (remove after)
        run: kubectl get pods -n kube-system
```

Push to `main` and approve the deploy. The step fails with **Forbidden**: `... cannot list resource "pods" in API group "" in the namespace "kube-system"`. Remove the step and push again.

Interview line: *"The CI/CD identity can only deploy into its own namespace. Cluster-wide changes need a human admin role, and every action is in the audit log (AKSAuditAdmin)."*

---

[← Part 16 — Kubernetes exercises](16-kubernetes-exercises.md) | [Index](../README.md) | [Part 18 — Daily start/stop, cost and teardown →](18-start-stop-cost-teardown.md)
