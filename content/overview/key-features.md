# Key Features

## 1. `ClusterResourceSupervisor` (Cluster-Scoped)

The `ClusterResourceSupervisor` enables **platform-level hibernation management** across multiple namespaces or ArgoCD AppProjects. Ideal for cluster administrators and GitOps-driven environments.

### Key Features

- **Cluster-wide scope**: Applies hibernation policies across the entire cluster.
- **Flexible namespace targeting**:
    - Explicit list of namespace names (`names`)
    - Dynamic selection via Kubernetes-standard **label selectors** (`matchLabels` and `matchExpressions`)
- **ArgoCD integration**:
    - Hold off ArgoCD syncing on named **AppProjects** while their namespaces sleep
    - Specify the ArgoCD namespace where AppProjects reside
- **Cron-based scheduling**, evaluated in UTC:
    - `sleepSchedule`: Cron expression to scale down workloads
    - `wakeSchedule` (optional): Cron expression to restore workloads; if omitted, resources remain asleep
- **Comprehensive status tracking**:
    - `currentStatus`: Overall state (`sleeping`, `running`, `error`)
    - `watchedNamespaces` / `ignoreNamespaces`: Lists of affected and excluded namespaces
    - `sleepingNamespaces`: Detailed records of scaled-down applications, including:
        - Namespace name
        - Application kind (`Deployment` or `StatefulSet`)
        - Original replica count (for accurate restoration)
        - Per-namespace and per-application status, with an `errorMessage` after a failed wake
    - `nextReconcileTime`: Predictable next action time (`ISO 8601 datetime`)
    - `conditions`: `Ready`, with the reason when the last sleep or wake failed

> **Use Case**: Enforce cost-saving hibernation for all `env=dev` namespaces, holding off ArgoCD syncing for the `platform-team` AppProject while they sleep.

---

## 2. `ResourceSupervisor` (Namespace-Scoped)

The `ResourceSupervisor` provides **self-service hibernation** within a single namespace. Designed for application teams who want autonomy without cluster-wide permissions.

### Key Features

- **Namespace-scoped**: Only affects resources in the same namespace where the CR is created.
- **Simple configuration**:
    - Minimal spec with just a `schedule` block
    - No need to manage namespace lists or label selectors
- **Cron-based scheduling**, evaluated in UTC:
    - `sleepSchedule`: Cron expression to hibernate workloads
    - `wakeSchedule` (optional): Cron expression to wake workloads; if not set, workloads stay asleep indefinitely
- **Comprehensive status tracking**:
    - `currentStatus`: Overall state (`sleeping`, `running`, `error`)
    - `nextReconcileTime`: Predictable next action time (`ISO 8601 datetime`)
    - `sleepingNamespaces`: Detailed records of scaled-down applications within its namespace, including:
        - Namespace name
        - Application kind (`Deployment` or `StatefulSet`)
        - Original replica count (for accurate restoration)
        - Per-namespace and per-application status, with an `errorMessage` after a failed wake
    - `conditions`: `Ready`, with the reason when the last sleep or wake failed

> **Use Case**: A development team creates a `ResourceSupervisor` in their `myapp-staging` namespace to sleep workloads every night and wake them each morning.

---

## When to Use Which?

| Scenario | Recommended CRD |
|--------|------------------|
| Cluster admin managing hibernation for many teams | `ClusterResourceSupervisor` |
| GitOps with ArgoCD AppProjects | `ClusterResourceSupervisor` |
| Team wants full control over their namespace | `ResourceSupervisor` |
