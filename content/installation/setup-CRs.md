# Setup ResourceSupervisor & ClusterResourceSupervisor

## Creating a `ResourceSupervisor`

This guide explains how to create a **namespace-scoped** `ResourceSupervisor` to hibernate workloads in a single namespace.

### Prerequisites

- The **Hibernation Operator** must be [installed](../installation/kubernetes.md) in your cluster.
- You have **edit** (or equivalent) permissions in the target namespace.

### Step 1: Create a `ResourceSupervisor` YAML

Create a file named `resource-supervisor.yaml`:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: my-namespace-hibernation
  namespace: my-app-staging  # ← Must match the namespace you want to hibernate
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # Sleep every day at 8 PM UTC
    wakeSchedule: "0 8 * * *"    # Wake up every day at 8 AM UTC
```

> 💡 **Cron Format**: Uses standard Unix cron syntax (`minute hour day month weekday`).  
> Timezone is **UTC**. Use tools like [crontab.guru](https://crontab.guru) to validate schedules.

### Step 2: Apply the Resource

```sh
kubectl apply -f resource-supervisor.yaml
```

### Step 3: Verify Status

Check the status to confirm the hibernation state:

```sh
kubectl get resourcesupervisor my-namespace-hibernation -n my-app-staging -o yaml
```

Look for:

- `status.currentStatus`: `running` or `sleeping`
- `status.nextReconcileTime`: Next scheduled action (ISO 8601)

### Notes

- Only `Deployments` and `StatefulSets` in the **same namespace** are affected.
- The operator **scales them to 0 replicas** during sleep and **restores original replica counts** on wake.
- If `wakeSchedule` is omitted, workloads stay asleep until manually deleted or updated.

---

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
