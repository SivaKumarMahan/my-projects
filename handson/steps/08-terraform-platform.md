# Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL)

[← Part 7 — Multistage Dockerfiles and local scans](07-dockerfiles-and-local-scans.md) | [Index](../README.md) | [Part 9 — The Helm chart →](09-helm-chart.md)

---

Create these files in `infra/platform/`.

## 8.1 `versions.tf`, `backend.hcl`, `variables.tf`, `main.tf`

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.11"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.7"
    }
    time = {
      source  = "hashicorp/time"
      version = "~> 0.13"
    }
  }

  backend "azurerm" {}
}

provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id
  resource_provider_registrations = "none"
  storage_use_azuread             = true
}
```

```hcl
resource_group_name  = "rg-your-existing-rg" # echo $RG
storage_account_name = "sttfstatesk01"       # echo $TFSTATE_SA
container_name       = "tfstate"
key                  = "platform.tfstate"
use_azuread_auth     = true
```

```hcl
variable "subscription_id" { type = string }
variable "resource_group_name" { type = string }
variable "location" { type = string }

variable "prefix" {
  type    = string
  default = "ttapp"
}

variable "suffix" { type = string }

variable "tfstate_storage_account" {
  type        = string
  description = "Storage account that holds the Terraform state (from Part 2)."
}

variable "kubernetes_version" {
  type        = string
  description = "Pick one version BELOW the latest from 'az aks get-versions' so you can practise an upgrade."
}

variable "system_vm_size" {
  type    = string
  default = "Standard_D2s_v5"
}

variable "user_vm_size" {
  type    = string
  default = "Standard_D2s_v5"
}

variable "aks_admin_object_ids" {
  type        = list(string)
  description = "Users who get 'Azure Kubernetes Service RBAC Cluster Admin' (you)."
}

variable "github_app_principal_id" {
  type        = string
  description = "Principal ID of id-gh-app. Gets deploy rights on the 'app' namespace."
}

variable "app_namespace" {
  type    = string
  default = "app"
}

variable "backend_service_account" {
  type    = string
  default = "backend"
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

```hcl
data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

# Read the outputs of the foundation stack from its state file.
data "terraform_remote_state" "foundation" {
  backend = "azurerm"
  config = {
    resource_group_name  = var.resource_group_name
    storage_account_name = var.tfstate_storage_account
    container_name       = "tfstate"
    key                  = "foundation.tfstate"
    use_azuread_auth     = true
  }
}

locals {
  rg       = data.azurerm_resource_group.rg.name
  location = var.location
  name     = "${var.prefix}-dev"
  f        = data.terraform_remote_state.foundation.outputs
}
```

`terraform_remote_state` reads the **outputs** of the foundation state file. The platform stack never changes foundation resources; it only references their IDs. (A common alternative is a `data` source for each resource by name.)

## 8.2 Application Gateway + WAF — `appgw.tf`

```hcl
resource "azurerm_public_ip" "appgw" {
  name                = "pip-${local.name}-appgw"
  resource_group_name = local.rg
  location            = local.location
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_web_application_firewall_policy" "waf" {
  name                = "waf-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  policy_settings {
    enabled                     = true
    mode                        = "Prevention" # "Detection" only logs
    request_body_check          = true
    max_request_body_size_in_kb = 128
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }
  }

  # A custom rule you can test with: curl -A "BadBot" http://<appgw-ip>/
  custom_rules {
    name      = "BlockBadBot"
    priority  = 1
    rule_type = "MatchRule"
    action    = "Block"

    match_conditions {
      match_variables {
        variable_name = "RequestHeaders"
        selector      = "User-Agent"
      }
      operator           = "Contains"
      negation_condition = false
      match_values       = ["BadBot"]
      transforms         = ["Lowercase"]
    }
  }
}

locals {
  # AGIC overwrites these names; we only need a valid starting configuration.
  appgw_fe_ip    = "appgw-fe-public"
  appgw_fe_port  = "port-80"
  appgw_pool     = "default-pool"
  appgw_settings = "default-settings"
  appgw_listener = "default-listener"
  appgw_rule     = "default-rule"
}

resource "azurerm_application_gateway" "appgw" {
  name                = "agw-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  firewall_policy_id  = azurerm_web_application_firewall_policy.waf.id
  tags                = var.tags

  sku {
    name = "WAF_v2"
    tier = "WAF_v2"
  }

  autoscale_configuration {
    min_capacity = 0
    max_capacity = 2
  }

  gateway_ip_configuration {
    name      = "appgw-ipconfig"
    subnet_id = local.f.snet_appgw_id
  }

  frontend_ip_configuration {
    name                 = local.appgw_fe_ip
    public_ip_address_id = azurerm_public_ip.appgw.id
  }

  frontend_port {
    name = local.appgw_fe_port
    port = 80
  }

  backend_address_pool {
    name = local.appgw_pool
  }

  backend_http_settings {
    name                  = local.appgw_settings
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 30
  }

  http_listener {
    name                           = local.appgw_listener
    frontend_ip_configuration_name = local.appgw_fe_ip
    frontend_port_name             = local.appgw_fe_port
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = local.appgw_rule
    rule_type                  = "Basic"
    http_listener_name         = local.appgw_listener
    backend_address_pool_name  = local.appgw_pool
    backend_http_settings_name = local.appgw_settings
    priority                   = 19000
  }

  # AGIC owns these blocks after the first deploy.
  # Without ignore_changes every "terraform apply" would wipe the AGIC config and cause an outage.
  lifecycle {
    ignore_changes = [
      backend_address_pool,
      backend_http_settings,
      frontend_port,
      http_listener,
      probe,
      redirect_configuration,
      request_routing_rule,
      ssl_certificate,
      url_path_map,
      rewrite_rule_set,
      tags,
    ]
  }
}
```

Two things are important here:

1. **App Gateway cannot be created empty.** Terraform gives it one dummy listener, rule and pool. After that, **AGIC owns the configuration**: it adds listeners, rules, pools and probes from your Ingress objects.
2. **`ignore_changes`.** Without it, the next `terraform apply` would "fix" the drift by deleting everything AGIC created, and the site would go down. This is a known AGIC + Terraform problem and a good interview story.

`autoscale_configuration` with `min_capacity = 0` keeps the cost at the fixed hourly price only. Production would use `min_capacity >= 2` for zone redundancy.

## 8.3 AKS — `aks.tf`

```hcl
# ---------- Control-plane identity ----------
# A custom private DNS zone needs a user-assigned identity that we can grant rights to
# BEFORE the cluster is created.
resource "azurerm_user_assigned_identity" "aks" {
  name                = "id-${local.name}-aks"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags
}

resource "azurerm_role_assignment" "aks_dns" {
  scope                = local.f.private_dns_zone_ids["aks"]
  role_definition_name = "Private DNS Zone Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "aks_network" {
  scope                = local.f.spoke_vnet_id
  role_definition_name = "Network Contributor"
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
  principal_type       = "ServicePrincipal"
}

resource "time_sleep" "wait_for_aks_rbac" {
  depends_on      = [azurerm_role_assignment.aks_dns, azurerm_role_assignment.aks_network]
  create_duration = "60s"
}

# ---------- Cluster ----------
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  kubernetes_version  = var.kubernetes_version
  sku_tier            = "Free"
  node_resource_group = "${local.rg}-aks-nodes" # AKS creates this RG itself
  tags                = var.tags

  # --- Private API server ---
  private_cluster_enabled             = true
  private_dns_zone_id                 = local.f.private_dns_zone_ids["aks"]
  private_cluster_public_fqdn_enabled = false
  dns_prefix_private_cluster          = local.name

  # --- Identity and access ---
  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }

  local_account_disabled            = true # no static admin kubeconfig
  role_based_access_control_enabled = true
  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true # Kubernetes permissions come from Azure role assignments
    tenant_id          = data.azurerm_client_config.current.tenant_id
  }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  # --- Nodes ---
  node_provisioning_profile {
    mode = "Manual" # classic node pools (not Node Auto Provisioning)
  }

  default_node_pool {
    name                         = "system"
    vm_size                      = var.system_vm_size
    orchestrator_version         = var.kubernetes_version
    node_count                   = 1
    vnet_subnet_id               = local.f.snet_aks_id
    max_pods                     = 30
    only_critical_addons_enabled = true # taint: CriticalAddonsOnly=true:NoSchedule
    temporary_name_for_rotation  = "systemtmp"

    upgrade_settings {
      max_surge = "33%"
    }
  }

  # --- Networking ---
  network_profile {
    network_plugin    = "azure"  # Azure CNI: pods get real VNet IPs (needed for AGIC)
    network_policy    = "calico" # enforces Kubernetes NetworkPolicy objects
    service_cidr      = "10.100.0.0/16"
    dns_service_ip    = "10.100.0.10"
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
  }

  # --- Add-ons ---
  ingress_application_gateway {
    gateway_id = azurerm_application_gateway.appgw.id
  }

  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  oms_agent {
    log_analytics_workspace_id      = local.f.log_analytics_workspace_id
    msi_auth_for_monitoring_enabled = true
  }

  workload_autoscaler_profile {
    vertical_pod_autoscaler_enabled = true
  }

  node_os_upgrade_channel = "NodeImage"

  depends_on = [time_sleep.wait_for_aks_rbac]
}

resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  mode                  = "User"
  vm_size               = var.user_vm_size
  orchestrator_version  = var.kubernetes_version
  vnet_subnet_id        = local.f.snet_aks_id
  max_pods              = 30
  auto_scaling_enabled  = true # Cluster Autoscaler
  min_count             = 1
  max_count             = 3
  node_labels           = { "workload" = "apps" }
  tags                  = var.tags

  upgrade_settings {
    max_surge = "33%"
  }

  lifecycle {
    ignore_changes = [node_count] # the autoscaler changes it
  }
}

# ---------- Role assignments the cluster needs ----------
# Kubelet pulls images from ACR (replaces "az aks update --attach-acr").
resource "azurerm_role_assignment" "kubelet_acr_pull" {
  scope                = local.f.acr_id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_kubernetes_cluster.aks.kubelet_identity[0].object_id
  principal_type       = "ServicePrincipal"
}

# AGIC add-on identity: manage the gateway, read the RG, join the gateway subnet.
locals {
  agic_principal_id = azurerm_kubernetes_cluster.aks.ingress_application_gateway[0].ingress_application_gateway_identity[0].object_id
}

resource "azurerm_role_assignment" "agic_appgw" {
  scope                = azurerm_application_gateway.appgw.id
  role_definition_name = "Contributor"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "agic_rg_reader" {
  scope                = data.azurerm_resource_group.rg.id
  role_definition_name = "Reader"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "agic_subnet" {
  scope                = local.f.snet_appgw_id
  role_definition_name = "Network Contributor"
  principal_id         = local.agic_principal_id
  principal_type       = "ServicePrincipal"
}

# You: full Kubernetes admin through Entra ID.
resource "azurerm_role_assignment" "aks_admins" {
  for_each             = toset(var.aks_admin_object_ids)
  scope                = azurerm_kubernetes_cluster.aks.id
  role_definition_name = "Azure Kubernetes Service RBAC Cluster Admin"
  principal_id         = each.value
}

# GitHub app pipeline: may fetch a kubeconfig, and is admin ONLY inside the app namespace.
resource "azurerm_role_assignment" "github_cluster_user" {
  scope                = azurerm_kubernetes_cluster.aks.id
  role_definition_name = "Azure Kubernetes Service Cluster User Role"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}

resource "azurerm_role_assignment" "github_namespace_admin" {
  scope                = "${azurerm_kubernetes_cluster.aks.id}/namespaces/${var.app_namespace}"
  role_definition_name = "Azure Kubernetes Service RBAC Admin"
  principal_id         = var.github_app_principal_id
  principal_type       = "ServicePrincipal"
}

# ---------- Container Insights data collection rule ----------
resource "azurerm_monitor_data_collection_rule" "ci" {
  name                = "MSCI-${local.location}-aks-${local.name}"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags

  destinations {
    log_analytics {
      name                  = "ciworkspace"
      workspace_resource_id = local.f.log_analytics_workspace_id
    }
  }

  data_flow {
    streams      = ["Microsoft-ContainerInsights-Group-Default"]
    destinations = ["ciworkspace"]
  }

  data_sources {
    extension {
      name           = "ContainerInsightsExtension"
      extension_name = "ContainerInsights"
      streams        = ["Microsoft-ContainerInsights-Group-Default"]
      extension_json = jsonencode({
        dataCollectionSettings = {
          interval               = "1m"
          namespaceFilteringMode = "Off"
          enableContainerLogV2   = true
        }
      })
    }
  }
}

resource "azurerm_monitor_data_collection_rule_association" "ci" {
  name                    = "ContainerInsightsExtension"
  target_resource_id      = azurerm_kubernetes_cluster.aks.id
  data_collection_rule_id = azurerm_monitor_data_collection_rule.ci.id
}
```

Walk through it like you would in an interview:

| Setting | What it does | Interview question it answers |
|---|---|---|
| `private_cluster_enabled` + custom `private_dns_zone_id` | API server has only a private IP, name in our zone | "Have you created a private AKS? How?" |
| user-assigned control-plane identity + 2 role assignments **before** the cluster | AKS needs rights on the DNS zone and VNet during creation | "service principal vs managed identity", "system vs user-assigned" |
| `local_account_disabled` + `azure_rbac_enabled` | no static admin kubeconfig; every `kubectl` call is an Entra ID identity checked against Azure roles | "How do you authenticate to AKS?", "local accounts in Kubernetes" |
| `oidc_issuer_enabled` + `workload_identity_enabled` | pods can exchange their service-account token for an Entra token | "How does AKS connect to Key Vault?" |
| system pool with `only_critical_addons_enabled` | taint `CriticalAddonsOnly=true:NoSchedule`, so only system pods run there | taints/tolerations, node pools |
| user pool `auto_scaling_enabled`, 1–3 nodes | **Cluster Autoscaler** adds nodes when pods are Pending | "How do you increase nodes automatically?" |
| `network_plugin = "azure"` | Azure CNI: pods get VNet IPs. AGIC **needs** routable pod IPs | "Which network plugin do you use? Which component gives a pod its IP?" (the CNI plugin, called by kubelet) |
| `network_policy = "calico"` | enforces NetworkPolicy objects | "What are network plugins used for?" |
| `ingress_application_gateway` | AGIC add-on, managed by AKS | Ingress controller |
| `key_vault_secrets_provider` | Secrets Store CSI driver + Azure provider, rotation every 2 min | CSI, secrets in pods |
| `oms_agent` + DCR | Container Insights (logs + inventory to Log Analytics) | "How do AKS logs reach Azure Monitor?" |
| `vertical_pod_autoscaler_enabled` | VPA add-on (used in recommendation mode) | HPA vs VPA vs CA |
| `max_surge = "33%"` | during an upgrade, add 1 extra node before draining an old one | zero-downtime upgrades |
| kubelet identity → **AcrPull** | nodes pull images from ACR without passwords | "ImagePullBackOff 401" |
| AGIC identity → Contributor on App Gateway, Reader on RG, Network Contributor on its subnet | AGIC must update App Gateway through ARM | AGIC troubleshooting |
| GitHub app identity → Cluster User + RBAC Admin **on namespace `app`** | the pipeline can deploy into `app` and nowhere else | least privilege for CI/CD |

> The azurerm docs note that AGIC is a *non-critical* add-on. With `only_critical_addons_enabled` on the system pool, the AGIC pod needs a **user** node pool to run on. Our `user` pool is required, not optional.

## 8.4 PostgreSQL + private endpoint — `postgres.tf`

```hcl
resource "random_password" "pg" {
  length           = 24
  special          = true
  override_special = "!#%*-_=+"
}

resource "azurerm_postgresql_flexible_server" "pg" {
  name                          = "psql-${var.prefix}-${var.suffix}"
  resource_group_name           = local.rg
  location                      = local.location
  version                       = "16"
  sku_name                      = "B_Standard_B1ms" # cheapest burstable tier
  storage_mb                    = 32768
  backup_retention_days         = 7
  administrator_login           = "pgadmin"
  administrator_password        = random_password.pg.result
  public_network_access_enabled = false # only reachable through the private endpoint
  tags                          = var.tags

  lifecycle {
    ignore_changes = [zone] # Azure picks a zone; do not fight it
  }
}

resource "azurerm_postgresql_flexible_server_database" "app" {
  name      = "appdb"
  server_id = azurerm_postgresql_flexible_server.pg.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

resource "azurerm_private_endpoint" "pg" {
  name                = "pe-${local.name}-psql"
  resource_group_name = local.rg
  location            = local.location
  subnet_id           = local.f.snet_pe_id
  tags                = var.tags

  private_service_connection {
    name                           = "psc-psql"
    private_connection_resource_id = azurerm_postgresql_flexible_server.pg.id
    subresource_names              = ["postgresqlServer"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "default"
    private_dns_zone_ids = [local.f.private_dns_zone_ids["postgres"]]
  }
}

# The backend reads these from Key Vault through the CSI driver.
locals {
  db_secrets = {
    "db-url"      = "jdbc:postgresql://${azurerm_postgresql_flexible_server.pg.fqdn}:5432/${azurerm_postgresql_flexible_server_database.app.name}?sslmode=require"
    "db-username" = azurerm_postgresql_flexible_server.pg.administrator_login
    "db-password" = random_password.pg.result
  }
}

resource "azurerm_key_vault_secret" "db" {
  for_each     = local.db_secrets
  name         = each.key
  value        = each.value
  key_vault_id = local.f.key_vault_id
}
```

- `public_network_access_enabled = false` + a **private endpoint**: the server has no public reachability at all.
- The private endpoint's `private_dns_zone_group` writes the A record into `privatelink.postgres.database.azure.com`.
- The password is generated by Terraform and written straight into Key Vault. You never type it. It is in the Terraform **state**, which is why the state account is locked down (Entra-only, no keys, versioned).
- The other model is **VNet integration** (a delegated subnet). Know the difference: VNet integration puts the server *inside* a subnet you dedicate to it; a private endpoint puts only a NIC in your subnet, and the server stays a PaaS service.

## 8.5 Workload Identity for the backend — `workload-identity.tf`

```hcl
# Identity used by the backend pods to read Key Vault (no passwords stored anywhere).
resource "azurerm_user_assigned_identity" "backend" {
  name                = "id-${local.name}-backend"
  resource_group_name = local.rg
  location            = local.location
  tags                = var.tags
}

# Trust: "a token from THIS cluster for THIS service account may act as this identity".
resource "azurerm_federated_identity_credential" "backend" {
  name                      = "fic-aks-backend"
  user_assigned_identity_id = azurerm_user_assigned_identity.backend.id
  audience                  = ["api://AzureADTokenExchange"]
  issuer                    = azurerm_kubernetes_cluster.aks.oidc_issuer_url
  subject                   = "system:serviceaccount:${var.app_namespace}:${var.backend_service_account}"
}

resource "azurerm_role_assignment" "backend_kv" {
  scope                = local.f.key_vault_id
  role_definition_name = "Key Vault Secrets User" # read-only
  principal_id         = azurerm_user_assigned_identity.backend.principal_id
  principal_type       = "ServicePrincipal"
}
```

The federated credential says: *"Tokens issued by **this AKS cluster's OIDC issuer** for the service account **`app/backend`** may act as this managed identity."* No secret is involved. If someone deploys a pod with a different service account or namespace, it gets nothing.

## 8.6 Diagnostics — `diagnostics.tf`

```hcl
# Send logs to Log Analytics so you can query them with KQL.
# "Dedicated" = one table per log type (AGWFirewallLogs, AKSAuditAdmin, ...) instead of AzureDiagnostics.
resource "azurerm_monitor_diagnostic_setting" "appgw" {
  name                           = "diag-appgw"
  target_resource_id             = azurerm_application_gateway.appgw.id
  log_analytics_workspace_id     = local.f.log_analytics_workspace_id
  log_analytics_destination_type = "Dedicated"

  enabled_log {
    category_group = "allLogs" # access log, performance log, firewall (WAF) log
  }

  enabled_metric {
    category = "AllMetrics"
  }
}

resource "azurerm_monitor_diagnostic_setting" "aks" {
  name                           = "diag-aks"
  target_resource_id             = azurerm_kubernetes_cluster.aks.id
  log_analytics_workspace_id     = local.f.log_analytics_workspace_id
  log_analytics_destination_type = "Dedicated"

  # Only the useful, low-volume categories (the 1 GB/day cap is on).
  enabled_log {
    category = "kube-audit-admin" # who created/changed/deleted what (no read calls)
  }

  enabled_log {
    category = "cluster-autoscaler"
  }

  enabled_log {
    category = "guard" # Entra ID authentication decisions
  }
}
```

## 8.7 Alerts — `alerts.tf`

```hcl
# Metric alert: average node CPU above 80% for 15 minutes.
resource "azurerm_monitor_metric_alert" "node_cpu" {
  name                = "alert-${local.name}-node-cpu"
  resource_group_name = local.rg
  scopes              = [azurerm_kubernetes_cluster.aks.id]
  description         = "AKS node CPU above 80%"
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"
  tags                = var.tags

  criteria {
    metric_namespace = "Microsoft.ContainerService/managedClusters"
    metric_name      = "node_cpu_usage_percentage"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = local.f.action_group_id
  }
}

# Log alert: any container in CrashLoopBackOff in the last 10 minutes.
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "crashloop" {
  name                 = "alert-${local.name}-crashloop"
  resource_group_name  = local.rg
  location             = local.location
  scopes               = [local.f.log_analytics_workspace_id]
  description          = "Pods in CrashLoopBackOff"
  severity             = 2
  evaluation_frequency = "PT5M"
  window_duration      = "PT10M"
  tags                 = var.tags

  criteria {
    query                   = <<-KQL
      KubePodInventory
      | where ContainerStatusReason == "CrashLoopBackOff"
      | summarize Restarts = count() by Namespace, Name
    KQL
    time_aggregation_method = "Count"
    operator                = "GreaterThan"
    threshold               = 0

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  action {
    action_groups = [local.f.action_group_id]
  }
}
```

## 8.8 Outputs and values

```hcl
output "aks_name" { value = azurerm_kubernetes_cluster.aks.name }
output "aks_private_fqdn" { value = azurerm_kubernetes_cluster.aks.private_fqdn }
output "appgw_public_ip" { value = azurerm_public_ip.appgw.ip_address }
output "postgres_fqdn" { value = azurerm_postgresql_flexible_server.pg.fqdn }
output "backend_identity_client_id" { value = azurerm_user_assigned_identity.backend.client_id }
output "tenant_id" { value = data.azurerm_client_config.current.tenant_id }
```

```hcl
subscription_id         = "00000000-0000-0000-0000-000000000000" # echo $SUBSCRIPTION_ID
resource_group_name     = "rg-your-existing-rg"                  # echo $RG
location                = "centralindia"                         # echo $LOCATION
prefix                  = "ttapp"
suffix                  = "sk01"          # echo $SUFFIX
tfstate_storage_account = "sttfstatesk01" # echo $TFSTATE_SA

# One version BELOW the newest from: az aks get-versions -l $LOCATION -o table
kubernetes_version = "1.33"

aks_admin_object_ids    = ["11111111-1111-1111-1111-111111111111"] # echo $MY_OBJECT_ID
github_app_principal_id = "33333333-3333-3333-3333-333333333333"   # echo $APP_PRINCIPAL_ID
```

Pick the Kubernetes version:

```bash
az aks get-versions -l $LOCATION -o table
```

Choose the **second-newest** minor version (for example, if 1.34 is newest, use 1.33). In Part 16 you'll upgrade it, which answers "how did you upgrade AKS, and what challenges did you face?"

## 8.9 Apply

```bash
source ~/three-tier-env.sh
cd ~/three-tier-aks/infra/platform
terraform init -backend-config=backend.hcl
terraform validate
terraform plan -var-file=dev.tfvars -out=tfplan
terraform apply tfplan     # 15-25 minutes (AKS and App Gateway are slow)
terraform output
```

Run it from your **laptop** for this first time. Key Vault allows your IP, so Terraform can write the `db-*` secrets. Later the pipeline runs it from the runner, which reaches Key Vault through the private endpoint.

If something fails, read the error carefully and check [Appendix A](appendix-a-blockers.md). Typical first-run errors: quota, SKU not available in the region, a policy denying the node resource group, or role assignments not yet applied (just run `apply` again).

## 8.10 Verify from the jumpbox

```bash
ssh azureuser@$JUMPBOX_IP
az login --use-device-code        # if the session expired

# 1) Private API server: name resolves to a PRIVATE IP from the hub VNet
az aks show -g $RG -n $AKS_NAME --query "{fqdn:privateFqdn, k8s:kubernetesVersion, state:provisioningState}" -o table
nslookup $(az aks show -g $RG -n $AKS_NAME --query privateFqdn -o tsv)   # → 10.1.4.x

# 2) kubeconfig with Entra ID login (no local admin account exists)
az aks get-credentials -g $RG -n $AKS_NAME --overwrite-existing
kubelogin convert-kubeconfig -l azurecli
kubectl get nodes -o wide          # node IPs are 10.1.4.x (snet-aks)
kubectl get pods -A -o wide        # pod IPs are ALSO 10.1.4.x: that is Azure CNI

# 3) Add-ons running
kubectl get pods -n kube-system | grep -E "ingress-appgw|secrets-store|ama-"
kubectl get pods -A | grep -iE "calico|tigera"
kubectl describe node -l kubernetes.azure.com/mode=system | grep Taints    # CriticalAddonsOnly

# 4) PostgreSQL over the private endpoint
nslookup $PG_NAME.postgres.database.azure.com      # CNAME privatelink → 10.1.1.x
export PGPASSWORD=$(az keyvault secret show --vault-name $KV_NAME -n db-password --query value -o tsv)
psql "host=$PG_NAME.postgres.database.azure.com port=5432 dbname=appdb user=pgadmin sslmode=require" -c "select version();"

# 5) App Gateway is running
az network application-gateway show -g $RG -n $APPGW_NAME --query "{state:operationalState, sku:sku.name}" -o table
```

From your **laptop**, try `nslookup <private fqdn>` and `psql ...`. Both fail. That is the expected, secure result: the API server and the database are only reachable from inside the VNets.

Create the namespace the app will live in (your admin role can; the pipeline's namespace-scoped role cannot create namespaces):

```bash
kubectl create namespace app
```

```bash
# on your laptop
git add infra/platform && git commit -m "Platform stack: private AKS, App Gateway WAF, PostgreSQL" && git push
```

---

[← Part 7 — Multistage Dockerfiles and local scans](07-dockerfiles-and-local-scans.md) | [Index](../README.md) | [Part 9 — The Helm chart →](09-helm-chart.md)
