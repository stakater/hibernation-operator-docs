# Use Cases

The Hibernation Operator is intended for clusters that run environments continuously, even though those environments are only used for part of the day. The scenarios below describe the situations for which it was designed.

## Development and test platforms

**Scenario.** A platform team operates a shared cluster for many development teams. Each team maintains `dev`, `test` and `staging` namespaces, all of which continue to run overnight and at weekends while no one is using them.

**Approach.** A single `ClusterResourceSupervisor` selects every non-production namespace by label and hibernates those namespaces outside working hours:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: non-production-office-hours
spec:
  schedule:
    sleepSchedule: "0 19 * * 1-5"
    wakeSchedule: "0 7 * * 1-5"
  namespaces:
    labelSelector:
      matchExpressions:
        - key: env
          operator: In
          values: [dev, test, staging]
```

Newly created namespaces with a matching label are included without any change to the supervisor. A namespace that must remain available, for example during a demonstration or a load test, is annotated with `hibernation.stakater.com/exclude: "true"` rather than being removed from the selector.

## Team self-service

**Scenario.** A product team owns the `payments-staging` namespace and is best placed to determine its own working hours. The team should be able to reduce costs without raising a request with the platform team or holding cluster-level permissions.

**Approach.** The team creates a `ResourceSupervisor` in its own namespace. Because the resource can only affect the namespace in which it is created, permission to create it can safely be granted to namespace owners, and each team can define its own schedule. For further details, refer to [Hibernate Workloads in a Single Namespace](../guides/create-resource-supervisor.md).

## GitOps-managed environments

**Scenario.** The environments are deployed by ArgoCD. A workload scaled to zero is treated as drift from Git, so ArgoCD synchronises it back to its declared replica count and hibernation does not take effect.

**Approach.** A `ClusterResourceSupervisor` that specifies the relevant AppProjects under `spec.argocd` applies a deny sync window to them for the duration of the sleep period, so that ArgoCD does not restore the hibernated workloads before the wake time. The namespaces to be hibernated are still selected under `spec.namespaces`. For further details, refer to [ArgoCD Integration](../integrations/argocd.md).

## Holiday and shutdown periods

**Scenario.** During an organisation-wide closure, non-critical environments should remain unavailable for the entire period and be restored only when a deliberate decision is made to do so.

**Approach.** A `ClusterResourceSupervisor` with an empty schedule hibernates the selected namespaces immediately and keeps them hibernated. Deleting the supervisor at the end of the period restores every workload to its recorded replica count.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: year-end-shutdown
spec:
  schedule: {}
  namespaces:
    labelSelector:
      matchLabels:
        criticality: low
```

## Demonstration and training environments

**Scenario.** Environments used for customer demonstrations or training are only required during business hours, and many copies of them may exist at any time.

**Approach.** A `ResourceSupervisor` is included in the environment template, so every copy is provisioned with a hibernation schedule already in place and no manual intervention is required.

## Multi-tenant clusters

**Scenario.** A cluster hosts many tenants, and the non-production environments of lower-tier tenants are not required to run overnight.

**Approach.** A `ClusterResourceSupervisor` selects namespaces by a tenant or tier label, such as `tier: basic`, while namespaces belonging to other tenants continue to run. When [Multi-Tenant Operator](https://docs.stakater.com/mto/) is in use, every tenant namespace already carries the `stakater.com/tenant: <tenant-name>` label, so a single selector can cover an entire tenant.

## Limitations

The operator is not the appropriate tool in the following situations:

* **Ephemeral environments that should be removed.** Hibernation retains the namespace, its configuration and its volumes. Preview environments that are no longer needed should be deleted instead.
* **Storage costs.** Only `Deployments` and `StatefulSets` are scaled. `PersistentVolumeClaims` are not affected.
* **Other workload kinds.** `DaemonSets`, `Jobs`, `CronJobs` and custom workload resources are not hibernated.

## Next steps

* [Key Features](key-features.md): a description of each feature
* [Quick Start](../getting-started/quick-start.md): hibernating a first namespace
* [Hibernate Workloads Across Multiple Namespaces](../guides/create-cluster-resource-supervisor.md): the cluster-scoped resource in detail
