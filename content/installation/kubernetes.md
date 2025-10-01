# On Kubernetes

This document contains instructions for installing, uninstalling, and configuring the **Hibernation Operator** on Kubernetes.

1. [Installing via Helm CLI](#installing-via-helm-cli)  
1. [Uninstall](#uninstall-via-helm-cli)

## Requirements

* A **Kubernetes** cluster (v1.24 or higher)  
* [Helm CLI](https://helm.sh/docs/intro/install/)  
* [kubectl](https://kubernetes.io/docs/tasks/tools/)  
* *(Optional but recommended)* [cert-manager](https://cert-manager.io/docs/installation/) — required **only if you enable the webhook**

> 💡 The Hibernation Operator uses admission webhooks for CR validation (e.g., cron format checks). If you disable the webhook (`webhook.create=false`), cert-manager is **not required**.

## Installing via Helm CLI

The public Helm chart for the Hibernation Operator is available in the public [**Stakater Helm repository**](https://github.com/orgs/stakater/packages/container/package/public/charts/template-operator).

### Install the Operator

Install into the recommended namespace `hibernation-system`:

```sh
helm install hibernation-operator oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-operator-system \
  --create-namespace
```

* ✅ This installs
* The **Hibernation Controller** (manages both `ClusterResourceSupervisor` and `ResourceSupervisor`)
* The **Webhook** (for validation)
* Required **RBAC**, **CRDs**, and **Service** resources

### Optional: Enable ArgoCD Integration

If you use ArgoCD and want to target AppProjects, enable ArgoCD support:

```sh
helm install hibernation-operator stakater/hibernation-operator \
  --namespace hibernation-system \
  --create-namespace \
  --set argoCD.enabled=true \
  --set argoCD.namespace=argocd
```

### Wait for Pods to Start

```sh
kubectl get pods -n hibernation-operator-system --watch
```

Once all pods are `Running`, you can begin creating hibernation policies:

* [Create a ClusterResourceSupervisor](../how-to-guides/create-cluster-resource-supervisor.md)  
* [Create a ResourceSupervisor](../how-to-guides/create-resource-supervisor.md)

## Uninstall via Helm CLI

To uninstall the Hibernation Operator:

```sh
helm uninstall hibernation-operator --namespace hibernation-operator-system
```

> ⚠️ **Note**: This removes the operator and its RBAC, but **does not delete your CRs** (`ClusterResourceSupervisor`, `ResourceSupervisor`).  
> If you want to fully clean up, delete any remaining CRs first:

```sh
kubectl delete clusterresourcesupervisors.hibernation.stakater.com --all
kubectl delete resourcesupervisors.hibernation.stakater.com --all --all-namespaces
```

Then uninstall the Helm release.

## Notes

* The operator **does not require a database, cache, or external scheduler**—it works entirely with native Kubernetes resources.
* Workloads are only scaled if explicitly targeted by a supervisor. **No namespace is modified by default**.
* Replica counts are stored in the CR’s `status` field, ensuring safe restoration even after operator restarts.
* For production use, consider pinning to a specific chart version:

  ```sh
    helm install hibernation-operator stakater/hibernation-operator --version a.b.ccc ...
  ```
