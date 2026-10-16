# 20 — Bicep Test Cases: Hands-On Lab Walkthrough

The lab behind the results in [03](03-bicep-testing-and-validation.md). It proves three things before Bicep was used for real environments:

1. A **single merged template** and **split per-resource templates** produce the same infrastructure, and redeploying the split templates over the merged deployment changes nothing.
2. AKS **control plane and node pool versions** can be changed independently through Bicep, and downgrades are rejected.
3. The deployed resources actually **work** (storage access rules, AKS nodes, LoadBalancer services, autoscaling, monitoring).

## Requirements

- **TestCase 1** — merge VNet, Storage Account and AKS into **one** template and deploy it. Every property in the template must appear in the portal.
- **TestCase 2** — deploy **individual** templates for the same resources **without removing** the TestCase 1 resources. Nothing should change in the portal; any change is noted and investigated.
- **No hard-coded values** in templates — everything comes from `.bicepparam` files, and the values are the same across all test cases.
- Check clean-up behaviour, including `az deployment sub delete` for subscription-scope deployments.

## Lab architecture

```
Resource group: bicepResourceGroup (East US)
├── VNet  bicepVNet 10.0.0.0/16
│   └── Subnet bicepSubnet 10.0.0.0/24 (private endpoint network policies disabled)
├── Storage account (StorageV2, Standard_LRS, Cool, TLS 1.2, network ACL default Deny, bypass AzureServices)
│   └── Private endpoint (groupId: blob) + NIC in bicepSubnet
├── User-assigned managed identity for AKS
└── AKS bicepAKSCluster
    ├── Azure CNI, service CIDR 10.96.0.0/12, DNS service IP 10.96.0.10
    ├── Kubernetes RBAC enabled
    ├── Private API server (TestCase 1.0) / public API server (TestCase 1.1 and 2.x)
    └── System node pool nodepool1: VMSS, Standard_D2s_v3, 2 nodes, autoscale 1–5
```

| Component | Purpose |
|---|---|
| VNet and subnet | Network where AKS nodes and the storage private endpoint live |
| Storage account | Locked down with private endpoint and network ACLs |
| Private endpoint | Storage reachable only from inside the VNet |
| AKS cluster | Nodes in the subnet, user-assigned managed identity |
| Private AKS cluster | No public API server access |

### Resources Azure created automatically

**In `bicepResourceGroup`:** VNet, storage account, storage private endpoint and its NIC, AKS cluster, AKS managed identity — plus a `kube-apiserver` private endpoint in the VNet when the cluster is private.

**In the node resource group `MC_bicepResourceGroup_bicepAKSCluster_eastus`:**

| Resource | Purpose |
|---|---|
| Private DNS zone `<guid>.privatelink.eastus.azmk8s.io` | Resolves the private API server |
| NSG `aks-agentpool-<id>-nsg` | Controls traffic to nodes |
| VM scale set `aks-nodepool1-<id>-vmss` | The worker nodes |
| Managed identity `<cluster>-agentpool` | Kubelet identity for Azure access |
| Public IP | Outbound traffic for the cluster |
| Private endpoint `kube-apiserver` + NIC | Private connectivity to the API server |
| Load balancer `kubernetes` | Outbound and `LoadBalancer` service traffic |

Deleting `bicepResourceGroup` removes everything, including the `MC_` group.

## TestCase 1.0 — single merged template

`main.bicep` declares all resources; `main.bicepparam` holds every value.

```bicep
param location string
param vnetName string
param subnetName string
param vnetAddressPrefix array
param subnetAddressPrefix array
param privateEndpointNetworkPolicies string
param storageAccountName string
param accountType string
param minimumTlsVersion string
param kind string
param accessTier string
param networkAclsBypass string
param networkAclsDefaultAction string
param privateEndpointName string
param privateLinkServiceConnectionName string
param groupIds array
param aksUserIdentity string
param aksClusterName string
param aksNodeCount int
param aksVmSize string
param aksKubernetesVersion string
param dnsPrefix string
param serviceCidr string
param dnsServiceIP string
param identityType string
param networkPlugin string
param agentPoolName string
param agentPoolType string
param agentPoolMode string
param enableRBAC bool
param enablePrivateCluster bool
param enableAutoScaling bool
param minCount int
param maxCount int

resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: { addressPrefixes: vnetAddressPrefix }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: subnetAddressPrefix[0]
          privateEndpointNetworkPolicies: privateEndpointNetworkPolicies
        }
      }
    ]
  }
}

resource storage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: storageAccountName
  location: location
  kind: kind
  sku: { name: accountType }
  properties: {
    minimumTlsVersion: minimumTlsVersion
    accessTier: accessTier
    networkAcls: {
      bypass: networkAclsBypass
      defaultAction: networkAclsDefaultAction
    }
  }
}

resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-04-01' = {
  name: privateEndpointName
  location: location
  properties: {
    privateLinkServiceConnections: [
      {
        name: privateLinkServiceConnectionName
        properties: {
          privateLinkServiceId: storage.id
          groupIds: groupIds
        }
      }
    ]
    subnet: { id: vnet.properties.subnets[0].id }
  }
}

resource aksIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: aksUserIdentity
  location: location
}

resource aks 'Microsoft.ContainerService/managedClusters@2023-04-01' = {
  name: aksClusterName
  location: location
  identity: {
    type: identityType
    userAssignedIdentities: { '${aksIdentity.id}': {} }
  }
  properties: {
    kubernetesVersion: aksKubernetesVersion
    dnsPrefix: dnsPrefix
    enableRBAC: enableRBAC
    networkProfile: {
      networkPlugin: networkPlugin
      serviceCidr: serviceCidr
      dnsServiceIP: dnsServiceIP
    }
    apiServerAccessProfile: { enablePrivateCluster: enablePrivateCluster }
    agentPoolProfiles: [
      {
        name: agentPoolName
        count: aksNodeCount
        vmSize: aksVmSize
        vnetSubnetID: vnet.properties.subnets[0].id
        type: agentPoolType
        mode: agentPoolMode
        enableAutoScaling: enableAutoScaling
        minCount: minCount
        maxCount: maxCount
      }
    ]
  }
}

output vnetId string = vnet.id
output storageId string = storage.id
output aksId string = aks.id
output aksIdentityId string = aksIdentity.id
```

```bicep
using './main.bicep'

param location = 'eastus'
param vnetName = 'bicepVNet'
param subnetName = 'bicepSubnet'
param vnetAddressPrefix = ['10.0.0.0/16']
param subnetAddressPrefix = ['10.0.0.0/24']
param privateEndpointNetworkPolicies = 'Disabled'
param storageAccountName = 'bicepstorage01'
param accountType = 'Standard_LRS'
param kind = 'StorageV2'
param accessTier = 'Cool'
param minimumTlsVersion = 'TLS1_2'
param networkAclsBypass = 'AzureServices'
param networkAclsDefaultAction = 'Deny'
param privateEndpointName = 'bicepStoragePE'
param privateLinkServiceConnectionName = 'storageConnection'
param groupIds = ['blob']
param aksUserIdentity = 'bicepAksIdentity'
param aksClusterName = 'bicepAKSCluster'
param aksNodeCount = 2
param aksVmSize = 'Standard_D2s_v3'
param aksKubernetesVersion = '1.29.0'
param dnsPrefix = 'bicepaksdns'
param serviceCidr = '10.96.0.0/12'
param dnsServiceIP = '10.96.0.10'
param identityType = 'UserAssigned'
param networkPlugin = 'azure'
param agentPoolName = 'nodepool1'
param agentPoolType = 'VirtualMachineScaleSets'
param agentPoolMode = 'System'
param enableRBAC = true
param enablePrivateCluster = true
param enableAutoScaling = true
param minCount = 1
param maxCount = 5
```

```bash
az login
az group create --name bicepResourceGroup --location eastus
az deployment group create --resource-group bicepResourceGroup \
  --template-file main.bicep --parameters main.bicepparam
az deployment operation group list --resource-group bicepResourceGroup --name main   # per-resource status
```

**Result:** all resources created with every property from the template.

### TestCase 1.1 — same template, ready for version tests

Two changes from 1.0:

- New parameter **`orchestratorVersion`** set on the node pool, so the control plane (`kubernetesVersion`) and node pool versions can be changed independently.
- **`enablePrivateCluster = false`**, so `kubectl` can reach the API server directly from a workstation during testing.

```bicep
agentPoolProfiles: [
  {
    name: agentPoolName
    // ...same as 1.0...
    orchestratorVersion: orchestratorVersion
  }
]
```

## TestCase 2.0 — split templates over the existing deployment

One template + parameter file per resource, deployed in dependency order on top of the TestCase 1 resources.

| File | Scope | Contents |
|---|---|---|
| `resourceGroup.bicep` | Subscription (`targetScope = 'subscription'`) | Resource group, location from `deployment().location` |
| `vnet.bicep` | Resource group | VNet + subnet; outputs `vnetId`, `subnetId` |
| `storage.bicep` | Resource group | Storage account + private endpoint; takes `subnetId` |
| `aks.bicep` | Resource group | Managed identity + AKS; takes `subnetId` |

```bicep
// resourceGroup.bicep
targetScope = 'subscription'
param resourceGroupName string

resource rg 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  name: resourceGroupName
  location: deployment().location
}
```

```bash
az deployment sub create --location eastus \
  --template-file resourceGroup.bicep --parameters resourceGroup.bicepparam
az deployment group create -g bicepResourceGroup --template-file vnet.bicep    --parameters vnet.bicepparam
az deployment group create -g bicepResourceGroup --template-file storage.bicep --parameters storage.bicepparam
az deployment group create -g bicepResourceGroup --template-file aks.bicep     --parameters aks.bicepparam
```

**Result:** each deployment succeeded and **nothing changed** in the existing resource group, VNet, storage account, private endpoint, AKS cluster or the `MC_` resources — the templates are idempotent whether merged or split.

**Lesson learned:** in the lab, `storage.bicepparam` and `aks.bicepparam` passed `subnetId` as a full resource ID string, which embeds the subscription ID — a hard-coded value the requirements wanted to avoid. Better options:

```bicep
// reference the existing VNet instead of passing an ID
resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = { name: vnetName }
var subnetId = '${vnet.id}/subnets/${subnetName}'

// or build it from names
var subnetId = resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, subnetName)
```

…or wire the VNet module's `subnetId` output straight into the storage and AKS modules from a parent template.

## TestCase 2.1–2.6 — AKS version changes

Only two parameters change in `aks.bicepparam` between runs:

| Case | `aksKubernetesVersion` | `orchestratorVersion` | Outcome |
|---|---|---|---|
| 1.1 | 1.29.0 | 1.29.0 | Baseline |
| 2.1 | 1.30.0 | 1.29.0 | Control plane only; nodes untouched, system pods rescheduled |
| 2.2 | 1.30.0 | 1.30.0 | Nodes recreated (same names), all pods recreated |
| 2.3 | 1.31.1 | 1.30.0 | Control plane only |
| 2.4 | 1.30.0 | 1.31.1 | ❌ `Downgrading Kubernetes version 1.31.1 to 1.30.0 is not allowed.` |
| 2.5 | 1.31.1 | 1.31.1 | Nodes recreated, all pods recreated |
| 2.6 | 1.31.1 | 1.30.0 | ❌ node pool downgrade not allowed |

```bash
az aks get-versions --location eastus --output table      # versions available in the region
az deployment group create -g bicepResourceGroup --template-file aks.bicep --parameters aks.bicepparam

# Verify both versions after each run
az aks show -g bicepResourceGroup -n bicepAKSCluster --query kubernetesVersion -o table
az aks nodepool show -g bicepResourceGroup --cluster-name bicepAKSCluster \
  --name nodepool1 --query orchestratorVersion -o table
```

Detailed observations for each case are in [03](03-bicep-testing-and-validation.md).

## What was checked during every run

| Check | Command |
|---|---|
| Nodes: status, version, IP, image, runtime | `kubectl get nodes -o wide` |
| Pods across namespaces and where they run | `kubectl get pods -A -o wide` (add `--watch` during upgrades) |
| Node conditions and capacity | `kubectl describe node <node>` |
| Shell into a pod | `kubectl exec -it <nginx-pod> -- /bin/sh` |
| Pod has outbound internet | `kubectl exec -it <nginx-pod> -- curl -I http://google.com` |
| Restarts (highest last) | `kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'` |
| Anything not Running | `kubectl get pods -A \| grep -v Running` (empty = healthy) |
| Pod detail and logs | `kubectl describe pod <pod> -n <ns>`, `kubectl logs <pod> -n <ns>` |
| Resource usage | `kubectl top nodes`, `kubectl top pods --all-namespaces` |
| Deployments healthy | `kubectl get deployments -A` |
| Delete / evict / schedule events | `kubectl get events -A --sort-by='.lastTimestamp' \| grep -i 'Deleted\|Evicted\|Scheduled'` |
| Reset node count | `az aks scale -g bicepResourceGroup -n bicepAKSCluster --nodepool-name nodepool1 --node-count 2` |

### Test workload

```bash
az aks get-credentials -g bicepResourceGroup -n bicepAKSCluster --overwrite-existing
kubectl create deployment nginx --image=nginx --port=80
kubectl expose deployment nginx --type=LoadBalancer --port=80 --target-port=80
kubectl get services            # wait for EXTERNAL-IP, then browse to it

# Generate load against the service
kubectl create deployment load-test --image=busybox -- \
  /bin/sh -c "while true; do wget -q -O- http://nginx.default.svc.cluster.local; done"
kubectl scale deployment load-test --replicas=10
```

`kubectl create deployment` does **not** accept `--replicas` together with a command in older kubectl versions, so scale it separately as above. (`kubectl run --replicas` was removed entirely.)

Exposing nginx as `LoadBalancer` made AKS add a new public frontend IP, a load-balancing rule and health probe on the `kubernetes` load balancer, and an `k8s-azure-lb_allow_IPv4_…` inbound rule on the node NSG.

### Working with the private cluster (TestCase 1.0)

With `enablePrivateCluster = true` the API server has no public endpoint, so local `kubectl` can't reach it. `az aks command invoke` runs commands inside the cluster through Azure instead:

```bash
az aks command invoke -g bicepResourceGroup -n bicepAKSCluster --command "kubectl get nodes"
az aks command invoke -g bicepResourceGroup -n bicepAKSCluster --command "kubectl create deployment nginx --image=nginx"
az aks command invoke -g bicepResourceGroup -n bicepAKSCluster --command "kubectl expose deployment nginx --port=80 --type=LoadBalancer"
az aks command invoke -g bicepResourceGroup -n bicepAKSCluster --command "kubectl get svc nginx"
```

## Functional tests on the deployed resources

| Resource | Test | Expected / observed |
|---|---|---|
| Storage | Create a container and upload files from outside the VNet | ❌ Blocked — the account and container are private (network ACL `Deny` + private endpoint) |
| Storage | Remove the private endpoint, set `allowBlobPublicAccess: true` and container `publicAccess: 'Container'` | ✅ Upload and download work — proves the lock-down was what blocked access |
| AKS | View node status and cluster health | Nodes `Ready`, metrics reflect the workload |
| AKS | Access nodes for troubleshooting | Node configuration and logs inspectable (`kubectl describe node`, `kubectl debug node/...`) |
| AKS | Deploy NGINX behind a `LoadBalancer` service | Default NGINX page served on the external IP |
| AKS | Logs and Container Insights in Azure Monitor | Pod events, errors and performance metrics visible |
| AKS | Generate load and watch the autoscaler | Extra nodes provisioned up to `maxCount` (5) |

## Clean-up

```bash
az aks delete --name bicepAKSCluster --resource-group bicepResourceGroup
az network vnet delete --name bicepVNet --resource-group bicepResourceGroup
az storage account delete --name bicepstorage01 --resource-group bicepResourceGroup
az group delete --name bicepResourceGroup --yes --no-wait        # removes everything, incl. MC_ group

az deployment group list --resource-group bicepResourceGroup -o table   # deployment history
az deployment sub delete --name <deployment-name>   # removes only the deployment RECORD, not the resources
```

`az deployment sub delete` / `az deployment group delete` delete the **deployment history entry**, not the resources it created. To remove resources, delete them or the resource group — or use Deployment Stacks, which track and delete managed resources ([02](02-iac-bicep-deployment-stacks.md)).

## Interview Q&A

**How did you prove the Bicep templates were safe to run against existing infrastructure?**
Deployed one merged template, then redeployed the same resources from split templates on top of it and confirmed in the portal and with what-if that nothing changed.

**Why separate `kubernetesVersion` and `orchestratorVersion`?**
`kubernetesVersion` is the control plane; `orchestratorVersion` is per node pool. Separating them lets you upgrade the control plane first and the pools later — the normal AKS upgrade order.

**What happens to workloads during each kind of upgrade?**
Control-plane-only: nodes stay, a few system pods (kube-proxy, CSI, konnectivity, cloud-node-manager) are rescheduled. Node pool: nodes are reimaged/replaced and every pod is recreated, so PDBs and replicas matter.

**How do you run kubectl against a private AKS cluster without a jump box?**
`az aks command invoke`, or a self-hosted agent / VM inside the VNet.

**Does `az deployment group delete` remove resources?**
No — only the deployment record. Resources stay until deleted directly, by deleting the resource group, or by a deployment stack with an `--action-on-unmanage` delete mode.
