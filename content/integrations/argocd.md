# ArgoCD Integration

The **Hibernation Operator** provides optional, native integration with **ArgoCD**, enabling platform teams to apply hibernation schedules based on **ArgoCD AppProjects**. This allows hibernation policies to align with GitOps application boundaries rather than infrastructure namespaces—ideal for organizations using ArgoCD for declarative application delivery.

> 💡 **Note**: ArgoCD integration is **optional** and must be explicitly enabled during installation.

## How It Works

When a `ClusterResourceSupervisor` specifies `argocd.appProjects`, the Hibernation Operator:

1. **Discovers** all namespaces associated with the listed ArgoCD `AppProject`s (by reading the `AppProject.spec.destinations` field).
1. **Applies hibernation** (scaling `Deployments`/`StatefulSets` to 0) to workloads in those namespaces according to the defined `sleepSchedule` and `wakeSchedule`.
1. **Tracks state** in the CR’s `status.sleepingNamespaces` to ensure accurate restoration.

This means you can hibernate entire application portfolios (e.g., `frontend-team`, `data-platform`) with a single policy—without manually listing namespaces.

---

## Enabling ArgoCD Integration

### Step 1: Install Hibernation Operator with ArgoCD Support

During Helm installation, enable ArgoCD and specify the ArgoCD namespace:

```sh
helm install hibernation-operator oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-system \
  --create-namespace \
  --set argoCD.enabled=true \
  --set argoCD.namespace=argocd
```

> ✅ The operator requires **read-only access** to `AppProject` resources in the ArgoCD namespace.

### Step 2: Create a `ClusterResourceSupervisor` Targeting AppProjects

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
meta
  name: argocd-hibernation-policy
spec:
  argocd:
    namespace: argocd                # ← Namespace where ArgoCD is installed
    appProjects:                     # ← List of AppProject names
      - frontend-team
      - mobile-apps
      - data-platform
  schedule:
    sleepSchedule: "0 18 * * 1-5"   # Sleep weekdays at 6 PM UTC
    wakeSchedule: "0 8 * * 1-5"     # Wake weekdays at 8 AM UTC
```

Apply it:

```sh
kubectl apply -f cluster-resource-supervisor-argocd.yaml
```

### Step 3: Verify Targeted Namespaces

Check the status to see which namespaces are being watched:

```sh
kubectl get clusterresourcesupervisor argocd-hibernation-policy -o jsonpath='{.status.watchedNamespaces}'
```

Example output:

```json
["frontend-dev", "mobile-staging", "data-platform-prod"]
```

These are the namespaces defined in the `destinations` of the specified AppProjects.

---

## Key Benefits

| Benefit | Description |
|--------|-------------|
| **GitOps-Aligned Hibernation** | Hibernation follows application ownership (via AppProjects), not manual namespace lists. |
| **Dynamic Namespace Discovery** | New namespaces added to an AppProject are **automatically included** in hibernation. |
| **No Duplication** | Avoid maintaining separate namespace lists in both ArgoCD and hibernation policies. |
| **Safe & Observable** | Original replica counts are preserved; status shows exactly which apps are sleeping. |

---

## Requirements & Permissions

- ArgoCD must be installed in the cluster (typically in the `argocd` namespace).
- The Hibernation Operator needs the following RBAC (automatically included when `argoCD.enabled=true`):

```yaml
- apiGroups: ["argoproj.io"]
  resources: ["appprojects"]
  verbs: ["get", "list", "watch"]
```

> 🔒 The operator **never modifies** ArgoCD resources—it only reads `AppProject` definitions.

---

## Combining with Other Targeting Methods

You can **combine** ArgoCD targeting with explicit namespaces or label selectors:

```yaml
spec:
  argocd:
    namespace: argocd
    appProjects: ["frontend-team"]
  namespaces:
    names: ["legacy-staging"]
    labelSelector:
      matchLabels:
        env: dev
```

In this case, hibernation applies to:

- All namespaces in the `frontend-team` AppProject **plus**
- The `legacy-staging` namespace **plus**
- Any namespace with label `env=dev`

> 🔄 The union of all targeting methods is used.

---

## Troubleshooting Tips

- **AppProject not found?** Ensure the `appProjects` names **exactly match** the `AppProject` resource names (case-sensitive).
- **Namespace not hibernating?** Verify the namespace appears in `AppProject.spec.destinations`.
- **Permission errors?** Check that the operator’s ServiceAccount has `get/list/watch` on `appprojects.argoproj.io`.
