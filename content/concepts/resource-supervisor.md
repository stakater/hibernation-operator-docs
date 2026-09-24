# ResourceSupervisor

A `ResourceSupervisor` hibernates the workloads in the namespace it lives in. It is namespace-scoped, so a team that owns a namespace can park its own workloads outside working hours without needing cluster-wide permissions. To cover a group of namespaces from one place, use a [ClusterResourceSupervisor](cluster-resource-supervisor.md).

Only `Deployments` and `StatefulSets` are affected. Everything else in the namespace is left running.

For the complete field listing, see the [API Reference](../reference/api.md).

## Scheduling modes

The `spec.schedule` block accepts two cron expressions, and which of them you set determines the behavior.

| `sleepSchedule` | `wakeSchedule` | Behavior |
| --- | --- | --- |
| Set | Set | Workloads cycle between the two times |
| Set | Omitted | Workloads sleep at the next sleep time and stay asleep |
| Omitted | Omitted | Workloads sleep immediately and stay asleep |

Both expressions use standard five-field Unix cron syntax (`minute hour day month weekday`) and are evaluated in UTC, not the cluster's local timezone.

### Recurring hibernation

Setting both schedules is the common case, parking an environment overnight and bringing it back in the morning.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: nightly-hibernation
  namespace: my-app-staging
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # Sleep daily at 20:00 UTC
    wakeSchedule: "0 8 * * *"     # Wake daily at 08:00 UTC
```

### Sleep with no scheduled wake

Omitting `wakeSchedule` sleeps the workloads at the next sleep time and leaves them there. The operator still reconciles at each subsequent sleep time, so workloads added to the namespace later are hibernated too rather than being left running.

```yaml
spec:
  schedule:
    sleepSchedule: "0 12 * * *"   # Sleep at the next 12:00 UTC, no wake
```

To bring the workloads back, add a `wakeSchedule` or delete the resource.

### Immediate sleep

An empty schedule sleeps the workloads as soon as the resource is created, and reconciles hourly to catch anything new. This suits ephemeral environments such as CI preview namespaces, where the namespace should be parked from the moment it exists.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: sleep-now
  namespace: ci-preview-pr123
spec:
  schedule: {}
```

!!! note
    Setting `wakeSchedule` without `sleepSchedule` is not supported, and the resource is rejected.

## Replica counts and restoration

Before scaling a workload down, the operator records its replica count in `status.sleepingNamespaces` on the `ResourceSupervisor`, and a wake restores exactly that count. A wake is restore only: a workload the operator never slept is left alone, and no count is ever guessed.

Workloads already at zero replicas are skipped, so they are not later "restored" to zero-with-a-record they never had.

!!! warning
    The status is the only record of the counts. If it is lost, the workloads stay at zero and the operator emits a `LedgerLost` Warning Event instead of inventing a count. Scale them back by hand.

Versions before v0.1.104 stored the count in the annotation `hibernation.stakater.com/original-replicas` on each workload, and woke anything without it to 1 replica. The annotation is still read on wake, so workloads slept by an older version come back at their recorded count, but nothing writes it any more.

Deleting the `ResourceSupervisor` wakes its workloads first. A finalizer holds the resource until the restore succeeds, and a failed wake keeps it in place and retries, which makes deletion the straightforward way to cancel hibernation.

## Exclusions

A namespace annotated `hibernation.stakater.com/exclude: "true"` is never hibernated, even if a `ResourceSupervisor` exists in it, and anything already asleep there is woken. The operator's own namespace is excluded the same way. This gives platform teams a way to protect a namespace regardless of what is created inside it.

## Status

`status.currentStatus` reports `running`, `sleeping`, or `error`, and `status.nextReconcileTime` gives the next time the operator will act. The `Ready` condition in `status.conditions` says whether the last sleep or wake succeeded, and if not why. See [Troubleshooting](../troubleshooting.md#finding-why-a-sleep-or-wake-failed) for its reasons.

```sh
kubectl get resourcesupervisor nightly-hibernation -n my-app-staging -o jsonpath='{.status}'
```

## Differences from ClusterResourceSupervisor

`ResourceSupervisor` has no `namespaces` field and cannot target anything beyond its own namespace, and no `argocd` field, so it cannot suppress ArgoCD syncing during sleep. If the namespace's workloads are managed by ArgoCD, ArgoCD will see the scaled-down state as drift and may restore it. Use a [ClusterResourceSupervisor](cluster-resource-supervisor.md) where that matters.

!!! warning
    Nothing prevents a `ClusterResourceSupervisor` from also selecting a namespace that already has a `ResourceSupervisor`. The admission webhook rejects overlap between two `ClusterResourceSupervisor` resources, but it does not check for this case, and the two controllers will then act on the same workloads independently. Avoid the overlap, or exclude the namespace with the annotation above.

## Related guides

- [Hibernate Workloads in a Single Namespace](../guides/create-resource-supervisor.md)
- [Hibernate a Tenant](../guides/hibernate-resources.md)
