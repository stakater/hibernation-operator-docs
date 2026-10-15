# Architecture

The Hibernation Operator scales `Deployments` and `StatefulSets` down to zero on a schedule and restores them afterwards, so that environments needed only during working hours stop consuming compute for the rest of the time. It manages two custom resources: the [ResourceSupervisor](resource-supervisor.md), which a namespace owner creates to hibernate their own namespace, and the [ClusterResourceSupervisor](cluster-resource-supervisor.md), which a platform team uses to hibernate a group of namespaces.

## Components

| Deployment | Description |
| --- | --- |
| `hibernation-operator-controller-manager` | Hibernates and restores workloads according to the schedule of each resource, and reports the outcome in its status. |
| `hibernation-operator-webhook` | Validates both resources when they are created or updated. |

## Behaviour

### At the sleep time

Every `Deployment` and `StatefulSet` with a non-zero replica count in the governed namespaces is scaled to zero. Its replica count is recorded in the status of the resource first, so that it can be restored later. Workloads already at zero are left as they are.

### At the wake time

Each workload recorded in the status is restored to its recorded replica count. Workloads that the operator did not hibernate are not modified.

### When a resource is deleted

The workloads of the resource are restored before the deletion completes, and any ArgoCD sync windows it applied are removed. If a restoration fails, the resource is retained until it succeeds.

### When a namespace is excluded or leaves the selection

Namespaces annotated `hibernation.stakater.com/exclude: "true"`, and the namespace in which the operator runs, are never hibernated. When a namespace becomes excluded or is no longer selected, the workloads hibernated there are restored at the next scheduled sleep or wake. Such namespaces are listed in `status.ignoreNamespaces`.

### When the operator restarts

The schedule is evaluated against the current time rather than against a running timer, and the recorded replica counts are stored in the Kubernetes API. A restarted operator therefore continues where it left off.

## ArgoCD

ArgoCD treats a workload scaled to zero as drift from Git and scales it back up, which would undo hibernation. When `spec.argocd` is set on a `ClusterResourceSupervisor`, the operator applies a `deny` sync window to the specified AppProjects for the duration of the sleep period.

!!! note
    This is the operator's only interaction with ArgoCD. AppProjects do not determine which namespaces are hibernated; these are always selected through `spec.namespaces`.

## Admission validation

The webhook rejects a resource when:

- A cron expression in `spec.schedule` cannot be parsed.
- `sleepSchedule` and `wakeSchedule` are identical.
- `wakeSchedule` is set without `sleepSchedule`.
- A `ClusterResourceSupervisor` lists the same namespace twice, or selects a namespace already selected by another `ClusterResourceSupervisor`.

An empty schedule is valid and hibernates the workloads immediately and indefinitely. The webhook does not prevent a namespace from being governed by both a `ClusterResourceSupervisor` and its own `ResourceSupervisor`, so such overlaps must be avoided by convention.

The metrics endpoint is described in [Metrics](../reference/metrics.md).
