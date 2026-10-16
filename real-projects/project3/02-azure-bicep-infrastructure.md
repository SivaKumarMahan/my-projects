# 02 — Azure Infrastructure in Bicep

Each Azure resource has its own Bicep template and `.bicepparam` file. Resources were first built in the portal for `dev`, **exported as ARM templates** (App Service, App Service Plan, Storage, Key Vault, App Insights, Private Endpoint, MySQL, VM, NIC, Log Analytics, VNets, OpenAI, Static Web Apps, AI Translator) and then rewritten as clean, parameterised Bicep.

## Conventions

- **Naming:** `<type>-<organisation>-<environment>[-<service>]`, e.g. `vn-<org>-dev`, `wa-<org>-dev-backend`, `mysql-<org>-dev`, `swa-<org>-dev-admin`.
- **Tags on everything:** `Name`, `Company`, `Environment`.
- **`@allowed` environments:** `dev`, `stage`, `prod` (plus `devops` where needed) — a typo fails at compile time.
- **Per-environment maps in params:** one param file holds values for all environments and the template picks with `map[environment]` (VNet CIDRs, private IPs).
- **`existing` references** instead of IDs passed around, e.g. the App Service, VNet or hub VNet looked up by naming convention.

## Deployment order and dependencies

| # | Resource | Depends on | Manual step |
|---|---|---|---|
| 1 | Virtual network (3 public + 3 private subnets, peering to hub) | — | — |
| 2 | MySQL Flexible Server + private endpoint + private DNS zone | VNet | Confirm the environment VNet exists |
| 3 | App Service Plan → App Service (VNet integrated) → access restriction (VPN only) | VNet | Confirm the VNet exists |
| 4 | Static Web App | — | — |
| 5 | Virtual machine (public IP, NSG, NIC) | VNet | Confirm the VNet exists |
| 6 | Log Analytics + Application Insights + diagnostic settings | App Service | Confirm the App Service exists |
| 7 | AI Translator | — | — |
| 8 | Azure OpenAI account → model deployments | — | — |
| 9 | Storage account → blob container | — | — |
| 10 | Key Vault | — | Upload secrets after creation |

```bash
az deployment group create \
  --resource-group <rg> \
  --template-file <template>.bicep \
  --parameters <template>.bicepparam
```

## Virtual network: computed subnets and hub peering

```bicep
var addressSpace = vNetAddressSpace[environment]          // dev 10.1.0.0/16, stage 10.2.0.0/16, prod 10.3.0.0/16
var subnetPrefix = [for i in range(0, 6): cidrSubnet(addressSpace, subnetMask, i)]

var publicSubnets  = [for i in range(0, 3): { name: 'vn-${organisationName}-${environment}-public-subnet-${i + 1}',  addressPrefix: subnetPrefix[i] }]
var privateSubnets = [for i in range(3, 3): { name: 'vn-${organisationName}-${environment}-private-subnet-${i - 2}', addressPrefix: subnetPrefix[i] }]

resource vnet 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: 'vn-${organisationName}-${environment}'
  location: location
  properties: {
    addressSpace: { addressPrefixes: [addressSpace] }
    subnets: [for subnet in concat(publicSubnets, privateSubnets): {
      name: subnet.name
      properties: {
        addressPrefix: subnet.addressPrefix
        defaultOutboundAccess: contains(subnet.name, 'public')
        delegations: contains(subnet.name, 'public-subnet-1') ? [
          { name: 'appservice', properties: { serviceName: 'Microsoft.Web/serverFarms' } }
        ] : []
      }
    }]
  }
}
```

- **`cidrSubnet()`** carves six equal subnets from the environment CIDR — no hand-typed ranges.
- **Public subnet 1 is delegated to `Microsoft.Web/serverFarms`** for App Service VNet integration.
- **`defaultOutboundAccess`** only on public subnets; private subnets have no implicit internet egress.
- **Peering both ways** with the shared hub VNet (`allowVirtualNetworkAccess`, `allowForwardedTraffic`), so VPN users reach the environment.

> Review notes: the subnet mask parameter must be the new prefix length (e.g. `24`), not a count; and the peering was declared both inline in the VNet and as a child resource — declare it once (child resource) to avoid conflicting updates.

## MySQL Flexible Server with private access only

```
App Service (VNet integrated) ──► private DNS zone ──► A record ──► private endpoint NIC (fixed IP in private-subnet-2) ──► MySQL
```

- `publicNetworkAccess: 'Disabled'`.
- **Private endpoint** in private subnet 2 with a **fixed IP per environment** (from a `privateIPAddress[environment]` map) and a custom NIC name.
- **Private DNS zone** linked to the VNet, **zone group** on the endpoint, and an **A record** to the endpoint IP.
- Admin password is a **`@secure()`** parameter.
- Versions allowed: `8.0.21`, `5.7`. Backup retention 7 days, geo-redundant backup off for non-prod.

```bicep
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-11-01' = {
  name: 'mysql-${organisationName}-${environment}-private-endpoint'
  location: location
  properties: {
    customNetworkInterfaceName: 'mysql-${organisationName}-${environment}-nic'
    ipConfigurations: [{
      name: 'mysql-ipconfig'
      properties: { privateIPAddress: privateIPAddress[environment], groupId: 'mysqlServer', memberName: 'mysqlServer' }
    }]
    subnet: { id: resourceId('Microsoft.Network/virtualNetworks/subnets', 'vn-${organisationName}-${environment}', 'vn-${organisationName}-${environment}-private-subnet-2') }
    privateLinkServiceConnections: [{
      name: 'mysql-plsc'
      properties: { privateLinkServiceId: mysqlServer.id, groupIds: ['mysqlServer'] }
    }]
  }
}
```

## App Service and Static Web App

- **App Service Plan:** Linux (`reserved: true`), B2 / Basic. **App Service:** `NODE|20-lts`, `virtualNetworkSubnetId` set to the delegated subnet.
- **Access restriction** resource makes the backend reachable **only through the VPN**.
- **Static Web App** (Standard) for the admin UI; build settings `appLocation: '/'`, `outputLocation: 'dist'`; outputs the default hostname.

## Monitoring: App Insights wired into the App Service from Bicep

- Workspace-based **Application Insights** on a **Log Analytics** workspace (30-day retention).
- **Diagnostic settings** on both App Insights and the App Service (`allLogs` + `AllMetrics`) to the workspace.
- The template writes the App Insights **app settings** onto the existing App Service (`Microsoft.Web/sites/config`), including the connection string and agent extension versions:

| Setting | Value / purpose |
|---|---|
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | From the App Insights resource |
| `ApplicationInsightsAgent_EXTENSION_VERSION` | `~3` (Linux agent) |
| `XDT_MicrosoftApplicationInsights_Mode` | `recommended` |
| Profiler / snapshot / instrumentation engine | Explicitly disabled or versioned |
| `slotSetting` | `true` on agent settings so they stay with the slot during swaps |

> Caution: deploying `sites/config` named `web` with an `appSettings` array **replaces all app settings**. Merge with existing settings (or manage every setting in the same template) so other values aren't wiped.

## Azure OpenAI, AI Translator, Storage, Key Vault, VM

- **OpenAI:** `kind: OpenAI`, S0, custom subdomain; two model deployments (`gpt-35-turbo-16k` version `0613`, `text-embedding-3-large` version `1`) with Standard SKU and capacity per environment.
- **AI Translator:** `kind: TextTranslation`, S1, custom subdomain.
- **Storage:** StorageV2, Standard_LRS, Cool tier, blob container `article-images-<env>`.
- **Key Vault:** standard SKU, soft delete 90 days, purge protection on, access policies passed as an array (one entry per principal).
- **VM for ChromaDB:** Ubuntu 22.04 LTS Gen2, B1s, static private IP, public IP, NSG rules generated from a list (SSH, HTTPS, vector DB port, VPN server port) with priorities `300 + index`.

```bicep
securityRules: [for (rule, i) in rules: {
  name: rule.name
  properties: {
    protocol: rule.protocol
    destinationPortRange: string(rule.port)
    sourceAddressPrefix: rule.source        // restrict to the VPN / hub range, never '*'
    sourcePortRange: '*'
    destinationAddressPrefix: '*'
    access: 'Allow'
    direction: 'Inbound'
    priority: 300 + i
  }
}]
```

## Interview Q&A

**How did you keep the database private?**
Public access disabled, a private endpoint with a fixed IP in a private subnet, a private DNS zone linked to the VNet with a zone group, and the App Service reaching it through VNet integration.

**Why one template per resource?**
Teams could deploy and change resources independently in dependency order while the platform was still evolving; later these can be composed into a main template with modules.

**How did you go from portal resources to IaC?**
Exported the ARM templates as a reference, then rewrote them in Bicep with parameters, naming conventions, `@allowed` values and environment maps instead of hard-coded values.

**What does `cidrSubnet()` give you?**
Deterministic subnet ranges computed from the VNet CIDR, so every environment gets the same layout with no hand-typed ranges.
