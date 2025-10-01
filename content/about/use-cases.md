# 🧩 Use Cases

## ✅ **`ClusterResourceSupervisor` – Cluster-Scoped Hibernation**

### 1. **Platform Team Managing Dev/Test Environments**

> **Scenario**: A central platform team operates a shared Kubernetes cluster for 50+ development teams. To reduce cloud costs, they want all non-production namespaces to sleep during nights and weekends.  
> **Solution**: Create a single `ClusterResourceSupervisor` with a label selector like `env in (dev, test, staging)` and a sleep schedule of `0 18 * * 1-5` (sleep at 6 PM on weekdays) and wake at `0 8 * * 1-5`.

### 2. **GitOps-Driven Hibernation via ArgoCD AppProjects**

> **Scenario**: Your organization uses ArgoCD with AppProjects to group applications (e.g., `frontend-team`, `data-platform`). You want to hibernate entire AppProjects during off-hours.  
> **Solution**: Define a `ClusterResourceSupervisor` targeting `appProjects: ["frontend-team", "data-platform"]` in the `argocd` namespace. The operator automatically discovers all namespaces managed by those AppProjects and applies hibernation.

### 3. **Temporary Cluster-Wide Freeze for Maintenance**

> **Scenario**: During a company-wide holiday break, you want to pause all non-critical workloads across the cluster.  
> **Solution**: Deploy a `ClusterResourceSupervisor` with an empty `labelSelector` (which matches all namespaces) and no `wakeSchedule`. Workloads stay asleep until manually removed or overridden.

### 4. **Cost Optimization in Multi-Tenant Clusters**

> **Scenario**: You run a multi-tenant cluster where each tenant has a namespace labeled with `tenant-id`. Finance requires cost reports per tenant, and hibernation is part of the SLA.  
> **Solution**: Use label selectors (`matchLabels: { tenant-tier: basic }`) to hibernate only lower-tier tenants, while premium tenants remain running.

---

## ✅ **`ResourceSupervisor` – Namespace-Scoped Hibernation**

### 1. **Application Team Self-Service Hibernation**

> **Scenario**: A product team owns the `payment-service-staging` namespace and wants it to sleep every night to save costs, but wake up by 9 AM for QA.  
> **Solution**: The team creates a `ResourceSupervisor` in their namespace with `sleepSchedule: "0 20 * * *"` and `wakeSchedule: "0 9 * * *"`. No cluster admin involvement needed.

### 2. **CI/CD Dynamic Environments**

> **Scenario**: Your CI pipeline spins up ephemeral namespaces for PR previews (e.g., `pr-1234`). You want them to auto-sleep after 2 hours and never wake up.  
> **Solution**: The CI job creates a `ResourceSupervisor` with a one-time-like cron (e.g., using a tool that schedules a future sleep) or a short recurring sleep with no wake. Alternatively, pair with a TTL controller—but `ResourceSupervisor` gives fine-grained control over *what* sleeps (Deployments/StatefulSets).

### 3. **Training or Demo Environments**

> **Scenario**: You provide demo environments for sales or training that should only run during business hours.  
> **Solution**: In each demo namespace (`demo-customer-x`), deploy a `ResourceSupervisor` with weekday business-hour schedules. Easy to template and replicate.

### 4. **Compliance-Driven Runtime Restrictions**

> **Scenario**: A regulated workload must only run during approved maintenance windows (e.g., weekends).  
> **Solution**: Namespace owners define a `ResourceSupervisor` that enforces strict `sleepSchedule`/`wakeSchedule` aligned with compliance policies.

---

## 🔄 Complementary Use Case: **Hybrid Governance**

> **Scenario**: Your platform provides a default hibernation policy for all `env=dev` namespaces via `ClusterResourceSupervisor`, **but** allows teams to override it with a local `ResourceSupervisor` if they need custom behavior.  
> **Implementation**: Your operator is designed to **skip namespaces** that contain a `ResourceSupervisor`, giving precedence to namespace-scoped control. This enables both standardization and flexibility.
