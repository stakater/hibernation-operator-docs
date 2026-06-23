# Benefits

## ✅ **Benefits of `ClusterResourceSupervisor` (Cluster-Scoped)**

1. **Centralized Control**  
   - Enables cluster administrators to define **global hibernation policies** that apply across many namespaces or ArgoCD AppProjects.
   - Ideal for cost optimization in large environments (e.g., dev/test clusters that sleep nights and weekends).

1. **Flexible Namespace Targeting**  
   - Supports **explicit namespace lists** (`names`) **and** **dynamic selection via label selectors** (`matchLabels`, `matchExpressions`).
   - Allows automatic inclusion of new namespaces that match certain labels (e.g., `env: dev`).

1. **ArgoCD Integration**  
   - Can target **ArgoCD AppProjects**, enabling hibernation based on GitOps application boundaries.
   - Useful in GitOps workflows where applications are grouped into projects—no need to track individual namespaces.

1. **Rich Observability & auditability**  
   - Detailed `status` includes:
     - `watchedNamespaces` / `ignoreNamespaces`
     - `sleepingNamespaces` with per-application replica counts and kinds (`Deployment`/`StatefulSet`)
   - Helps operators debug, verify, and report on hibernation impact.

1. **Cluster-Wide Efficiency**  
   - One CR can manage dozens or hundreds of namespaces—reducing configuration sprawl.

---

## ✅ **Benefits of `ResourceSupervisor` (Namespace-Scoped)**

1. **Self-Service for Teams**  
   - Application owners or namespace tenants can define their **own hibernation schedule** without cluster-level permissions.
   - Empowers autonomy while still enabling cost savings.

1. **Simplicity & Low Overhead**  
   - Minimal spec: only requires a `schedule` (sleep/wake cron expressions).
   - No need to understand label selectors, ArgoCD, or cross-namespace concerns.

1. **Namespace Isolation**  
   - Each namespace can have **independent schedules**—e.g., staging sleeps weekends, QA sleeps every night.
   - Avoids "one-size-fits-all" policies that may not suit all workloads.

1. **Lightweight Status**  
   - Status tracks only `currentStatus` and `nextReconcileTime`—sufficient for local use cases.
   - Reduces cognitive load and storage overhead.

1. **RBAC-Friendly**  
   - Can be managed by users with only **namespace-level roles** (e.g., `edit` or custom roles), improving security posture.

---

## 🎯 When to Use Which?

| Need | Solved By |
|------|-----------|
| Platform team managing 100 dev namespaces | `ClusterResourceSupervisor` |
| Developer team wanting to hibernate their own namespace | `ResourceSupervisor` |
| GitOps-driven environment using ArgoCD AppProjects | `ClusterResourceSupervisor` |
| Avoiding privilege escalation for namespace owners | `ResourceSupervisor` |
