# ClusterResourceSupervisor

A `ClusterResourceSupervisor` applies one hibernation schedule to a group of namespaces. It is cluster-scoped and intended for platform teams parking development, test, or preview environments at scale. For a single namespace managed by the team that owns it, use a [ResourceSupervisor](resource-supervisor.md).

As with the namespace-scoped resource, only `Deployments` and `StatefulSets` are affected.

For the complete field listing, see the [API Reference](../reference/api.md).

## Selecting namespaces

Namespaces come from `spec.namespaces`, by explicit name, by label selector, or both. The results are combined, so a namespace selected either way is hibernated.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: dev-environments-hibernation
spec:
  namespaces:
    names:
      - legacy-staging
    labelSelector:
      matchLabels:
        env: dev
  schedule:
    sleepSchedule: "0 18 * * 1-5"
    wakeSchedule: "0 8 * * 1-5"
```

`labelSelector` is a standard Kubernetes label selector, so `matchLabels` and `matchExpressions` both work and are combined with AND. It is re-evaluated on each reconcile, which is what lets namespaces created later be picked up without editing the resource.

!!! warning
    An empty `labelSelector: {}` matches no namespaces, not all of them, and a `ClusterResourceSupervisor` with no `spec.namespaces` at all hibernates nothing. To cover a broad set, apply a label the target namespaces share and select on that.

## Scheduling

`spec.schedule` behaves as it does for `ResourceSupervisor`: two five-field cron expressions evaluated in UTC. Setting both cycles the workloads, and omitting `wakeSchedule` sleeps them at the next sleep time and leaves them asleep.

```yaml
spec:
  schedule:
    sleepSchedule: "0 22 * * *"
    wakeSchedule: "0 6 * * *"
```

## Exclusions

Two filters run after selection, and a namespace caught by either is reported in `status.ignoreNamespaces` rather than hibernated:

- The namespace is annotated `hibernation.stakater.com/exclude: "true"`.
- The namespace is the one the operator itself runs in.

The annotation is how an individual team opts out of a platform-wide policy without needing the selector changed.

## Overlap between supervisors

The admission webhook rejects a `ClusterResourceSupervisor` that claims a namespace already claimed by another one, and also rejects duplicate names within a single resource. Two cluster-scoped supervisors therefore cannot fight over the same namespace.

!!! warning
    That check does not extend to `ResourceSupervisor`. A namespace can hold its own `ResourceSupervisor` while also being selected here, and both controllers will then act on the same workloads on their own schedules. Exclude the namespace with the annotation if it should manage itself.

## ArgoCD sync windows

Workloads managed by ArgoCD present a problem: scaling them to zero is drift from Git, and ArgoCD will scale them back up. The `spec.argocd` field addresses this by writing a `deny` sync window onto the AppProjects you name, covering the sleep schedule and its computed duration.

```yaml
spec:
  namespaces:
    labelSelector:
      matchLabels:
        team: frontend
  argocd:
    namespace: argocd
    appProjects:
      - frontend-team
  schedule:
    sleepSchedule: "0 22 * * *"
    wakeSchedule: "0 6 * * *"
```

!!! warning
    `spec.argocd` does not select namespaces. It suppresses syncing for the AppProjects listed, nothing more, and the namespaces to hibernate still come entirely from `spec.namespaces`. A resource with `argocd` but no `namespaces` writes sync windows and hibernates nothing.

Two further constraints:

- The operator replaces `spec.syncWindows` on each named AppProject rather than appending, so any sync windows maintained there by other means are overwritten.
- The ArgoCD `AppProject` CRD must be present. If it is absent the operator logs that and continues hibernating without touching ArgoCD.

## Status

| Field | Shows |
| --- | --- |
| `currentStatus` | `running`, `sleeping`, or `error` |
| `watchedNamespaces` | Namespaces currently being managed |
| `ignoreNamespaces` | Selected namespaces filtered out by the exclusions above |
| `sleepingNamespaces` | Per-namespace detail of scaled-down workloads and their original replica counts |
| `nextReconcileTime` | Next scheduled sleep or wake |

```sh
kubectl get clusterresourcesupervisor dev-environments-hibernation -o jsonpath='{.status}'
```

When a namespace you expected is missing from `watchedNamespaces`, `ignoreNamespaces` is the first place to look.

## Choosing between the two

Use a `ResourceSupervisor` when a namespace should decide its own hibernation and the team owning it holds only namespace-level permissions. Use a `ClusterResourceSupervisor` when a platform team sets the policy across many namespaces, when the set is defined by labels and changes over time, or when ArgoCD sync windows need suppressing during sleep.

## Related guides

- [Hibernate Workloads Across Multiple Namespaces](../guides/create-cluster-resource-supervisor.md)
- [Hibernate a Tenant](../guides/hibernate-resources.md)
