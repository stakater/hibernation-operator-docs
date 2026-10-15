---
head:
  - - meta
    - name: keywords
      content: hibernation operator, kubernetes, cost optimization, sleep, wake, cron, argocd
---

# Hibernation Operator

Development, test, demonstration and preview environments are typically in use for only part of the working week, yet their workloads continue to run and to hold the capacity they request. The Hibernation Operator scales these workloads down to zero according to a schedule and restores them to their previous state when they are next required.

Platform teams can apply hibernation to many namespaces through a single resource, while application teams can manage hibernation for their own namespace without requiring cluster-level permissions.

## How it works

1. **Select namespaces:** a `ResourceSupervisor` governs the namespace in which it is created, while a `ClusterResourceSupervisor` selects namespaces by name or by label.
1. **Set a schedule:** a sleep and a wake time, each expressed as a cron expression in UTC.
1. **Sleep:** at the sleep time, every `Deployment` and `StatefulSet` in the selected namespaces is scaled to zero, and its replica count is recorded in the status of the resource.
1. **Wake:** at the wake time, each workload is restored to its recorded replica count.

## Example

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

The namespace sleeps every weekday evening and wakes the following morning. It remains hibernated over the weekend, as no wake is scheduled between the Friday sleep and Monday morning. When the resource is deleted, its workloads are restored before the deletion completes.

## Where to go next

### Evaluating the operator

- [Key Features](overview/key-features.md): a description of each feature
- [Use Cases](overview/use-cases.md): the scenarios the operator is designed for
- [Architecture](concepts/architecture.md): the components and how they behave

### Running the operator

- [Installation](getting-started/installation/overview.md): installation on OpenShift or Kubernetes
- [Quick Start](getting-started/quick-start.md): hibernating a first namespace
- [Hibernate Workloads in a Single Namespace](guides/create-resource-supervisor.md): the namespace-scoped resource
- [Hibernate Workloads Across Multiple Namespaces](guides/create-cluster-resource-supervisor.md): the cluster-scoped resource
