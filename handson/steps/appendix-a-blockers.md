# Appendix A — Blockers and the easy way around

[← Part 19 — Interview questions → where you did it](19-interview-questions-map.md) | [Index](../README.md) | [Appendix B — Adding QA and Prod later →](appendix-b-qa-and-prod.md)

---

| Blocker | Symptom | Easy way around |
|---|---|---|
| Resource provider not registered | `MissingSubscriptionRegistration` | Ask the admin to run `az provider register -n <name>` once. For `Microsoft.DevTestLab`, just delete the VM shutdown schedule resource |
| Can't read provider status | `cannot read` in Part 1.4 | Continue; only act if an apply fails |
| vCPU quota | `QuotaExceeded` / `OperationNotAllowed` | Use smaller sizes (`Standard_B2s_v2`/`Standard_B4ms`), max 2 user nodes, or another region (the resource group region decides) |
| SKU not available in region | `SkuNotAvailable` for D2s_v5 or B1ms | Try `Standard_D2as_v5`, `Standard_D2s_v4`; for PostgreSQL `B_Standard_B2s`; or check `az vm list-skus -l $LOCATION --size Standard_D2 -o table` |
| Policy denies public IPs | jumpbox or App Gateway IP creation fails | Jumpbox: remove its public IP and use **`az aks command invoke`** for kubectl/helm (runs through the Azure API, no network path needed). Deploy job: GitHub-hosted runner + `az aks command invoke -g $RG -n $AKS_NAME --command "helm upgrade ..." --file helm`. App Gateway needs a public frontend IP; ask for a policy exemption for this RG |
| Policy requires tags | node resource group creation fails | Add the required tags to `var.tags`. If the policy targets the AKS-created node RG, ask the admin for an exemption |
| Role assignment refused | `AuthorizationFailed` on `roleAssignments/write` | Your User Access Administrator may have a **condition** that limits which roles you can assign. Ask which roles are allowed, or ask the admin to create those 3–4 assignments once |
| `az ad signed-in-user show` blocked | Graph permission error | Copy your Object ID from Portal → Microsoft Entra ID → Users → you |
| Key Vault 403 from laptop | `Forbidden ... client address not authorized` | Your public IP changed: `export TF_VAR_my_ip=$(curl -s ifconfig.me)` and re-apply the foundation |
| Need a private GitHub repo | SonarQube Cloud free is public-only; environment approvals on private repos need a paid plan | Run **SonarQube Community Build** on the jumpbox: `docker run -d --name sonarqube -p 9000:9000 sonarqube:community`, create a token at `http://localhost:9000` (via SSH tunnel), run the Sonar steps on the self-hosted runner with `-Dsonar.host.url=http://localhost:9000`. Skip environment approvals or use `workflow_dispatch` as the "approval" |
| Self-hosted runners not allowed | | GitHub-hosted runner + `az aks command invoke` for deploy; Terraform from your laptop (Key Vault firewall allows your IP) |
| App Gateway takes too long or costs too much | | Use NGINX Ingress with an internal LB + `kubectl port-forward` from the jumpbox. You lose the WAF part, but keep everything else |
| AGIC pod not running | Ingress has no ADDRESS | `kubectl get pods -n kube-system -l app=ingress-appgw`; check the user node pool exists and the AGIC role assignments (Part 8.3) |

---

[← Part 19 — Interview questions → where you did it](19-interview-questions-map.md) | [Index](../README.md) | [Appendix B — Adding QA and Prod later →](appendix-b-qa-and-prod.md)
