---
head:
  - - meta
    - name: keywords
      content: hibernation operator, kubernetes, cost optimization, sleep, wake, cron, argocd
---

# Hibernation Operator

**Reduce the cost of idle Kubernetes environments.**

Development, test, demonstration and preview environments are typically in use for only part of the working week, yet their workloads continue to run and to hold the capacity they request. The Hibernation Operator scales these workloads down to zero according to a schedule and restores them to their previous state when they are next required.

Platform teams can apply hibernation to many namespaces through a single resource, while application teams can manage hibernation for their own namespace without requiring cluster-level permissions.

## Capabilities

| Capability | Question it addresses | How the operator addresses it |
|---|---|---|
| **Scheduled sleep and wake** | How can idle hours, such as nights and weekends, be removed from the bill? | Scales `Deployments` and `StatefulSets` to zero on a sleep schedule and restores them on a wake schedule |
| **Exact restoration** | Will every workload return to its previous state? | Records each replica count in the resource status before scaling down, and restores from that record |
| **Two scopes** | Who decides what is hibernated? | `ClusterResourceSupervisor` for platform teams across namespaces, and `ResourceSupervisor` for application teams within a single namespace |
| **Namespace targeting** | Which environments should be hibernated? | Selects namespaces by name, by label selector, or by both |
| **Administrator exclusions** | Which namespaces must never be hibernated? | A single namespace annotation excludes a namespace regardless of any schedule |
| **ArgoCD awareness** | How is ArgoCD prevented from restoring hibernated workloads? | Applies a deny sync window to the specified AppProjects for the duration of the sleep period |

## A single resource

To hibernate a staging namespace outside working hours, a team creates one resource in that namespace:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: office-hours
  namespace: payments-staging
spec:
  schedule:
    sleepSchedule: "0 20 * * 1-5"   # 20:00 UTC, Monday to Friday
    wakeSchedule: "0 8 * * 1-5"     # 08:00 UTC, Monday to Friday
```

At 20:00, every `Deployment` and `StatefulSet` in `payments-staging` is scaled to zero, and its replica count is recorded in the status of the resource. At 08:00, each workload is restored to its recorded replica count. The namespace remains hibernated over the weekend, as no wake is scheduled between the Friday sleep and Monday morning. When the resource is deleted, its workloads are restored before the deletion completes.

The same schedule, applied through a `ClusterResourceSupervisor`, governs every namespace that the resource selects. Refer to [Key Features](overview/key-features.md) for a full description of both resources.

## Where to go next

### Evaluating the operator

- [Key Features](overview/key-features.md): a description of each feature
- [Use Cases](overview/use-cases.md): the scenarios the operator is designed for
- [Architecture](concepts/architecture.md): the components and the reconcile flow

### Running the operator

- [Installation](getting-started/installation/overview.md): installation on OpenShift or Kubernetes
- [Quick Start](getting-started/quick-start.md): hibernating a first namespace
- [Hibernate Workloads in a Single Namespace](guides/create-resource-supervisor.md): the namespace-scoped resource
- [Hibernate Workloads Across Multiple Namespaces](guides/create-cluster-resource-supervisor.md): the cluster-scoped resource
