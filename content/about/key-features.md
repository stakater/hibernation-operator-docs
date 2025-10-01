# 🌐 Key Features

## **1. `ClusterResourceSupervisor` (Cluster-Scoped)**

The `ClusterResourceSupervisor` enables **platform-level hibernation management** across multiple namespaces or ArgoCD AppProjects. Ideal for cluster administrators and GitOps-driven environments.

### ✅ Key Features

- **Cluster-wide scope**: Applies hibernation policies across the entire cluster.
- **Flexible namespace targeting**:
    - Explicit list of namespace names (`names`)
    - Dynamic selection via Kubernetes-standard **label selectors** (`matchLabels` and `matchExpressions`)
- **ArgoCD integration**:
    - Target hibernation by **ArgoCD AppProject names**
    - Specify the ArgoCD namespace where AppProjects reside
- **Cron-based scheduling**:
    - `sleepSchedule`: Cron expression to scale down workloads
    - `wakeSchedule` (optional): Cron expression to restore workloads; if omitted, resources remain asleep
- **Comprehensive status tracking**:
    - `currentStatus`: Overall state (`sleeping`, `running`, `error`)
    - `watchedNamespaces` / `ignoreNamespaces`: Lists of affected and excluded namespaces
    - `sleepingNamespaces`: Detailed records of scaled-down applications, including:
        - Namespace name
        - Application kind (`Deployment` or `StatefulSet`)
        - Original replica count (for accurate restoration)
        - Per-namespace and per-application status
    - `nextReconcileTime`: Predictable next action time (ISO 8601 datetime)

> 💡 **Use Case**: Enforce cost-saving hibernation for all `env=dev` namespaces or all applications in the `platform-team` ArgoCD AppProject.

---

## **2. `ResourceSupervisor` (Namespace-Scoped)**

The `ResourceSupervisor` provides **self-service hibernation** within a single namespace. Designed for application teams who want autonomy without cluster-wide permissions.

### ✅ Key Features

- **Namespace-scoped**: Only affects resources in the same namespace where the CR is created.
- **Simple configuration**:
    - Minimal spec with just a `schedule` block
    - No need to manage namespace lists or label selectors
- **Cron-based scheduling**:
    - `sleepSchedule`: Required cron expression to hibernate workloads
    - `wakeSchedule` (optional): Cron expression to wake workloads; if not set, workloads stay asleep indefinitely
- **Lightweight status**:
    - `currentStatus`: Current hibernation state (`sleeping`, `running`, `error`)
    - `nextReconcileTime`: Next scheduled action (ISO 8601 datetime)

> 💡 **Use Case**: A development team creates a `ResourceSupervisor` in their `myapp-staging` namespace to sleep workloads every night and wake them each morning.

---

## 🎯 When to Use Which?

| Scenario | Recommended CRD |
|--------|------------------|
| Cluster admin managing hibernation for many teams | `ClusterResourceSupervisor` |
| GitOps with ArgoCD AppProjects | `ClusterResourceSupervisor` |
| Team wants full control over their namespace | `ResourceSupervisor` |
| Multi-tenant platform with mixed governance | **Both** — use cluster for defaults, namespace for overrides |
