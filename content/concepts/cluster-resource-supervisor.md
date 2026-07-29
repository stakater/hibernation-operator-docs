# ClusterResourceSupervisor

The `ClusterResourceSupervisor` is a **cluster-scoped** custom resource that enables **centralized hibernation management** across multiple namespaces. It is designed for **platform administrators** who need to enforce cost-saving policies at scale—whether by targeting explicit namespaces, using dynamic label selectors, or integrating with **ArgoCD AppProjects**.

> ✅ **Scope**: Cluster-wide
> ✅ **Permissions**: Requires cluster-admin or equivalent
> ✅ **Use Case**: Platform-level hibernation for dev/test environments, GitOps portfolios, or tenant groups

---

## Supported Targeting Methods

You can define **one or more** of the following targeting strategies. The operator applies hibernation to the **union** of all matched namespaces.

### 1. **Explicit Namespace List**

List namespaces by name:

```yaml
spec:
  namespaces:
    names:
      - team-a-dev
      - team-b-staging
      - demo-env
```

### 2. **Label Selector (Dynamic)**

Use Kubernetes-standard label selectors to match namespaces dynamically:

```yaml
spec:
  namespaces:
    labelSelector:
      matchLabels:
        env: dev
        team: frontend
      matchExpressions:
        - key: "cost-center"
          operator: In
          values: ["cc-100", "cc-200"]
```

🔍 **Note**:

> - `matchLabels` and `matchExpressions` are **AND** together.
> - An empty `labelSelector: {}` matches **all namespaces**.
> - A missing/`null` `labelSelector` matches **none**.

### 3. **ArgoCD AppProject Integration** *(Optional)*

Target all namespaces associated with one or more ArgoCD `AppProject`s:

```yaml
spec:
  argocd:
    namespace: argocd               # ← Namespace where ArgoCD is installed
    appProjects:                    # ← List of AppProject names
      - frontend-team
      - data-platform
```

> 🔄 The operator reads `AppProject.spec.destinations` to discover target namespaces.
> ✅ **No manual namespace listing needed**—ideal for GitOps environments.
> ⚠️ **Prerequisite**: ArgoCD integration must be [enabled during installation](../getting-started/installation/kubernetes.md#optional-enable-argocd-integration).

---

## Hibernation Scheduling

Define when workloads should sleep and wake using standard **cron expressions** (UTC timezone).

### Full Cycle: Sleep + Wake

```yaml
spec:
  schedule:
    sleepSchedule: "0 18 * * 1-5"   # Weekdays at 6 PM UTC
    wakeSchedule: "0 8 * * 1-5"     # Weekdays at 8 AM UTC
```

### Permanent Sleep (Manual Wake)

Omit `wakeSchedule` to keep workloads asleep until the CR is updated or deleted:

```yaml
spec:
  schedule:
    sleepSchedule: "0 0 1 * *"      # Sleep on the 1st of every month
    # wakeSchedule: omitted → stay asleep
```

---

## Ignored Namespaces

Even if a namespace matches your targeting rules, it will be **excluded** if it has:

- The annotation:

  ```yaml
  hibernation.stakater.com/exclude: "true"
  ```

> 🔒 This allows teams to opt out of platform-wide hibernation policies.

---

## Status Tracking

The operator populates rich status fields for observability and debugging:

```yaml
status:
  currentStatus: sleeping                 # "running", "sleeping", or "error"
  nextReconcileTime: "2025-02-01T08:00:00Z"
  watchedNamespaces:                      # Namespaces being managed
    - team-a-dev
    - frontend-staging
  ignoreNamespaces:                       # Excluded (e.g., via annotation)
    - kube-system
  sleepingNamespaces:                     # Detailed state of scaled-down apps
    - Namespace: team-a-dev
      status: sleeping
      sleepingApplications:
        - name: web-api
          kind: Deployment
          replicas: 3                    # ← Original replica count (restored on wake)
          status: sleeping
```

Use `kubectl describe` or `kubectl get -o yaml` to inspect:

```sh
kubectl get clusterresourcesupervisor my-policy -o jsonpath='{.status}'
```

---

## Example: Platform-Wide Dev Hibernation

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
    sleepSchedule: "0 18 * * 1-5"
    wakeSchedule: "0 8 * * 1-5"
```

> 🌐 Applies to **all namespaces** labeled `env=dev`, now and in the future.

---

## Example: ArgoCD + Label Selector Combo

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: hybrid-hibernation
spec:
  argocd:
    namespace: argocd
    appProjects:
      - mobile-apps
  namespaces:
    names:
      - legacy-staging
    labelSelector:
      matchLabels:
        temporary: "true"
  schedule:
    sleepSchedule: "0 20 * * *"
    wakeSchedule: "0 8 * * *"
```

🔄 Hibernates:

> - All namespaces in the `mobile-apps` AppProject
> - Plus `legacy-staging`
> - Plus any namespace with label `temporary=true`

---

## Key Notes

- ❌ **Not namespace-scoped**: Cannot be created inside a namespace.
- ✅ **Safe by default**: Only modifies `Deployments` and `StatefulSets`.
- ✅ **Stateful restoration**: Replica counts are stored in `status.sleepingNamespaces`.
- ✅ **Coexists with `ResourceSupervisor`**: If a namespace has a local `ResourceSupervisor`, the operator **skips it** (to avoid conflicts)—unless your operator logic is designed otherwise. *(Confirm behavior in your implementation.)*

---

## When to Use `ClusterResourceSupervisor`

| Scenario | Recommended |
|--------|-------------|
| Hibernating 50+ dev namespaces | ✅ |
| GitOps with ArgoCD AppProjects | ✅ |
| Dynamic namespace selection via labels | ✅ |
| Team self-service in one namespace | ❌ → Use `ResourceSupervisor` |

---

## API Reference

**Group/Version:** `hibernation.stakater.com/v1beta1` · **Kind:** `ClusterResourceSupervisor` · **Scope:** Cluster

### Spec

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schedule` | `object` (Hibernation) | Yes | Hibernation schedule applied to all matched namespaces. |
| `schedule.sleepSchedule` | `string` | No | Standard 5-field Unix cron expression (UTC) for scaling workloads to zero. |
| `schedule.wakeSchedule` | `string` | No | Standard 5-field Unix cron expression (UTC) for restoring workloads. If omitted, workloads stay asleep. |
| `namespaces` | `object` | No | Targets namespaces by name and/or label selector. |
| `namespaces.names` | `[]string` | No | Explicit list of namespace names to manage. |
| `namespaces.labelSelector` | `object` (`metav1.LabelSelector`) | No | Standard Kubernetes label selector (`matchLabels` / `matchExpressions`). Empty `{}` matches all namespaces; absent matches none. |
| `argocd` | `object` | No | Target namespaces via ArgoCD AppProjects. Requires ArgoCD integration enabled at install time. |
| `argocd.appProjects` | `[]string` | Yes (within `argocd`) | ArgoCD AppProject names whose destination namespaces follow the schedule. |
| `argocd.namespace` | `string` | Yes (within `argocd`) | Namespace where the ArgoCD AppProjects reside. |

### Status

| Field | Type | Description |
|-------|------|-------------|
| `currentStatus` | `string` (`sleeping`, `running`, `error`) | Overall state of managed workloads. |
| `nextReconcileTime` | `string` (RFC 3339 timestamp) | Next scheduled sleep/wake reconciliation. |
| `watchedNamespaces` | `[]string` | Namespaces currently managed by this resource. |
| `ignoreNamespaces` | `[]string` | Namespaces excluded from management (e.g. via the exclude annotation). |
| `sleepingNamespaces` | `[]object` (SleepingNamespace) | Per-namespace record of scaled-down workloads, used for accurate restoration. |
| `sleepingNamespaces[].Namespace` | `string` | The namespace containing the sleeping applications. |
| `sleepingNamespaces[].status` | `string`  (`sleeping`, `running`, `error`)  | Per-namespace error/state indicator. |
| `sleepingNamespaces[].sleepingApplications` | `[]object` (SleepingApplication) | Workloads scaled down in the namespace. |
| `sleepingNamespaces[].sleepingApplications[].name` | `string` | Name of the sleeping application. |
| `sleepingNamespaces[].sleepingApplications[].kind` | `string` | Workload kind. |
| `sleepingNamespaces[].sleepingApplications[].replicas` | `int32` | Original replica count, preserved for restoration on wake. |
