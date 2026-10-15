# Concepts

## Core Resources

### ClusterResourceSupervisor

A **ClusterResourceSupervisor** is a **cluster-scoped** custom resource that defines hibernation schedules for **multiple namespaces** across the cluster. It enables platform administrators to centrally manage cost-saving policies by targeting namespaces either explicitly by name, dynamically via **Kubernetes label selectors**, or through **ArgoCD AppProjects**. The operator scales down `Deployments` and `StatefulSets` to zero during sleep windows and restores them to their original replica counts during wake windows. Its rich status field tracks watched, ignored, and sleeping namespaces along with preserved replica states for reliable recovery.

### ResourceSupervisor

A **ResourceSupervisor** is a **namespace-scoped** custom resource that allows **application teams or namespace owners** to define hibernation schedules **within their own namespace**. It provides a lightweight, self-service mechanism to scale workloads down and up based on cron expressions—without requiring cluster-level permissions. This resource is ideal for teams managing staging, CI/CD preview, or demo environments that should run only during specific hours.

### Hibernation Schedule

The **Hibernation Schedule** is defined via two optional cron expressions:

- **`sleepSchedule`**: Specifies when workloads should be scaled to zero. Leaving both empty sleeps the workloads immediately.
- **`wakeSchedule`** (optional): Specifies when workloads should be restored.  

> If `wakeSchedule` is omitted, resources remain asleep indefinitely until manually woken or the CR is updated. Both schedules use standard Unix cron format (e.g., `"0 18 * * 1-5"`) and are evaluated in UTC, never in the cluster's or the browser's timezone.

### Sleeping State

The **Sleeping State** refers to the condition where a `Deployment` or `StatefulSet` has been scaled to **0 replicas** by the Hibernation Operator. The original replica count is preserved in the CR’s `status.sleepingNamespaces` field, and waking restores exactly that count. Only workloads the supervisor slept are woken, everything else remains untouched.

## Integration Concepts

### ArgoCD AppProject Integration

When a `ClusterResourceSupervisor` specifies `argocd.appProjects`, the operator writes a `deny` sync window onto those ArgoCD **AppProjects** so ArgoCD does not scale the hibernated workloads back up. It does not select namespaces: those still come only from `spec.namespaces`.

### Label-Based Namespace Selection

Instead of listing namespaces individually, platform teams can use **Kubernetes-standard label selectors** (`matchLabels` and `matchExpressions`) to dynamically include namespaces that match certain criteria (e.g., `env: dev`, `team: frontend`). This supports scalable, policy-driven hibernation in large clusters.
