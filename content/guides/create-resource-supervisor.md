# Hibernate Workloads in a Single Namespace

A `ResourceSupervisor` scales the `Deployments` and `StatefulSets` in its own namespace down to zero on a schedule, and back to their original replica counts when the wake schedule fires. It is namespace-scoped, so a team that owns a namespace can set this up without cluster-wide permissions.

To cover several namespaces at once, use a [ClusterResourceSupervisor](create-cluster-resource-supervisor.md) instead.

## Prerequisites

- The Hibernation Operator is [installed](../getting-started/installation/kubernetes.md) in the cluster.
- You have edit permissions in the namespace you want to hibernate.

## Step 1: Define the ResourceSupervisor

Create `resource-supervisor.yaml`. The `metadata.namespace` is the namespace that will be hibernated, since the resource only acts on its own namespace.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: my-namespace-hibernation
  namespace: my-app-staging
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # Sleep daily at 20:00 UTC
    wakeSchedule: "0 8 * * *"     # Wake daily at 08:00 UTC
```

!!! note
    Both schedules use standard five-field Unix cron syntax (`minute hour day month weekday`) and are evaluated in UTC, not the cluster's local timezone. [crontab.guru](https://crontab.guru) is useful for checking an expression before applying it.

Omitting `wakeSchedule` puts the workloads to sleep and leaves them there until you edit or delete the resource. That is a deliberate option for environments you want parked indefinitely, but it is easy to do by accident.

## Step 2: Apply it

```sh
kubectl apply -f resource-supervisor.yaml
```

## Step 3: Check the status

```sh
kubectl get resourcesupervisor my-namespace-hibernation -n my-app-staging -o yaml
```

Two fields tell you where things stand:

- `status.currentStatus` is `running`, `sleeping`, or `error`.
- `status.nextReconcileTime` is the next time the operator will sleep or wake the workloads.

## What gets hibernated

Only `Deployments` and `StatefulSets` in the same namespace are touched. Other workload types, and resources in any other namespace, are left alone. Workloads already sitting at zero replicas are skipped rather than recorded as sleeping.

Before scaling a workload down, the operator writes its replica count to the annotation `hibernation.stakater.com/original-replicas` on the workload itself. Waking reads the count back from there, so you can always see what a sleeping workload will return to:

```sh
kubectl get deploy -n my-app-staging -o custom-columns=\
NAME:.metadata.name,REPLICAS:.spec.replicas,ORIGINAL:'.metadata.annotations.hibernation\.stakater\.com/original-replicas'
```

Deleting the `ResourceSupervisor` wakes its workloads first. A finalizer holds the resource until the restore completes, so removing the supervisor is a safe way to cancel hibernation rather than a way to strand workloads at zero.

A namespace annotated with `hibernation.stakater.com/exclude: "true"` is skipped, even if a `ResourceSupervisor` exists in it.

## Related guides

- [Hibernate Workloads Across Multiple Namespaces](create-cluster-resource-supervisor.md)
- [Hibernate a Tenant](hibernate-resources.md)
