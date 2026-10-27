# Part 5 — Jumpbox and GitHub self-hosted runner

[← Part 4 — Terraform stack 1: foundation](04-terraform-foundation.md) | [Index](../README.md) | [Part 6 — The application code →](06-application-code.md)

---

## 5.1 Log in to the jumpbox

```bash
JUMPBOX_IP=$(cd ~/three-tier-aks/infra/foundation && terraform output -raw jumpbox_public_ip)
echo "export JUMPBOX_IP=$JUMPBOX_IP" >> ~/three-tier-env.sh
ssh azureuser@$JUMPBOX_IP

# on the jumpbox
cloud-init status --wait            # wait until "status: done"
ls /var/log/cloud-init-tools-done   # exists = all tools installed
az version && kubectl version --client && kubelogin --version && helm version && psql --version
```

If SSH times out, your public IP has changed. Update `TF_VAR_my_ip` and run `terraform apply` on the foundation again. The NSG rule and the Key Vault firewall both use it.

## 5.2 Prove private DNS works from inside the VNet

```bash
# on the jumpbox
az login --use-device-code
az account set --subscription <your-subscription-id>
nslookup kv-ttapp-<suffix>.vault.azure.net
# → ...privatelink.vaultcore.azure.net  Address: 10.1.1.4   (private endpoint IP!)
```

Same name, different answer. Your laptop got a public IP; the jumpbox gets the private endpoint IP because the hub VNet is **linked** to the zone. Explain this in interviews with exactly this example.

Copy your environment file to the jumpbox so the same variables work there:

```bash
# on your laptop
scp ~/three-tier-env.sh azureuser@$JUMPBOX_IP:~/
# on the jumpbox: add "source ~/three-tier-env.sh" to ~/.bashrc
echo 'source ~/three-tier-env.sh' >> ~/.bashrc
```

(`LOCATION` in that file runs `az group show`, so it works once `az login` is done on the jumpbox.)

## 5.3 Register the GitHub self-hosted runner

1. GitHub repo → **Settings → Actions → Runners → New self-hosted runner** → Linux, x64. Keep the page open; it shows a download URL and a **token** that is valid for 1 hour.
2. On the jumpbox, install the runner **as the `ghrunner` user** (created by cloud-init). That keeps the pipeline's `az login` separate from yours.

```bash
sudo -iu ghrunner
mkdir actions-runner && cd actions-runner
# paste the "Download" commands from the GitHub page (curl -o actions-runner-linux-x64-<ver>.tar.gz ... ; tar xzf ...)
./config.sh --url https://github.com/<owner>/three-tier-aks --token <TOKEN> \
  --name jumpbox-runner --labels aks-deployer --unattended
exit

# back as azureuser: install dependencies and run it as a service (starts on boot)
cd /home/ghrunner/actions-runner
sudo ./bin/installdependencies.sh
sudo ./svc.sh install ghrunner
sudo ./svc.sh start
sudo ./svc.sh status
```

The runner now shows **Idle** in GitHub with labels `self-hosted, Linux, X64, aks-deployer`. Workflows select it with `runs-on: [self-hosted, linux, aks-deployer]`.

> The VM shuts down at 23:00 every night (auto-shutdown). Start it before you run pipelines: `az vm start -g $RG -n vm-ttapp-dev-jumpbox`.

**Interview answer — Microsoft/GitHub-hosted vs self-hosted runners/agents:** hosted runners are clean, maintained by the provider, and sit on the public internet. Self-hosted runners live in *your* network (they can reach private AKS, private endpoints and on-prem), keep caches between runs, and can be bigger machines. But **you** patch them, secure them, and isolate jobs, and you must never let untrusted fork PRs run on them.

---

[← Part 4 — Terraform stack 1: foundation](04-terraform-foundation.md) | [Index](../README.md) | [Part 6 — The application code →](06-application-code.md)
