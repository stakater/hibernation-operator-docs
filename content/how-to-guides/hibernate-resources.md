# Hibernate a Tenant

Implementing hibernation for tenants' namespaces efficiently manages cluster resources by temporarily reducing workload activities during off-peak hours. This guide demonstrates how to configure hibernation schedules for tenant namespaces, leveraging Tenant and ResourceSupervisor for precise control.

## Configuring Hibernation for Tenant Namespaces

You can manage workloads in your cluster with MTO by implementing a hibernation schedule for your tenants.
Hibernation downsizes the running `Deployments` and `StatefulSets` in a tenant’s namespace according to a defined cron schedule. You can set a hibernation schedule for your tenants by adding the ‘spec.hibernation’ field to the tenant's respective Custom Resource.

```yaml
hibernation:
  sleepSchedule: 23 * * * *
  wakeSchedule: 26 * * * *
```

`spec.hibernation.sleepSchedule` accepts a cron expression indicating the time to put the workloads in your tenant’s namespaces to sleep.

`spec.hibernation.wakeSchedule` accepts a cron expression indicating the time to wake the workloads in your tenant’s namespaces up.

!!! note
    Both sleep and wake schedules must be specified for your tenant's hibernation schedule to be valid.

Additionally, adding the `hibernation.stakater.com/exclude: 'true'` annotation to a namespace excludes it from hibernating.

!!! note
    This is only true for hibernation applied via the Tenant Custom Resource, and does not apply for hibernation done by manually creating a ClusterResourceSupervisor (details about that below).

## Cluster & Namespace Resource Supervisor

Adding a Hibernation Schedule to a Tenant creates an accompanying ClusterResourceSupervisor Custom Resource.

When the sleep timer is activated, the Resource Supervisor puts your applications to sleep and store their previous state. When the wake timer is activated, it uses the stored state to bring them back to running state.

Enabling ArgoCD support for Tenants will also hibernate applications in the tenants' `appProjects`.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: sigma-tenant
spec:
  argocd:
    appProjects:
      - sigma-tenant
    namespace: openshift-gitops
  schedule:
    sleepSchedule: 42 * * * *
    wakeSchedule: 45 * * * *

  namespaces:
    labelSelector:
      matchLabels:
        stakater.com/current-tenant: sigma
      matchExpressions: {}
    names:
    - tenant-ns1
    - tenant-ns2
```

> Currently, Hibernation is available only for `StatefulSets` and `Deployments`.

### Manual creation of ResourceSupervisor

Hibernation can also be applied by creating a ResourceSupervisor resource manually.
The ResourceSupervisor definition will contain the hibernation cron schedule, the names of the namespaces to be hibernated, and the names of the ArgoCD AppProjects whose ArgoCD Applications have to be hibernated (as per the given schedule).

This method can be used to hibernate:

- Some specific namespaces and AppProjects in a tenant
- A set of namespaces and AppProjects belonging to different tenants
- Namespaces and AppProjects belonging to a tenant that the cluster admin is not a member of
- Non-tenant namespaces and ArgoCD AppProjects

As an example, the following ClusterResourceSupervisor could be created manually, to apply hibernation explicitly to the 'ns1' and 'ns2' namespaces, and to the 'sample-app-project' AppProject.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: hibernator
spec:
  argocd:
    appProjects:
      - sample-app-project
    namespace: openshift-gitops
  schedule:
    sleepSchedule: 42 * * * *
    wakeSchedule: 45 * * * *

  namespaces:
    labelSelector:
      matchLabels: {}
      matchExpressions: {}
    names:
    - ns1
    - ns2
```

## Freeing up unused resources with hibernation

## Hibernation States

The ClusterResourceSupervisor will look like this at 'sleeping' time (as per the schedule):

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: example
spec:
  argocd:
    appProjects: []
    namespace: ''
  schedule:
    sleepSchedule: 0 20 * * 1-5
    wakeSchedule: 0 8 * * 1-5
  namespaces:
    labelSelector:
      matchLabels: {}
      matchExpressions: {}
    names:
      - build
      - stage
      - dev
status:
  currentStatus: sleeping
  nextReconcileTime: '2024-06-11T08:00:00Z'
  sleepingNamespaces:
  - Namespace: build
    sleepingApplications:
    - kind: Deployment
      name: Example
      replicas: 3
  - Namespace: stage
    sleepingApplications:
    - kind: Deployment
      name: Example
      replicas: 3
```

## Hibernating namespaces and/or ArgoCD Applications with Cluster ResourceSupervisor

Bill, the cluster administrator, wants to hibernate a collection of namespaces and AppProjects belonging to multiple different tenants. He can do so by creating a ResourceSupervisor manually, specifying the hibernation schedule in its spec, the namespaces and ArgoCD Applications that need to be hibernated as per the mentioned schedule.
Bill can also use the same method to hibernate some namespaces and ArgoCD Applications that do not belong to any tenant on his cluster.

The example given below will hibernate the ArgoCD Applications in the 'test-app-project' AppProject; and it will also hibernate the 'ns2' and 'ns4' namespaces.

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: test-cluster-resource-supervisor
spec:
  argocd:
    appProjects:
      - test-app-project
    namespace: argocd-ns
  schedule:
    sleepSchedule: 0 20 * * 1-5
    wakeSchedule: 0 8 * * 1-5
  namespaces:
    labelSelector:
      matchLabels: {}
      matchExpressions: {}
    names:
      - ns2
      - ns4
status:
  currentStatus: sleeping
  nextReconcileTime: '2022-10-13T08:00:00Z'
  sleepingNamespaces:
  - Namespace: build
    sleepingApplications:
    - kind: Deployment
      name: test-deployment
      replicas: 3
    - kind: Deployment
      name: test-deployment
      replicas: 3
```
