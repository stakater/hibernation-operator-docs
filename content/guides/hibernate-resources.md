# Hibernate a Tenant

This guide describes how to hibernate every namespace that belongs to a [Multi-Tenant Operator](https://docs.stakater.com/mto/) tenant by using a `ClusterResourceSupervisor`.

!!! note
    The `spec.hibernation` field was removed from the `Tenant` resource in MTO 1.7. Tenant hibernation is configured with a `ClusterResourceSupervisor` that selects the tenant's namespaces by label.

## Prerequisites

* The Hibernation Operator is installed in the cluster. Refer to [Installation](../getting-started/installation/overview.md).
* Permission to create `ClusterResourceSupervisor` resources, which are cluster scoped.

## Hibernate all namespaces of a tenant

MTO applies the label `stakater.com/tenant: <tenant-name>` to every namespace it manages for a tenant. A `ClusterResourceSupervisor` can select those namespaces with a single label selector:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: sigma-hibernation
spec:
  schedule:
    sleepSchedule: "0 20 * * 1-5"   # 20:00 UTC, Monday to Friday
    wakeSchedule: "0 8 * * 1-5"     # 08:00 UTC, Monday to Friday
  namespaces:
    labelSelector:
      matchLabels:
        stakater.com/tenant: sigma
```

At 20:00 UTC on each weekday, every `Deployment` and `StatefulSet` in the namespaces of the `sigma` tenant is scaled to zero, and its replica count is recorded in the resource status. At 08:00 UTC, each workload is restored to its recorded count. Namespaces created for the tenant later are included at the next scheduled reconcile.

## Keep a namespace of the tenant running

To exclude one of the tenant's namespaces, annotate it rather than changing the selector:

```sh
kubectl annotate namespace sigma-prod hibernation.stakater.com/exclude=true
```

The exclusion applies to every supervisor, whether it was created for a tenant or not. Excluded namespaces are listed in `status.ignoreNamespaces`.

## Tenants deployed with ArgoCD

If the tenant's workloads are deployed by ArgoCD, ArgoCD synchronises them back to their declared replica count after they are hibernated. To prevent this, name the tenant's AppProjects under `spec.argocd`:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: sigma-hibernation
spec:
  schedule:
    sleepSchedule: "0 20 * * 1-5"
    wakeSchedule: "0 8 * * 1-5"
  namespaces:
    labelSelector:
      matchLabels:
        stakater.com/tenant: sigma
  argocd:
    namespace: openshift-gitops
    appProjects:
      - sigma
```

The operator applies a deny sync window to the `sigma` AppProject for the duration of the sleep period. `spec.argocd` does not select namespaces; the namespaces are still selected by `spec.namespaces`. Refer to [ArgoCD Integration](../integrations/argocd.md) before enabling it.

## Verify the status

While the tenant is hibernated, the status of the resource lists each workload and the replica count it will be restored to:

```yaml
status:
  currentStatus: sleeping
  nextReconcileTime: "2026-06-11T08:00:00Z"
  watchedNamespaces:
    - sigma-dev
    - sigma-staging
  sleepingNamespaces:
    - Namespace: sigma-dev
      sleepingApplications:
        - kind: Deployment
          name: web
          replicas: 3
    - Namespace: sigma-staging
      sleepingApplications:
        - kind: StatefulSet
          name: database
          replicas: 1
```

## Related guides

* [Hibernate Workloads Across Multiple Namespaces](create-cluster-resource-supervisor.md)
* [Hibernate Workloads in a Single Namespace](create-resource-supervisor.md)
