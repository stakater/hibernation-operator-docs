# Setup ClusterResourceSupervisor

## Creating a `ClusterResourceSupervisor`

This guide explains how to create a **cluster-scoped** `ClusterResourceSupervisor` to manage hibernation across multiple namespaces or ArgoCD AppProjects.

### Prerequisites

- The **Hibernation Operator** must be [installed](../installation/kubernetes.md).
- You have **cluster-admin** permissions.

### Step 1: Choose a Targeting Strategy

You can target namespaces in **three ways** (use one or combine `names` + `labelSelector`):

| Method | Use Case |
|-------|--------|
| **Explicit namespace list** | Known, static namespaces |
| **Label selector** | Dynamic selection (e.g., all `env=dev`) |
| **ArgoCD AppProjects** | GitOps-aligned hibernation |

### Step 2: Create a `ClusterResourceSupervisor` YAML

#### Example A: Label-Based Targeting

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: dev-environments-hibernation
spec:
  namespaces:
    labelSelector:
      matchLabels:
        env: dev
  schedule:
    sleepSchedule: "0 18 * * 1-5"   # Weekdays at 6 PM UTC
    wakeSchedule: "0 8 * * 1-5"     # Weekdays at 8 AM UTC
```

#### Example B: ArgoCD AppProject Integration

> ✅ Ensure `argoCD.enabled=true` was set during [Helm install](../installation/kubernetes.md#optional-enable-argocd-integration)

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: argocd-frontend-hibernation
spec:
  argocd:
    namespace: argocd
    appProjects:
      - frontend-team
      - mobile-apps
  schedule:
    sleepSchedule: "0 22 * * *"     # Sleep at 10 PM UTC daily
    # No wakeSchedule → stay asleep until manually updated
```

#### Example C: Explicit Namespace List

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: specific-namespaces-hibernation
spec:
  namespaces:
    names:
      - team-a-staging
      - team-b-test
      - demo-env
  schedule:
    sleepSchedule: "0 0 * * 0"      # Every Sunday at midnight
    wakeSchedule: "0 0 * * 1"       # Every Monday at midnight
```

### Step 3: Apply the Resource

```sh
kubectl apply -f cluster-resource-supervisor.yaml
```

### Step 4: Verify Status

```sh
kubectl get clusterresourcesupervisor dev-environments-hibernation -o yaml
```

Key status fields:

- `status.watchedNamespaces`: List of namespaces being managed
- `status.sleepingNamespaces`: Details of scaled-down apps (with original replica counts)
- `status.currentStatus`: Overall state (`running`, `sleeping`, `error`)
- `status.nextReconcileTime`: Next scheduled action

### Notes

- The operator **merges** `names` and `labelSelector` results (union of both).
- If **both `namespaces` and `argocd`** are defined, the operator targets **all namespaces from both sources**.
- Namespaces not matching any criteria are **ignored** (safe by default).
