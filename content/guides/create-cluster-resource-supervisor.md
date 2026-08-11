# Hibernate Workloads Across Multiple Namespaces

A `ClusterResourceSupervisor` applies one hibernation schedule to a group of namespaces. It is cluster-scoped, so it suits a platform team parking development or test environments outside working hours, rather than a single team managing its own namespace.

For one namespace owned by the team that hibernates it, use a [ResourceSupervisor](create-resource-supervisor.md) instead.

## Prerequisites

- The Hibernation Operator is [installed](../getting-started/installation/kubernetes.md) in the cluster.
- You have cluster-admin permissions.

## Step 1: Choose how to select namespaces

Namespaces come from `spec.namespaces`, either listed by name, matched by label, or both. The two are combined, so a namespace selected either way is hibernated.

| Field | Selects |
| --- | --- |
| `spec.namespaces.names` | A fixed list of namespaces you name explicitly |
| `spec.namespaces.labelSelector` | Every namespace matching the labels, re-evaluated as namespaces come and go |

!!! warning
    A `ClusterResourceSupervisor` with no `spec.namespaces` hibernates nothing. This matters most when using the ArgoCD integration described below, which does not select namespaces on its own.

An empty `labelSelector: {}` also matches nothing. To select broadly, use a label every target namespace carries rather than an empty selector.

## Step 2: Define the ClusterResourceSupervisor

Selecting by label, so namespaces labelled `env=dev` are picked up automatically as they are created:

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
    sleepSchedule: "0 18 * * 1-5"   # Weekdays at 18:00 UTC
    wakeSchedule: "0 8 * * 1-5"     # Weekdays at 08:00 UTC
```

Selecting a fixed set by name:

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
    sleepSchedule: "0 0 * * 0"      # Sundays at midnight UTC
    wakeSchedule: "0 0 * * 1"       # Mondays at midnight UTC
```

Schedules use five-field Unix cron syntax in UTC. Omitting `wakeSchedule` leaves the workloads asleep until the resource is edited or deleted.

## Step 3: Apply it

```sh
kubectl apply -f cluster-resource-supervisor.yaml
```

## Step 4: Check the status

```sh
kubectl get clusterresourcesupervisor dev-environments-hibernation -o yaml
```

| Field | Shows |
| --- | --- |
| `status.currentStatus` | `running`, `sleeping`, or `error` |
| `status.watchedNamespaces` | Namespaces currently being managed |
| `status.ignoreNamespaces` | Selected namespaces that were filtered out |
| `status.sleepingNamespaces` | Per-namespace detail of the scaled-down workloads |
| `status.nextReconcileTime` | Next scheduled sleep or wake |

If a namespace you expected is missing from `watchedNamespaces`, check `ignoreNamespaces`. Namespaces annotated `hibernation.stakater.com/exclude: "true"`, and the operator's own namespace, are always filtered out.

## Keeping ArgoCD from waking workloads

When applications are managed by ArgoCD, scaling them to zero puts them out of sync with Git, and ArgoCD will scale them back up. The `spec.argocd` field addresses this by writing a `deny` sync window onto the named AppProjects, matching the sleep schedule and its computed duration.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: argocd-frontend-hibernation
spec:
  namespaces:
    labelSelector:
      matchLabels:
        team: frontend
  argocd:
    namespace: argocd          # Namespace where the AppProjects live
    appProjects:
      - frontend-team
      - mobile-apps
  schedule:
    sleepSchedule: "0 22 * * *"
    wakeSchedule: "0 6 * * *"
```

!!! warning
    `spec.argocd` does not select namespaces. It only suppresses ArgoCD syncing for the AppProjects you name, and the namespaces to hibernate still come entirely from `spec.namespaces`. The example above hibernates the `team=frontend` namespaces; without that `namespaces` block it would hibernate nothing while still writing sync windows.

Two further points before enabling this:

- The operator replaces `spec.syncWindows` on each named AppProject rather than appending to it. Any sync windows you maintain there by other means are overwritten.
- The integration requires the ArgoCD `AppProject` CRD to be present, and `argoCD.enabled=true` at [install time](../getting-started/installation/kubernetes.md#optional-enable-argocd-integration). If the CRD is absent the operator logs the fact and carries on hibernating.

## Related guides

- [Hibernate Workloads in a Single Namespace](create-resource-supervisor.md)
- [Hibernate a Tenant](hibernate-resources.md)
