# On Kubernetes

This document contains instructions for installing, uninstalling, and configuring the **Hibernation Operator** on Kubernetes.

1. [Installing via Helm CLI](#installing-via-helm-cli)  
1. [Uninstall](#uninstall-via-helm-cli)

## Requirements

* A **Kubernetes** cluster (v1.24 or higher)  
* [Helm CLI](https://helm.sh/docs/intro/install/)  
* [kubectl](https://kubernetes.io/docs/tasks/tools/)  
* *(Optional but recommended)* [cert-manager](https://cert-manager.io/docs/installation/) — required **only if you enable the webhook**

> The Hibernation Operator uses admission webhooks for CR validation (e.g., cron format checks). If you disable the webhook (`webhook.create=false`), cert-manager is **not required**.

## Installing via Helm CLI

The public Helm chart for the Hibernation Operator is available in the public [**Stakater Helm repository**](https://github.com/orgs/stakater/packages/container/package/public/charts/template-operator).

### Install the Operator

Install into the recommended namespace `hibernation-system`:

```sh
helm install hibernation-operator oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-operator-system \
  --create-namespace
```

This installs:

* The Hibernation Controller, which manages both `ClusterResourceSupervisor` and `ResourceSupervisor`
* The webhook, for admission validation
* The required RBAC, CRDs, and Service resources

### Optional: Enable ArgoCD Integration

If your workloads are managed by ArgoCD, enable this so the operator can stop ArgoCD from syncing hibernated applications back up. See [ArgoCD Integration](../../integrations/argocd.md) for what it does and does not do:

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

* [Hibernate Workloads Across Multiple Namespaces](../../guides/create-cluster-resource-supervisor.md)
* [Hibernate Workloads in a Single Namespace](../../guides/create-resource-supervisor.md)

## Uninstall via Helm CLI

To uninstall the Hibernation Operator:

```sh
helm uninstall hibernation-operator --namespace hibernation-operator-system
```

> **Note**: This removes the operator and its RBAC, but **does not delete your CRs** (`ClusterResourceSupervisor`, `ResourceSupervisor`).
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
