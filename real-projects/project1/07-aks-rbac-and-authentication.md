# 07 — AKS RBAC and Authentication

## Authentication / authorization modes

| Mode | Authentication | Authorization |
|---|---|---|
| Local accounts with Kubernetes RBAC | Local (cluster certificate) | Kubernetes RBAC |
| Entra ID + Azure RBAC | Entra ID | Azure role assignments |
| **Entra ID + Kubernetes RBAC** (chosen) | Entra ID | Kubernetes Roles / Bindings |

Entra ID + Kubernetes RBAC was chosen for full control over both identity (Entra groups) and in-cluster permissions (Roles, ClusterRoles, Bindings managed in Helm).

## Bicep implementation

Two variables in the AKS control-plane module drive RBAC/AAD:

1. **`defaultManagedClusterProperties`** — baseline for every cluster:
   - `enableRBAC` — default `true`.
   - `disableLocalAccounts` — default `false`.
2. **`managedClusterProperties`** — extends defaults when `aadAuthentication = true`, adding:

```bicep
aadProfile: {
  managed: true
  enableAzureRBAC: aksParams.?enableAzureRBAC ?? false
  adminGroupObjectIDs: aksParams.?aadAdminGroupObjectIDs ?? []
}
```

### Rules found during testing

- `aadAuthentication` false/unset → no AAD integration; `enableAzureRBAC` and admin groups are ignored.
- **Since Kubernetes 1.25, `disableLocalAccounts: true` requires AAD integration** — otherwise: *"Since Kubernetes version 1.25, disableLocalAccounts can only be set on an Azure AD integration-enabled cluster."*
- `enableRBAC` is **immutable** — set only at cluster creation.

| enableRBAC | disableLocalAccounts | aadAuthentication | enableAzureRBAC | Admin groups | Result |
|---|---|---|---|---|---|
| true | false | false | false | — | ✅ Local + K8s RBAC |
| true | true | true | true | — / group | ✅ Entra + Azure RBAC |
| true | true | true | false | — / group | ✅ Entra + K8s RBAC |
| true | false | true | false/true | — / group | ✅ |
| false | false | false | false/true | — | ✅ (no RBAC) |
| true / false | true | false | any | any | ❌ 1.25+ local-accounts error |

## Rollout steps

### 1. Security groups (manual)

One admin group per environment (`sandboxAksAdmin`, `performanceAksAdmin`, `productionAksAdmin`) with at least one owner. Created by a Global Admin / Privileged Role Admin.

Bicep/ARM cannot create Microsoft Graph objects (groups/users) as normal resources — create them manually or via `az rest` calls to Graph inside a script.

### 2. Cluster

Create/update AKS with **Entra ID authentication + Kubernetes RBAC** and attach the admin group to the *Cluster admin ClusterRoleBinding*. Deploy through the AKS Bicep stack.

### 3. Pipeline identity

- UAMI per environment (e.g. `sandboxAksRbacMi`) created by a Bicep stack.
- Note its object (principal) ID and reference it in the Helm chart's RoleBindings so the pipeline can create Kubernetes objects.
- ARM service connection (Managed identity, subscription + RG scope) per environment (e.g. `sandboxAksRbacSc`). Creating it adds a Contributor assignment on the RG for Azure auth.

### 4. RBAC Helm chart

Deploys ClusterRoles, ClusterRoleBindings, Roles and RoleBindings so users and the pipeline identity get the right access.

```bash
helm upgrade --install --kube-context=<cluster> rbac-aks ./helm/RBAC \
  -f ./helm/RBAC/values.yaml \
  -f ./helm/RBAC/values/env/<env>/values.yaml \
  -f ./helm/RBAC/values/rbac/aks/values.yaml
```

Chart contents:

- ClusterRoles / ClusterRoleBindings: `view-role`, `edit-role`, `role&bindings-role`
- Roles / RoleBindings in `default` and `dev`: `view-role`, `edit-role`
- Cluster-level resources: nodes, namespaces
- Namespace-level: pods, services, endpoints, HPA, jobs, cronjobs, daemonsets, deployments, replicasets, statefulsets, network policies, ingresses

## RBAC test cases

Groups: **Cluster Admin Group** (bound to cluster-admin) and **RBAC Group** (granted Kubernetes RBAC).

```bash
kubectl auth can-i list nodes        # view check
kubectl auth can-i create pods       # edit check
kubectl auth can-i list pods --namespace=dev
```

### User in RBAC Group

| # | Configuration | Result |
|---|---|---|
| 1 | ClusterRoleBinding `view` | list nodes ✅, create pods ❌ |
| 2 | ClusterRoleBinding `view` + `edit` | get nodes ✅ |
| 3 | RoleBinding `view` in default | list nodes ❌, list pods (default) ✅, edit configmaps ❌ |
| 4 | RoleBinding `view` + `edit` in default | create deployments (default) ✅, (dev) ❌ |
| 5 | `view` + `edit` only in dev | list pods default ❌, dev ✅ |
| 6 | `edit` in default, `view` in dev | get pods default ✅, update deployments dev ❌ |

### User in Cluster Admin Group

Every configuration (view, edit, namespaced bindings) → full access (`delete nodes` ✅). **cluster-admin overrides any narrower binding.**

| Membership | Result |
|---|---|
| Neither group, cluster in local-account mode | admin by default |
| Neither group, Entra-only | no permissions |

Without the right permissions, k9s cannot list resources either.

### Q&A from testing

- **Can a RoleBinding reference a ClusterRole?** Yes. The binding is namespace-scoped, so the ClusterRole's rules apply only in that namespace — define once, bind per namespace.
- **Can a managed identity be added to an Entra group?** Not like a user; groups hold users, service principals or groups. Reference the identity's principal ID directly in RoleBindings instead.
- **Can a Role be reused across namespaces?** No — Roles are namespace-scoped. Use ClusterRole + RoleBinding per namespace, or ClusterRoleBinding for all namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: view-custom }
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: view-custom-binding, namespace: namespace-a }
subjects:
- kind: Group
  name: <aad-group-object-id>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view-custom
  apiGroup: rbac.authorization.k8s.io
```

## Kubelogin lifecycle and session control

### Method 1 — certificate (admin kubeconfig)

```bash
az aks get-credentials --resource-group <rg> --name <cluster> --admin --overwrite-existing
```

- kubeconfig holds `clusters` (API server + CA), `users` (client cert + key), `contexts`, `current-context`.
- Cert valid ~2 years; not tied to any Entra user.
- **`az logout` does not stop kubectl** — the certificate is self-contained. A stolen kubeconfig is a long-lived credential.
- Revocation is cluster-wide and disruptive: `az aks rotate-certs -g <rg> -n <cluster>`.

### Method 2 — Entra ID tokens via kubelogin (recommended for humans)

- kubeconfig uses an `exec` plugin: `kubelogin get-token --login azurecli --server-id <aks-server-app>`.
- In `azurecli` mode the **refresh token** lives in the Azure CLI MSAL cache (`~/.azure/msal_token_cache.json`); kubelogin caches the short-lived **access token** (~1 hour) under `~/.kube/cache/kubelogin/`.

### Revocation tests

| Test | Observation |
|---|---|
| Revoke sessions only | kubectl keeps working until the access token expires; after re-login the user is still in the group, so access returns. Revoking sessions alone does not remove authorization. |
| Revoke sessions + remove from group | Next token refresh is denied — access ends within ≤1 hour. |
| Delete the user | Immediate, absolute failure at login. |
| Conditional Access *Sign-in frequency = 1 hour* on the AKS AAD server app | Forces re-authentication every hour; after group removal, access ends at the next forced sign-in. Works independently of manual revocation. |

Automating removal:

```bash
az ad group member remove --group "$GROUP_NAME" --member-id "$USER_OBJECT_ID"
```

### Privileged Identity Management (PIM) for just-in-time access

- Users are **eligible**, not permanent, members of the AKS admin group.
- They **activate** for a limited time (e.g. max 8 hours) with justification and MFA.
- Full audit history of who activated what, when and why.
- Workflow: onboard the group to PIM → set member role policy (duration, MFA, justification) → make users eligible instead of direct members.

## RBAC Helm chart internals

**Files:** `helm/RBAC/values.yaml`, `helm/RBAC/values/rbac/aks/values.yaml`, `helm/RBAC/templates/resources/`, `helm/RBAC/templates/tpl/`, `helm/RBAC/README.md`

- **Layered values** — `values.yaml` (base roles and rules) → `values/env/<env>/values.yaml` (environment and kube context) → `values/rbac/aks/values.yaml` (bindings and subjects, highest precedence).
- **`enabled` toggle on every role and binding**, so access is switched on or off per environment without deleting code.
- **Reusable partials** — `_rbac.tpl` renders rules; `_role.tpl`, `_roleBinding.tpl`, `_clusterRole*.tpl` render objects.
- Templates `range` over maps and lists (skipping the `enabled` key), so adding a role is just a new values entry.
- **Subjects are Entra ID group object IDs** (`kind: Group`), tying Kubernetes RBAC to Entra groups.
- **Release-name guard** — templates render only when `Release.Name == "rbac-aks"`, preventing accidental installs under another name.
- Pattern: **view** (get/list/watch) and **edit** (CRUD) roles per namespace, plus cluster-scoped **view** for nodes and namespaces.
- `values.schema.json` rejects invalid verbs and environments (see [15](15-pre-commit-hooks-and-shift-left-validation.md)).

```yaml
# values/rbac/aks/values.yaml
roleBindings:
  enabled: true
  dev-edit:
    enabled: true
    namespace: dev
    roleRef: { kind: Role, name: edit-role }
    subjects:
      - kind: Group
        name: <entra-group-object-id>
```

```yaml
{{- if eq .Release.Name "rbac-aks" }}
{{- range $name, $rb := .Values.roleBindings }}
{{- if and (ne $name "enabled") $rb.enabled }}
{{ include "rbac.roleBinding" (dict "name" $name "rb" $rb) }}
{{- end }}
{{- end }}
{{- end }}
```

### More interview Q&A

**Azure RBAC for Kubernetes vs Kubernetes RBAC?**
With `enableAzureRBAC`, Azure role assignments decide authorization. Without it, Entra handles authentication and native Kubernetes RBAC (this chart) handles authorization.

**How do you audit who has access?**
Everything is in git and reviewed in pull requests; `helm template` shows the exact Roles and Bindings that will be applied.
