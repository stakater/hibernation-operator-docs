---
head:
  - - meta
    - name: keywords
      content: hibernation operator, kubernetes, cost optimization, sleep, wake, cron, argocd
---

# Welcome to the Docs

Managing Kubernetes clusters at scale often leads to underutilized resources—especially in development, staging, and CI/CD environments that run 24/7 but are only actively used during business hours. This results in unnecessary cloud spend and inefficient resource utilization.

The **Hibernation Operator** solves this by enabling **automated, policy-driven hibernation** of workloads. It safely scales down `Deployments` and `StatefulSets` to zero during off-hours and restores them to their original state when needed—helping teams reduce costs without sacrificing developer experience.

With the Hibernation Operator, you can:

- **Schedule hibernation** using standard cron expressions for sleep and wake times.
- **Target workloads** across multiple namespaces using explicit lists, dynamic label selectors, or **ArgoCD AppProjects** (for GitOps-aligned teams).
- **Empower platform teams** with cluster-wide policies via `ClusterResourceSupervisor`.
- **Enable self-service** for application teams using namespace-scoped `ResourceSupervisor`.
- **Preserve state reliably**: Original replica counts are stored in the CR’s status for accurate restoration—even after operator restarts.
- **Integrate seamlessly** with existing Kubernetes and ArgoCD workflows—no external dependencies required.

The Hibernation Operator is lightweight, secure, and built for real-world Kubernetes environments—whether you're running on OpenShift, EKS, AKS, GKE, or vanilla Kubernetes.

## Installation

Refer to the [installation guide](./installation/overview.md) to deploy the Hibernation Operator in your cluster via **Helm** or **Operator Lifecycle Manager (OLM)**.
