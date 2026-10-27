# Part 4 — Terraform stack 1: foundation

[← Part 3 — Networking explained (read before Terraform)](03-networking-explained.md) | [Index](../README.md) | [Part 5 — Jumpbox and GitHub self-hosted runner →](05-jumpbox-and-runner.md)

---

Create every file below in `infra/foundation/`. Read the comments in each file; they are the explanation.

## 4.1 Provider and backend — `versions.tf`

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.11"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.13"
    }
  }

  # Values come from backend.hcl at "terraform init" time (partial configuration).
  backend "azurerm" {}
}

provider "azurerm" {
  features {
    key_vault {
      # Purging a deleted vault needs a subscription-level permission we do not have.
      purge_soft_delete_on_destroy = false
    }
  }

  subscription_id = var.subscription_id

  # We only have rights on one resource group, so never try to register providers.
  resource_provider_registrations = "none"

  # Use Entra ID (not storage keys) for data-plane calls to storage.
  storage_use_azuread = true
}
```

Key points:

- `backend "azurerm" {}` is empty because the values come from `backend.hcl` at `init` time ("partial configuration"). The same code can then point at a different state file per environment.
- `resource_provider_registrations = "none"`: azurerm 5.x already defaults to this. We set it explicitly so it is visible.
- `storage_use_azuread = true`: Terraform talks to storage with Entra ID, not account keys (we disabled keys).

## 4.2 Backend config — `backend.hcl`

```hcl
resource_group_name  = "rg-your-existing-rg" # echo $RG
storage_account_name = "sttfstatesk01"       # echo $TFSTATE_SA
container_name       = "tfstate"
key                  = "foundation.tfstate"
use_azuread_auth     = true
```

Replace the two values with `echo $RG` and `echo $TFSTATE_SA`.

## 4.3 Variables — `variables.tf`

```hcl
variable "subscription_id" {
  type        = string
  description = "Subscription that holds your resource group."
}

variable "resource_group_name" {
  type        = string
  description = "Existing resource group where you have Contributor + User Access Administrator."
}

variable "location" {
  type        = string
  description = "Azure region, for example centralindia or eastus2."
}

variable "prefix" {
  type        = string
  description = "Short name used in every resource name."
  default     = "ttapp"
}

variable "suffix" {
  type        = string
  description = "3-6 lowercase letters/digits to make global names unique (ACR, Key Vault, PostgreSQL)."

  validation {
    condition     = can(regex("^[a-z0-9]{3,6}$", var.suffix))
    error_message = "suffix must be 3-6 lowercase letters or digits."
  }
}

variable "my_ip" {
  type        = string
  description = "Your public IP (curl -s ifconfig.me). Allowed to SSH to the jumpbox and to reach Key Vault."
}

variable "ssh_public_key" {
  type        = string
  description = "Contents of your ~/.ssh/id_ed25519.pub (or id_rsa.pub)."
}

variable "alert_email" {
  type        = string
  description = "Email address that receives Azure Monitor alerts."
}

variable "kv_admin_object_ids" {
  type        = list(string)
  description = "Object IDs that may read/write Key Vault secrets: you and the GitHub infra identity."
}

variable "github_app_principal_id" {
  type        = string
  description = "Principal ID of the GitHub app identity (id-gh-app). Gets AcrPush."
}

variable "hub_address_space" {
  type    = string
  default = "10.0.0.0/16"
}

variable "spoke_address_space" {
  type    = string
  default = "10.1.0.0/16"
}

variable "tags" {
  type = map(string)
  default = {
    project     = "three-tier-aks"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

## 4.4 `main.tf`

```hcl
data "azurerm_client_config" "current" {}

# We cannot create resource groups, so we read the one we were given.
data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

locals {
  rg       = data.azurerm_resource_group.rg.name
  location = var.location
  name     = "${var.prefix}-dev"
}
```

We **read** the resource group with a `data` source, because we are not allowed to create resource groups.

## 4.5 Network — `network.tf`

```hcl
locals {
  # Hub: shared things (jumpbox + GitHub runner).
  snet_jumpbox_cidr = cidrsubnet(var.hub_address_space, 8, 1) # 10.0.1.0/24

  # Spoke: the application.
  snet_appgw_cidr = cidrsubnet(var.spoke_address_space, 8, 0) # 10.1.0.0/24
  snet_pe_cidr    = cidrsubnet(var.spoke_address_space, 8, 1) # 10.1.1.0/24
  snet_aks_cidr   = cidrsubnet(var.spoke_address_space, 6, 1) # 10.1.4.0/22 (nodes + pods with Azure CNI)
}

# ---------- Hub VNet ----------
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-${local.name}-hub"
  resource_group_name = local.rg
  location            = local.location
  address_space       = [var.hub_address_space]
  tags                = var.tags
}

resource "azurerm_subnet" "jumpbox" {
  name                 = "snet-jumpbox"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = [local.snet_jumpbox_cidr]
}

# ---------- Spoke VNet ----------
resource "azurerm_virtual_network" "spoke" {
  name                = "vnet-${local.name}-spoke"
  resource_group_name = local.rg
  location            = local.location
  address_space       = [var.spoke_address_space]
  tags                = var.tags
}

resource "azurerm_subnet" "appgw" {
  name                 = "snet-appgw"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_appgw_cidr]
}

resource "azurerm_subnet" "pe" {
  name                 = "snet-private-endpoints"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_pe_cidr]

  # Without this, NSG rules are NOT applied to private endpoints.
  private_endpoint_network_policies = "Enabled"
}

resource "azurerm_subnet" "aks" {
  name                 = "snet-aks"
  resource_group_name  = local.rg
  virtual_network_name = azurerm_virtual_network.spoke.name
  address_prefixes     = [local.snet_aks_cidr]
}

# ---------- Peering (both directions are required) ----------
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  name                      = "peer-hub-to-spoke"
  resource_group_name       = local.rg
  virtual_network_name      = azurerm_virtual_network.hub.name
  remote_virtual_network_id = azurerm_virtual_network.spoke.id
  allow_forwarded_traffic   = false
}

resource "azurerm_virtual_network_peering" "spoke_to_hub" {
  name                      = "peer-spoke-to-hub"
  resource_group_name       = local.rg
  virtual_network_name      = azurerm_virtual_network.spoke.name
  remote_virtual_network_id = azurerm_virtual_network.hub.id
  allow_forwarded_traffic   = false
}

# ---------- NSGs ----------
resource "azurerm_network_security_group" "appgw" {
  name                = "nsg-${local.name}-appgw"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  # Application Gateway v2 health/management traffic. Without it the gateway fails.
  security_rule {
    name                       = "Allow-GatewayManager"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "GatewayManager"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "65200-65535"
  }

  security_rule {
    name                       = "Allow-Internet-HTTP-HTTPS"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "Internet"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_ranges    = ["80", "443"]
  }
}

resource "azurerm_network_security_group" "aks" {
  name                = "nsg-${local.name}-aks"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  # Documents the one flow that must work: App Gateway -> pod IPs.
  # The default AllowVnetInBound rule would allow it anyway.
  security_rule {
    name                       = "Allow-AppGw-To-Pods"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = local.snet_appgw_cidr
    source_port_range          = "*"
    destination_address_prefix = local.snet_aks_cidr
    destination_port_range     = "8080"
  }
}

resource "azurerm_network_security_group" "pe" {
  name                = "nsg-${local.name}-pe"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  security_rule {
    name                       = "Allow-Postgres-From-AKS-And-Jumpbox"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefixes    = [local.snet_aks_cidr, local.snet_jumpbox_cidr]
    source_port_range          = "*"
    destination_address_prefix = local.snet_pe_cidr
    destination_port_range     = "5432"
  }

  security_rule {
    name                       = "Allow-KeyVault-From-VNet"
    priority                   = 110
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "VirtualNetwork"
    source_port_range          = "*"
    destination_address_prefix = local.snet_pe_cidr
    destination_port_range     = "443"
  }

  security_rule {
    name                       = "Deny-All-Other-VNet-Inbound"
    priority                   = 4000
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_address_prefix      = "VirtualNetwork"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "*"
  }
}

resource "azurerm_network_security_group" "jumpbox" {
  name                = "nsg-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  security_rule {
    name                       = "Allow-SSH-From-My-IP"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_address_prefix      = "${var.my_ip}/32"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "22"
  }
}

resource "azurerm_subnet_network_security_group_association" "appgw" {
  subnet_id                 = azurerm_subnet.appgw.id
  network_security_group_id = azurerm_network_security_group.appgw.id
}

resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks.id
}

resource "azurerm_subnet_network_security_group_association" "pe" {
  subnet_id                 = azurerm_subnet.pe.id
  network_security_group_id = azurerm_network_security_group.pe.id
}

resource "azurerm_subnet_network_security_group_association" "jumpbox" {
  subnet_id                 = azurerm_subnet.jumpbox.id
  network_security_group_id = azurerm_network_security_group.jumpbox.id
}
```

Things to notice:

- `cidrsubnet()` calculates the subnet ranges from the VNet range, so no hard-coded subnets are scattered around.
- **Peering needs two resources**, one in each direction. If you create only one, the status stays `Initiated` instead of `Connected`.
- The App Gateway NSG has the **GatewayManager** rule. Without it, App Gateway v2 deployment fails.

## 4.6 Private DNS zones — `dns.tf`

```hcl
locals {
  # One private DNS zone per Azure service we reach over a private endpoint,
  # plus the zone for the private AKS API server.
  private_dns_zones = {
    keyvault = "privatelink.vaultcore.azure.net"
    postgres = "privatelink.postgres.database.azure.com"
    aks      = "privatelink.${var.location}.azmk8s.io"
  }

  vnets = {
    hub   = azurerm_virtual_network.hub.id
    spoke = azurerm_virtual_network.spoke.id
  }

  # Every zone is linked to every VNet, so names resolve to private IPs from both.
  zone_links = {
    for pair in setproduct(keys(local.private_dns_zones), keys(local.vnets)) :
    "${pair[0]}-${pair[1]}" => { zone = pair[0], vnet = pair[1] }
  }
}

resource "azurerm_private_dns_zone" "zones" {
  for_each            = local.private_dns_zones
  name                = each.value
  resource_group_name = local.rg
  tags                = var.tags
}

resource "azurerm_private_dns_zone_virtual_network_link" "links" {
  for_each             = local.zone_links
  name                 = "link-${each.key}"
  private_dns_zone_id  = azurerm_private_dns_zone.zones[each.value.zone].id
  virtual_network_id   = local.vnets[each.value.vnet]
  registration_enabled = false
  tags                 = var.tags
}
```

`setproduct()` creates 3 zones × 2 VNets = **6 links** from one resource block. Each private DNS zone is a **global** resource; it is linked to VNets, not placed in one.

## 4.7 Monitoring base — `monitoring.tf`

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "log-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
  daily_quota_gb      = 1 # protects your wallet in a lab
  tags                = var.tags
}

resource "azurerm_application_insights" "appi" {
  name                = "appi-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  workspace_id        = azurerm_log_analytics_workspace.law.id
  application_type    = "java"
  tags                = var.tags
}

resource "azurerm_monitor_action_group" "email" {
  name                = "ag-${local.name}-email"
  resource_group_name = local.rg
  short_name          = "ttappmail"
  tags                = var.tags

  email_receiver {
    name                    = "me"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }
}
```

- **Log Analytics workspace**: the database for logs. Container Insights, App Insights, App Gateway and AKS diagnostic logs all go here.
- **Application Insights** (workspace-based): app telemetry (requests, dependencies, exceptions). Its data also lands in the same workspace (`AppRequests`, `AppDependencies` tables).
- **Action group**: *who* gets notified. The alerts themselves live in the platform stack.

## 4.8 Container registry — `acr.tf`

```hcl
resource "azurerm_container_registry" "acr" {
  name                = "acr${var.prefix}${var.suffix}" # alphanumeric only, globally unique
  resource_group_name = local.rg
  location            = local.location
  sku                 = "Basic" # Premium is needed for private endpoints; see the optional step
  admin_enabled       = false   # only Entra ID / RBAC logins
  tags                = var.tags
}

# GitHub Actions (app pipeline) pushes images.
resource "azurerm_role_assignment" "github_app_acr_push" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPush"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}
```

- `admin_enabled = false`: nobody uses the shared admin username and password. Pushes use OIDC + **AcrPush**; AKS pulls with its kubelet identity + **AcrPull**.
- **Basic** SKU keeps the cost low. Private endpoints for ACR need **Premium**; see Appendix C.

## 4.9 Key Vault + private endpoint — `keyvault.tf`

```hcl
resource "azurerm_key_vault" "kv" {
  name                       = "kv-${var.prefix}-${var.suffix}"
  resource_group_name        = local.rg
  location                   = local.location
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"
  rbac_authorization_enabled = true # Azure RBAC instead of access policies
  soft_delete_retention_days = 7
  purge_protection_enabled   = false
  tags                       = var.tags

  # Public endpoint stays on, but only your IP is allowed.
  # Everything inside the VNets uses the private endpoint.
  public_network_access_enabled = true
  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
    ip_rules       = ["${var.my_ip}/32"]
  }
}

resource "azurerm_role_assignment" "kv_admins" {
  for_each             = toset(var.kv_admin_object_ids)
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id         = each.value
}

resource "azurerm_private_endpoint" "kv" {
  name                = "pe-${local.name}-kv"
  resource_group_name = local.rg
  location            = local.location
  subnet_id           = azurerm_subnet.pe.id
  tags                = var.tags

  private_service_connection {
    name                           = "psc-kv"
    private_connection_resource_id = azurerm_key_vault.kv.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  # Creates the A record in privatelink.vaultcore.azure.net automatically.
  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [azurerm_private_dns_zone.zones["keyvault"].id]
  }
}

# RBAC role assignments take a minute or two to apply.
resource "time_sleep" "wait_for_kv_rbac" {
  depends_on      = [azurerm_role_assignment.kv_admins]
  create_duration = "90s"
}

resource "azurerm_key_vault_secret" "appinsights" {
  name         = "appinsights-connection-string"
  value        = azurerm_application_insights.appi.connection_string
  key_vault_id = azurerm_key_vault.kv.id
  depends_on   = [time_sleep.wait_for_kv_rbac]
}
```

- `rbac_authorization_enabled = true` uses Azure RBAC roles instead of the old access policies. In azurerm 5.x this argument is **required**.
- **Network design**: the public endpoint is ON but the firewall allows only *your* IP, so you can manage secrets from your laptop. Everything in the VNets resolves `kv-….vault.azure.net` to the **private endpoint**.
- `time_sleep` waits for role assignments to apply. This is a real-world fix for `403 Forbidden` on the first apply.
- Soft delete is 7 days and purge protection is off. On `destroy`, the vault goes to "soft-deleted" state; we cannot purge it without subscription rights. So keep the foundation stack, or wait 7 days before reusing the name.

## 4.10 Jumpbox — `jumpbox.tf` and `cloud-init.yaml`

```hcl
# One small VM in the hub VNet. It is:
#   1. your jumpbox to reach the private AKS API, PostgreSQL and Key Vault
#   2. the GitHub self-hosted runner that deploys to the private cluster
resource "azurerm_public_ip" "jumpbox" {
  name                = "pip-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_network_interface" "jumpbox" {
  name                = "nic-${local.name}-jumpbox"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.jumpbox.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.jumpbox.id
  }
}

resource "azurerm_linux_virtual_machine" "jumpbox" {
  name                  = "vm-${local.name}-jumpbox"
  resource_group_name   = local.rg
  location              = local.location
  size                  = "Standard_B2s"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.jumpbox.id]
  custom_data           = base64encode(file("${path.module}/cloud-init.yaml"))
  tags                  = var.tags

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = 64
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "ubuntu-24_04-lts"
    sku       = "server"
    version   = "latest"
  }
}

# Stops the VM every night so you do not pay for it while sleeping.
resource "azurerm_dev_test_global_vm_shutdown_schedule" "jumpbox" {
  virtual_machine_id    = azurerm_linux_virtual_machine.jumpbox.id
  location              = local.location
  enabled               = true
  daily_recurrence_time = "2300"
  timezone              = "India Standard Time"
  tags                  = var.tags

  notification_settings {
    enabled = false
  }
}
```

```yaml
#cloud-config
package_update: true
packages:
  - ca-certificates
  - curl
  - jq
  - unzip
  - git
  - docker.io
  - postgresql-client
  - dnsutils
  - netcat-openbsd
runcmd:
  # Azure CLI, kubectl + kubelogin, Helm 4
  - curl -sL https://aka.ms/InstallAzureCLIDeb | bash
  - az aks install-cli
  - curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
  - systemctl enable --now docker
  - usermod -aG docker azureuser
  # Separate Linux user for the GitHub runner, so its "az login" never mixes with yours
  - useradd -m -s /bin/bash ghrunner
  - usermod -aG docker ghrunner
  - touch /var/log/cloud-init-tools-done
```

Change `timezone` to yours. It uses Windows time zone IDs, for example `"UTC"`, `"India Standard Time"`, `"Eastern Standard Time"`.

## 4.11 Outputs — `outputs.tf`

```hcl
# The platform stack reads these through terraform_remote_state.
output "spoke_vnet_id" { value = azurerm_virtual_network.spoke.id }
output "snet_appgw_id" { value = azurerm_subnet.appgw.id }
output "snet_aks_id" { value = azurerm_subnet.aks.id }
output "snet_pe_id" { value = azurerm_subnet.pe.id }
output "snet_appgw_cidr" { value = local.snet_appgw_cidr }
output "snet_pe_cidr" { value = local.snet_pe_cidr }

output "private_dns_zone_ids" {
  value = { for k, z in azurerm_private_dns_zone.zones : k => z.id }
}

output "log_analytics_workspace_id" { value = azurerm_log_analytics_workspace.law.id }
output "action_group_id" { value = azurerm_monitor_action_group.email.id }
output "acr_id" { value = azurerm_container_registry.acr.id }
output "acr_login_server" { value = azurerm_container_registry.acr.login_server }
output "key_vault_id" { value = azurerm_key_vault.kv.id }
output "key_vault_name" { value = azurerm_key_vault.kv.name }
output "jumpbox_public_ip" { value = azurerm_public_ip.jumpbox.ip_address }
```

The platform stack reads these with `terraform_remote_state`.

## 4.12 Values — `dev.tfvars`

```hcl
subscription_id     = "00000000-0000-0000-0000-000000000000" # echo $SUBSCRIPTION_ID
resource_group_name = "rg-your-existing-rg"                  # echo $RG
location            = "centralindia"                         # echo $LOCATION (must match your RG)
prefix              = "ttapp"
suffix              = "sk01"                           # echo $SUFFIX
ssh_public_key      = "ssh-ed25519 AAAA... you@laptop" # cat ~/.ssh/id_ed25519.pub

# my_ip and alert_email are NOT stored here. Pass them as environment variables:
#   export TF_VAR_my_ip=$(curl -s ifconfig.me)
#   export TF_VAR_alert_email="you@example.com"

# echo $MY_OBJECT_ID and echo $INFRA_PRINCIPAL_ID (Part 2)
kv_admin_object_ids = [
  "11111111-1111-1111-1111-111111111111",
  "22222222-2222-2222-2222-222222222222",
]

# echo $APP_PRINCIPAL_ID (Part 2)
github_app_principal_id = "33333333-3333-3333-3333-333333333333"
```

Fill in your values. Get them with `echo $SUBSCRIPTION_ID`, `echo $RG`, and so on, from Part 2.

> `dev.tfvars` has no secrets, so it is committed. `my_ip` and `alert_email` are passed as `TF_VAR_...` environment variables, from your shell locally and from GitHub secrets in the pipeline. That's how you keep values out of Git without a secret manager.

## 4.13 Apply the foundation (from your laptop)

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks/infra/foundation

export TF_VAR_my_ip=$(curl -s ifconfig.me)
export TF_VAR_alert_email="you@example.com"

terraform init -backend-config=backend.hcl
terraform fmt -recursive
terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan
terraform apply tfplan          # ~5-8 minutes
terraform output
```

Read the plan before you apply. Count the resources, and look for anything marked `-/+` (replace). That is what interviewers mean by "always review the plan".

## 4.14 Verify, and understand what you built

```bash
az network vnet list -g $RG -o table
az network vnet peering list -g $RG --vnet-name vnet-ttapp-dev-hub -o table     # peeringState = Connected
az network vnet subnet list -g $RG --vnet-name vnet-ttapp-dev-spoke -o table
az network nsg rule list -g $RG --nsg-name nsg-ttapp-dev-pe -o table
az network private-dns zone list -g $RG -o table
az network private-dns link vnet list -g $RG -z privatelink.vaultcore.azure.net -o table
az network private-endpoint list -g $RG -o table

# DNS from your laptop: Key Vault resolves to a PUBLIC IP (via privatelink CNAME)
nslookup $KV_NAME.vault.azure.net
```

Write down the CNAME chain from `nslookup`. You'll run the same lookup from the jumpbox in Part 5 and get **10.1.1.x**.

Also open the portal and look at the **Key Vault → Networking** blade (firewall + private endpoint connection) and the **private DNS zone → Recordsets** (the A record the endpoint created).

## 4.15 Commit

```bash
cd ~/three-tier-aks
git add infra/foundation && git commit -m "Foundation stack: network, DNS, ACR, Key Vault, monitoring, jumpbox" && git push
```

Do commit `.terraform.lock.hcl`. It pins the exact provider versions and checksums, so your laptop and the pipeline use the same provider.

(`.gitignore` comes in Part 6. Until then, make sure you are not committing `.terraform/` or `tfplan`: run `git status` before `git add`.)

---

[← Part 3 — Networking explained (read before Terraform)](03-networking-explained.md) | [Index](../README.md) | [Part 5 — Jumpbox and GitHub self-hosted runner →](05-jumpbox-and-runner.md)
