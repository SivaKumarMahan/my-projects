# Appendix B — Adding QA and Prod later

[← Appendix A — Blockers and the easy way around](appendix-a-blockers.md) | [Index](../README.md) | [Appendix C — Optional upgrades →](appendix-c-optional-upgrades.md)

---

1. **State**: same code, a new state key per environment (`key = "qa/platform.tfstate"`) and `qa.tfvars` / `prod.tfvars` (different CIDRs, SKUs, node counts). Production: a separate subscription or resource group, its own VNet ranges that don't overlap dev.
2. **Modules**: move `foundation` and `platform` into `infra/modules/`, and create thin `infra/envs/{dev,qa,prod}` roots that call them with different variables.
3. **GitHub environments** `qa` and `prod`, each with its own approvers, branch rules and variables (`AKS_NAME`, `RESOURCE_GROUP`), plus a federated credential per environment (`environment:qa`, `environment:prod`) on separate identities.
4. **Promote, don't rebuild**: deploy the same image SHA to QA, then Prod. If Prod has its own ACR, copy the image: `az acr import -n <prodacr> --source <devacr>.azurecr.io/ttapp-backend:<sha> --image ttapp-backend:<sha>`. Better still, deploy by **digest** (`image@sha256:...`).
5. **Pipeline**: `deploy-dev → deploy-qa (auto tests) → deploy-prod (approval)` as dependent jobs; the same Helm chart with `values-<env>.yaml`.

---

[← Appendix A — Blockers and the easy way around](appendix-a-blockers.md) | [Index](../README.md) | [Appendix C — Optional upgrades →](appendix-c-optional-upgrades.md)
