# Part 18 — Daily start/stop, cost and teardown

[← Part 17 — Break-and-fix lab](17-break-and-fix-lab.md) | [Index](../README.md) | [Part 19 — Interview questions → where you did it →](19-interview-questions-map.md)

---

## 18.1 Start of a lab session

```bash
source ~/three-tier-env.sh
az vm start -g $RG -n vm-ttapp-dev-jumpbox                # runner comes online by itself
# GitHub → Actions → infra → Run workflow: stack=platform, action=apply   (approve) ~20 min
#   or from the laptop: cd infra/platform && terraform apply -var-file=dev.tfvars
# jumpbox: kubectl create namespace app   (new cluster each time)
# GitHub → Actions → app-ci-cd → Run workflow (main)        (approve)
# optional: Part 15.2 (Prometheus/Grafana), Part 13.1 (certificate)
```

## 18.2 End of a lab session (do not skip)

```bash
# GitHub → Actions → infra → Run workflow: stack=platform, action=destroy   (approve) ~10-15 min
az vm deallocate -g $RG -n vm-ttapp-dev-jumpbox           # "stopped" alone still bills compute; deallocate does not
```

Check that nothing expensive is left:

```bash
az resource list -g $RG --query "[].{type:type, name:name}" -o table
```

## 18.3 Rough cost (pay-as-you-go, USD; varies by region; check the Azure Pricing Calculator)

| Resource | While platform is up | Idle (platform destroyed, VM deallocated) |
|---|---|---|
| App Gateway WAF_v2 (fixed part) | ≈ $0.45/h | 0 |
| 2 × D2s_v5 AKS nodes (+ disks) | ≈ $0.20/h | 0 |
| PostgreSQL B1ms + 32 GB | ≈ $0.02/h | 0 |
| Public IPs, load balancer, private endpoints | ≈ $0.05/h | ≈ $0.01/h (KV endpoint, jumpbox IP) |
| Jumpbox B2s | ≈ $0.05/h when running | disk only |
| ACR Basic, Key Vault, DNS zones, Log Analytics (1 GB/day cap) | small | small |
| **Total** | **≈ $0.8/h** | **< $1/day** |

A 4-hour session costs about **$3–4**. Left running for a month, it would be **≈ $600**. Create a **budget alert**: Portal → your resource group → **Cost Management → Budgets → Add** (for example $30/month, email at 50/80/100%).

Interview line on cost optimization: *"Dev infrastructure is created on demand with Terraform and destroyed after use. The VM has auto-shutdown, Log Analytics has a daily cap, App Gateway autoscales from 0 capacity units, nodes are right-sized with VPA recommendations, and the cluster autoscaler removes idle nodes."*

## 18.4 Full teardown (when you are completely done)

```bash
# 1) platform: GitHub infra workflow destroy, or locally:
cd ~/three-tier-aks/infra/platform && terraform destroy -var-file=dev.tfvars
# 2) foundation
cd ../foundation && terraform destroy -var-file=dev.tfvars
# 3) bootstrap resources
az identity delete -g $RG -n id-gh-infra
az identity delete -g $RG -n id-gh-app
az storage account delete -g $RG -n $TFSTATE_SA --yes
# 4) GitHub: remove the self-hosted runner (Settings → Actions → Runners)
```

The Key Vault stays **soft-deleted for 7 days** (purging needs subscription rights) and is then removed automatically. Its name can't be reused until then.

---

[← Part 17 — Break-and-fix lab](17-break-and-fix-lab.md) | [Index](../README.md) | [Part 19 — Interview questions → where you did it →](19-interview-questions-map.md)
