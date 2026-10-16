# 04 — Azure DevOps Pipelines, VM Deployments and Parameter Techniques

A reusable IaC toolkit used to bootstrap Azure environments: Azure DevOps YAML pipelines that deploy ARM or Bicep, build Java apps, deploy them to VMs or App Service, and host static sites.

## Pipeline catalogue

| Pipeline | What it does |
|---|---|
| ARM — VM + Tomcat | Build WAR with Maven → deploy VNet/NSG/public IP/NIC (ARM) → deploy VM with SSH key → install Tomcat over SSH → copy WAR |
| ARM — App Service | Build WAR with Maven → deploy App Service from ARM → `az webapp deploy` the WAR |
| ARM / Bicep — static website | Deploy storage account → enable static website → upload site to `$web` |
| Bicep — App Service | Plan + web app from Bicep, values from pipeline variables |
| Bicep — AKS | `az deployment group validate` → `az deployment group create` for VNet + Log Analytics + AKS |

All pipelines are **manually triggered** (`trigger: none`, `pr: none`) on `ubuntu-latest`, with values from **variable groups** and keys from **secure files**.

## Common steps

```yaml
trigger: none
pr: none
pool: { vmImage: ubuntu-latest }

variables:
  - group: ssh-key-azure
  - group: sandbox-azure

steps:
- task: DownloadSecureFile@1          # SSH key stored as a secure file
  name: deployKey
  inputs: { secureFile: 'key.pem' }

- script: |
    cp $(deployKey.secureFilePath) .
    chmod 600 key.pem
  displayName: Prepare SSH key

- task: Maven@3
  inputs:
    mavenPomFile: 'app/pom.xml'
    goals: 'package'
    jdkVersionOption: '1.11'
    publishJUnitResults: true
    testResultsFiles: '**/surefire-reports/TEST-*.xml'
```

## VM + Tomcat deployment flow

```
Maven package ─► ARM: VNet + subnet + NSG + public IP + NIC ─► look up NIC id ─► ARM: VM (SSH key auth)
     │                                                                              │
     └──────────────────────────────► scp install script, context.xml, tomcat.service ─► ssh: install Tomcat
                                                                                     │
                                                         scp WAR ─► move into Tomcat webapps (auto-deploy)
```

```bash
netInterfaceId=$(az network nic show --name "$NIC_NAME" --resource-group "$RG" --query id -o tsv)

az deployment group create --resource-group "$RG" --name virtualMachineDeploy \
  --template-file ./azure/ARM/virtualMachine.json --parameters @parametersVM.json \
  --parameters pemFile="$SSH_PUBLIC_KEY" networkInterfaceResourceId="$netInterfaceId"

ip=$(az vm list-ip-addresses -g "$RG" -n "$VM_NAME" \
     --query "[].virtualMachine.network.publicIpAddresses[*].ipAddress" -o tsv)
scp -i deploykey.pem installtomcat.sh context.xml tomcat.service "$USER@$ip:/tmp"
ssh -i deploykey.pem "$USER@$ip" "sudo sh /tmp/installtomcat.sh"
```

### Tomcat as a systemd service

```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64"
Environment="CATALINA_HOME=/opt/tomcat/apache-tomcat-9.0.x"
Environment="CATALINA_BASE=/opt/tomcat/apache-tomcat-9.0.x"
Environment="CATALINA_PID=/opt/tomcat/apache-tomcat-9.0.x/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
ExecStart=/opt/tomcat/apache-tomcat-9.0.x/bin/startup.sh
ExecStop=/opt/tomcat/apache-tomcat-9.0.x/bin/shutdown.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

The install script creates a `tomcat` system user (no login shell), installs the JDK, unpacks Tomcat under `/opt/tomcat`, sets ownership, replaces the manager apps' `context.xml` (strict `SameSite` cookies), installs the unit file, `daemon-reload` + `enable --now`, and opens port 8080 in `ufw`.

**Hardening to apply:** don't `chmod 777` directories (`/opt` or Tomcat); use `750` and ownership by `tomcat`. Keep the manager apps restricted to localhost/VPN (the `RemoteAddrValve` was commented out). Use `StrictHostKeyChecking` with a known-hosts entry instead of disabling it.

## Static website on Blob storage

```bash
az deployment group create -g "$RG" --name storageDeploy \
  --template-file ./azure/bicep/storageAccount.bicep --parameters ./azure/bicep/storageAccount.bicepparam
az storage blob service-properties update --account-name "$SA" \
  --static-website --index-document index.html --404-document error.html
az storage blob upload-batch -s ./site -d '$web' --account-name "$SA"
```

The storage template exposes the hardening options as parameters: minimum TLS, HTTPS only, shared key access, OAuth by default, public network access, network ACLs, infrastructure encryption, blob and container soft delete, point-in-time restore, change feed and versioning.

## Three ways of feeding parameters to Bicep

### 1. `readEnvironmentVariable()` — values from pipeline variables

```bicep
using 'storageAccount.bicep'
param storageAccountName = readEnvironmentVariable('accountName')
param minimumTlsVersion  = readEnvironmentVariable('tlsVersion')
param supportsHttpsTrafficOnly = bool(readEnvironmentVariable('supportHTTPS'))
param blobSoftDeleteRetentionDays = int(readEnvironmentVariable('retentionDays'))
param prefix = readEnvironmentVariable('environment', 'QA')       // default value
```

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: '<service-connection>'
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: az deployment group create -g $(resourceGroup) -f storageAccount.bicep -p storageAccount.bicepparam
  env:                               # variable-group values exposed to the CLI
    accountName: $(accountName)
    tlsVersion: $(tlsVersion)
    supportHTTPS: $(supportHTTPS)
```

Environment variables are strings, so booleans and numbers are converted with `bool()` / `int()`.

### 2. `loadYamlContent()` — values from a YAML file

```bicep
using 'mainAks.bicep'
param clusterName       = loadYamlContent('modules/aksParamValue.yml', 'clusterName')
param kubernetesVersion = loadYamlContent('modules/aksParamValue.yml', 'kubernetesVersion')
param maxCount          = loadYamlContent('modules/aksParamValue.yml', 'maxCount')
```

One readable YAML file per environment instead of long param files.

### 3. Key Vault `getSecret()` — secrets resolved at deploy time

```bicep
resource kv 'Microsoft.KeyVault/vaults@2023-02-01' existing = {
  name: kvName
  scope: resourceGroup(kvResourceGroup)
}

module vm 'modules/virtualMachine/virtualMachine.bicep' = {
  name: 'virtualMachine'
  params: {
    pemFile: kv.getSecret('pemKey')        // passed to a @secure() param, never in logs or state
  }
}
```

`getSecret()` only works when passed straight into a module's `@secure()` parameter, and the Key Vault needs *enabled for template deployment*.

## AKS from Bicep (modules)

```
mainAks.bicep
├── modules/vnet          → VNet + AKS subnet (outputs subnetId)
├── modules/log-analytics → workspace (outputs workspaceId)
└── modules/aks           → cluster: system + user pools, Azure CNI + Azure network policy,
                            standard LB, autoscale 1–3, SSH key from Key Vault, Container Insights (omsagent)
```

Pipeline: validate first, then create.

```bash
az deployment group validate -g "$RG" -f azure/bicep/mainAks.bicep -p azure/bicep/mainAks.bicepparam
az deployment group create   -g "$RG" -f azure/bicep/mainAks.bicep -p azure/bicep/mainAks.bicepparam
```

Other techniques in the toolkit:
- **NSG rules from JSON** with `loadJsonContent()` and a `for` loop.
- **Conditional resource:** the VM module only deploys when the username matches a naming rule (`if (contains(userName, 'devops'))`), a simple guardrail.
- **SSH-only Linux VMs:** `disablePasswordAuthentication: true` with the public key in `authorized_keys`.

## Interview Q&A

**How do you pass pipeline values into Bicep without generating param files?**
`readEnvironmentVariable()` in a `.bicepparam` file, with the pipeline exposing variable-group values as environment variables on the CLI task.

**How do you use Key Vault secrets in Bicep?**
`existing` Key Vault reference + `getSecret()` into a module `@secure()` parameter; the vault must allow template deployment.

**Why `validate` before `create`?**
Catches template and parameter errors (and some policy violations) before anything is changed. `what-if` adds a preview of the actual changes.

**What would you change in the VM + Tomcat pipeline today?**
Use a service connection instead of `az login` with a client secret, bake an image (Packer) or use cloud-init instead of SSH scripts, avoid `777` permissions, keep host key checking on, and deploy to App Service or containers where possible.
