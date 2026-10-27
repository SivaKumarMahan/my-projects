# Appendix D — Command cheat sheet

[← Appendix C — Optional upgrades](appendix-c-optional-upgrades.md) | [Index](../README.md)

---

```bash
# ---------- Azure / AKS ----------
az aks get-credentials -g $RG -n $AKS_NAME --overwrite-existing && kubelogin convert-kubeconfig -l azurecli
az aks show -g $RG -n $AKS_NAME -o table
az aks get-upgrades -g $RG -n $AKS_NAME -o table
az aks nodepool list -g $RG --cluster-name $AKS_NAME -o table
az aks command invoke -g $RG -n $AKS_NAME --command "kubectl get pods -A"     # private cluster without network access

# ---------- kubectl ----------
kubectl get pods -n app -o wide -w
kubectl describe pod <pod> -n app                 # Events at the bottom = the answer, most of the time
kubectl logs <pod> -n app --previous              # logs of the crashed container
kubectl get events -n app --sort-by=.lastTimestamp
kubectl top pods -n app ; kubectl top nodes
kubectl rollout status|history|undo|restart deploy/<name> -n app
kubectl get endpointslices -n app
kubectl auth can-i --list -n app
kubectl debug -n app -it <pod> --image=busybox:1.36 --target=<container> --custom=/tmp/debug.json -- sh

# ---------- Helm ----------
helm upgrade --install <rel> <chart> -n app --rollback-on-failure --wait --timeout 10m -f values.yaml --set k=v
helm list -n app ; helm history <rel> -n app ; helm get values <rel> -n app ; helm get manifest <rel> -n app
helm rollback <rel> <rev> -n app --wait
helm template <rel> <chart> --set k=v | less ; helm lint <chart>

# ---------- Terraform ----------
terraform init -backend-config=backend.hcl
terraform fmt -recursive && terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan && terraform apply tfplan
terraform state list ; terraform state show <addr>
terraform import <addr> <azure id> ; terraform force-unlock <lock id>
terraform destroy -var-file=dev.tfvars

# ---------- Networking checks ----------
nslookup <name>                                   # private (10.x) or public answer?
nc -zv -w 3 <host> <port>
az network application-gateway show-backend-health -g $RG -n $APPGW_NAME -o json
az network nsg rule list -g $RG --nsg-name <nsg> -o table
az network private-dns link vnet list -g $RG -z <zone> -o table
az network private-endpoint list -g $RG -o table
```

> When a topic from this lab comes up in an interview, note it in the interview tracker (`interview-questions` repo → `real-interview-questions/README.md`) with "did it hands-on: my-projects/handson Part N".

---

[← Appendix C — Optional upgrades](appendix-c-optional-upgrades.md) | [Index](../README.md)
