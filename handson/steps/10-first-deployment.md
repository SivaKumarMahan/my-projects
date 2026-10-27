# Part 10 — First deployment by hand

[← Part 9 — The Helm chart](09-helm-chart.md) | [Index](../README.md) | [Part 11 — SonarQube Cloud →](11-sonarqube-cloud.md)

---

Deploy once **manually** before automating it, so you understand every step the pipeline will do.

## 10.1 Push images from your laptop

ACR is public (Basic SKU) and your user has Contributor, so you can push from your laptop.

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks
az acr login -n $ACR_NAME
TAG=manual-1
for app in backend frontend; do
  docker build -t $ACR_NAME.azurecr.io/ttapp-$app:$TAG $app
  docker push $ACR_NAME.azurecr.io/ttapp-$app:$TAG
done
az acr repository list -n $ACR_NAME -o table
az acr repository show-tags -n $ACR_NAME --repository ttapp-backend -o table
```

## 10.2 Install with Helm from the jumpbox

```bash
ssh azureuser@$JUMPBOX_IP
git clone https://github.com/$GH_OWNER/$GH_REPO.git && cd $GH_REPO

BACKEND_CLIENT_ID=$(az identity show -g $RG -n id-ttapp-dev-backend --query clientId -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

helm upgrade --install ttapp helm/three-tier-app -n app \
  --rollback-on-failure --wait --timeout 10m \
  --set image.registry=$ACR_NAME.azurecr.io --set image.tag=manual-1 \
  --set azure.tenantId=$TENANT_ID --set azure.keyVaultName=$KV_NAME \
  --set azure.backendClientId=$BACKEND_CLIENT_ID
```

While it runs, watch from a second SSH session:

```bash
kubectl get pods -n app -w
```

You'll see backend pods go `ContainerCreating` (the CSI driver fetches secrets) → `Running 0/1` (startup probe) → `1/1` (readiness passed).

## 10.3 Check every layer

```bash
kubectl get deploy,rs,pods,svc,ingress,hpa,pdb,networkpolicy -n app -o wide
kubectl get endpointslices -n app                       # pod IPs behind each Service

# Ingress ADDRESS = App Gateway public IP (AGIC fills it in, takes ~1 min)
APPGW_IP=$(kubectl get ingress ttapp -n app -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://$APPGW_IP/api/info; echo
for i in $(seq 1 6); do curl -s http://$APPGW_IP/api/info; echo; done    # pod name changes: load balancing

# What AGIC wrote into App Gateway
az network application-gateway show -g $RG -n $APPGW_NAME \
  --query "{listeners:httpListeners[].name, pools:backendAddressPools[].{name:name, ips:backendAddresses[].ipAddress}, probes:probes[].{path:path}}" -o json
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{ip:address, health:health}" -o table
```

The backend pool IPs are exactly the **frontend pod IPs** from `kubectl get pods -n app -o wide`. That's AGIC + Azure CNI.

Open `http://<APPGW_IP>` in the browser and add some todos.

## 10.4 Check the secrets path

```bash
kubectl describe pod -n app -l app=backend | grep -A6 "Environment:"   # AZURE_CLIENT_ID etc. injected by webhook
kubectl get secretproviderclass -n app
kubectl get secret ttapp-backend-secrets -n app -o jsonpath='{.data.DB_USERNAME}' | base64 -d; echo   # pgadmin
kubectl get events -n app --sort-by=.lastTimestamp | tail -20
```

## 10.5 Debug a distroless container

```bash
kubectl exec -n app deploy/ttapp-backend -- sh          # fails: no shell in distroless

# The pod has runAsNonRoot: true, which also applies to debug containers,
# so run busybox as the same non-root user (65532) as the Java process.
echo '{"securityContext":{"runAsUser":65532,"runAsNonRoot":true}}' > /tmp/debug.json
kubectl debug -n app -it $(kubectl get pod -n app -l app=backend -o name | head -1) \
  --image=busybox:1.36 --target=backend --custom=/tmp/debug.json -- sh
# inside the debug container (shares the backend's process namespace):
ls /proc/1/root/mnt/secrets          # the files mounted by the CSI driver
wget -qO- localhost:8080/actuator/health
exit
```

## 10.6 Look at the database

```bash
export PGPASSWORD=$(az keyvault secret show --vault-name $KV_NAME -n db-password --query value -o tsv)
psql "host=$PG_NAME.postgres.database.azure.com port=5432 dbname=appdb user=pgadmin sslmode=require" \
  -c "select * from flyway_schema_history;" -c "select * from todo;"
```

The todos you added in the browser are now in PostgreSQL, so all three tiers work end to end.

---

[← Part 9 — The Helm chart](09-helm-chart.md) | [Index](../README.md) | [Part 11 — SonarQube Cloud →](11-sonarqube-cloud.md)
