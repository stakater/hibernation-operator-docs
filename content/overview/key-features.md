# Key Features

The operator is driven by two custom resources that share the same schedule format and the same status format. They differ only in who creates them and in how many namespaces they govern.

## Sleep and wake on a cron schedule

Most non-production environments are idle at night and at weekends, and hibernation removes those hours from the cost of running them. The `spec.schedule` field accepts a `sleepSchedule` and a `wakeSchedule`, each expressed as a standard five-field cron expression and evaluated in UTC. Both fields are optional, and the combination that is set determines the behaviour:

| `sleepSchedule` | `wakeSchedule` | Behaviour |
|---|---|---|
| Set | Set | Workloads sleep at each sleep time and wake at each wake time |
| Set | Empty | Workloads sleep at the next sleep time and remain asleep |
| Empty | Empty | Workloads sleep immediately and remain asleep until the schedule is changed or the resource is deleted |
| Empty | Set | Rejected by the admission webhook, as there is no sleep period to apply |

The admission webhook also rejects cron expressions that cannot be parsed, and sleep and wake schedules that are identical.

For further details, refer to [ResourceSupervisor](../concepts/resource-supervisor.md) and [ClusterResourceSupervisor](../concepts/cluster-resource-supervisor.md).

## Restore every workload to its previous replica count

A hibernated environment must come back exactly as it was. Before any workload is scaled down, the operator records its replica count in `status.sleepingNamespaces` on the resource responsible for hibernating it. Workloads are restored exclusively from this record, which ensures that:

* Each workload is restored to its own replica count rather than to a default value.
* Workloads that the operator did not hibernate are never modified when it restores others.
* Workloads already scaled to zero are skipped, so that their recorded count is never overwritten with zero.
* The record is stored in the Kubernetes API and therefore persists across operator restarts.

When a supervisor is deleted, its workloads are restored before the deletion completes. If a restoration fails, the resource is retained until it succeeds.

Hibernation applies to `Deployments` and `StatefulSets` only. `PersistentVolumeClaims`, and the storage costs associated with them, are not affected.

## Hibernate one namespace or many

Hibernation can be decided by the team that owns a namespace or by the platform team that runs the cluster, and each has its own resource:

| | `ResourceSupervisor` | `ClusterResourceSupervisor` |
|---|---|---|
| Scope | Namespace | Cluster |
| Governs | The namespace in which it is created | Every namespace it selects |
| Created by | Any user permitted to create it in that namespace | A cluster administrator |
| Namespace targeting | Not applicable | Namespace names and label selectors |
| ArgoCD sync windows | Not supported | Supported |

The admission webhook prevents two `ClusterResourceSupervisor` resources from selecting the same namespace. It does not, however, prevent a `ClusterResourceSupervisor` and a `ResourceSupervisor` from governing the same namespace, so such overlaps must be avoided by convention.

## Select namespaces by name or label

A platform team decides which environments are hibernated by selecting their namespaces. A `ClusterResourceSupervisor` combines two lists defined under `spec.namespaces`:

* `names`: namespaces specified explicitly by name.
* `labelSelector`: a standard Kubernetes label selector using `matchLabels` and `matchExpressions`. Namespaces created later with matching labels are included at the next scheduled sleep or wake.

A label selector that defines neither `matchLabels` nor `matchExpressions` selects **no** namespaces, rather than all of them.

When a namespace is removed from the scope of a supervisor, any workloads the supervisor had hibernated in that namespace are restored.

For further details, refer to [Hibernate Workloads Across Multiple Namespaces](../guides/create-cluster-resource-supervisor.md).

## Exclude namespaces from hibernation

Some namespaces must never be hibernated, whatever schedule selects them. A namespace carrying the following annotation is never hibernated by a supervisor of either kind:

```yaml
metadata:
  annotations:
    hibernation.stakater.com/exclude: "true"
```

An exclusion takes precedence over every schedule. Any workloads that a supervisor had already hibernated in the namespace are restored at its next scheduled sleep or wake. The namespace in which the operator runs is always excluded. Excluded namespaces are listed in `status.ignoreNamespaces`, which explains why a schedule may have had no effect.

## Keep ArgoCD from undoing hibernation

ArgoCD treats a workload scaled to zero as drift from the desired state in Git and synchronises it back to its declared replica count. When `spec.argocd` is set on a `ClusterResourceSupervisor`, the operator applies a `deny` sync window to each specified AppProject. The window opens at the sleep time and remains in effect until the wake time. Manual synchronisation remains permitted.

The `spec.argocd` field does not determine which namespaces are hibernated; these are always selected through `spec.namespaces`. Because the sync window is derived from `sleepSchedule`, no window is applied when that field is empty.

!!! warning
    The operator replaces the existing `spec.syncWindows` of each AppProject with its own window, and removes all sync windows when the supervisor is deleted. It should therefore not be used with AppProjects that define sync windows of their own.

!!! warning
    The operator's ClusterRole does not include permissions on `appprojects` in the `argoproj.io` API group. Grant `get` and `patch` on this resource to the operator's service account before using `spec.argocd`.

For further details, refer to [ArgoCD Integration](../integrations/argocd.md).

## Track hibernation state in the resource status

The status of each resource shows what is currently hibernated and when that will change. Both resources report the same fields:

| Field | Description |
|---|---|
| `currentStatus` | The current state: `sleeping`, `running` or `error` |
| `nextReconcileTime` | The time at which the next sleep or wake is due |
| `watchedNamespaces` | The namespaces the supervisor is permitted to hibernate |
| `ignoreNamespaces` | The namespaces that were selected but cannot be hibernated, because they are excluded or no longer exist |
| `sleepingNamespaces` | Each hibernated workload, with its kind, its recorded replica count and, after a failed wake, an `errorMessage` |
| `conditions` | A `Ready` condition stating whether the last sleep or wake succeeded and, if not, the reason |

If a sleep or wake fails, `currentStatus` is set to `error`, the `Ready` condition records the reason, such as `SleepFailed`, `WakeFailed` or `InvalidSleepSchedule`, and the operation is retried. The same failures are also reported as Kubernetes events on the resource. For further details, refer to [Troubleshooting](../troubleshooting.md).

## Next steps

* [Use Cases](use-cases.md): the scenarios the operator is designed for
* [Architecture](../concepts/architecture.md): the components and how they behave
* [Quick Start](../getting-started/quick-start.md): hibernating a first namespace
