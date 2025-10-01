# ResourceSupervisor

The `ResourceSupervisor` is a **namespace-scoped** custom resource that enables **self-service hibernation** of workloads within a **single namespace**. It allows application teams or namespace owners to define schedules for scaling down and restoring `Deployments` and `StatefulSets` without requiring cluster-wide permissions.

> ✅ **Only affects the namespace in which it is created.**

## Supported Workloads

- `Deployment`
- `StatefulSet`

These resources are scaled to **0 replicas** during sleep and restored to their **original replica count** during wake.

## Ignored Namespaces

While `ResourceSupervisor` is namespace-scoped and only acts within its own namespace, the Hibernation Operator still respects global exclusion rules. A namespace (including the one containing the `ResourceSupervisor`) will be **ignored** if it has:

- The annotation:
  
  ```yaml
  hibernation.stakater.com/exclude: "true"
  ```

> 🔒 The operator **never modifies** workloads in excluded namespaces—even if a `ResourceSupervisor` exists there.

## Supported Modes

### 1. Hibernation with Cron Schedule (Sleep + Wake)

Define both `sleepSchedule` and `wakeSchedule` to automatically cycle workloads on and off.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
meta
  name: nightly-hibernation
  namespace: my-app-staging  # ← Must match target namespace
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # Sleep daily at 8:00 PM UTC
    wakeSchedule: "0 8 * * *"     # Wake daily at 8:00 AM UTC
```

> 🕒 **Cron Format**: Standard Unix cron (`minute hour day month weekday`). Timezone is **UTC**.

---

### 2. Permanent Sleep (Manual Wake)

Omit `wakeSchedule` to keep workloads asleep indefinitely. Workloads will **only wake** when:

- The `ResourceSupervisor` is **deleted**, **or**
- A `wakeSchedule` is **added later**

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
meta
  name: pause-for-maintenance
  namespace: demo-env
spec:
  schedule:
    sleepSchedule: "0 12 * * *"   # Sleep today at noon UTC, never wake
    # wakeSchedule: omitted → stay asleep
```

> 💡 This is useful for temporary freezes (e.g., during security reviews or budget pauses).

---

### 3. Immediate Sleep (No Schedule)

To sleep **immediately**, create a `ResourceSupervisor` with an **empty `schedule`** block:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
meta
  name: sleep-now
  namespace: ci-preview-pr123
spec:
  schedule: {}
```

- Workloads are scaled to 0 **as soon as the CR is created**.
- They remain asleep until the CR is **deleted**.

> 🚀 Ideal for ephemeral environments (e.g., CI/CD preview namespaces).

---

## Status Tracking

The operator updates the CR’s `status` to reflect current state:

```yaml
status:
  currentStatus: sleeping      # or "running", "error"
  nextReconcileTime: "2025-02-01T08:00:00Z"
```

Use `kubectl describe` or `kubectl get -o yaml` to monitor:

```sh
kubectl get resourcesupervisor nightly-hibernation -n my-app-staging -o jsonpath='{.status}'
```

---

## Key Notes

- ❌ **No `argocd` field**: `ResourceSupervisor` **does not support ArgoCD integration**. Use `ClusterResourceSupervisor` for AppProject-based hibernation.
- ❌ **No `namespaces` field**: It **only affects its own namespace**—you cannot target other namespaces.
- ✅ **Safe by default**: Only `Deployments` and `StatefulSets` are scaled; all other resources are untouched.
- ✅ **Stateful restoration**: Original replica counts are stored in the CR’s status and restored accurately—even after operator restarts.

---

## Example: CI/CD Preview Namespace

A CI pipeline creates a namespace `pr-456` and immediately hibernates it to save costs:

```yaml
# pr-456-hibernation.yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
meta
  name: auto-sleep
  namespace: pr-456
spec:
  schedule: {}
```

Apply it:

```sh
kubectl apply -f pr-456-hibernation.yaml
```

Workloads sleep instantly. When the PR is merged, the pipeline deletes the namespace (or the CR), waking workloads briefly before cleanup.
