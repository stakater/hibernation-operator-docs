# On Kubernetes

This page describes how to install the Hibernation Operator on Kubernetes with Helm. To remove the operator, refer to [Uninstalling](../uninstalling.md).

## Requirements

* A Kubernetes cluster, version 1.24 or later
* The [Helm CLI](https://helm.sh/docs/intro/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [cert-manager](https://cert-manager.io/docs/installation/), installed in the cluster

The chart always deploys the validating admission webhook, and it issues the webhook and metrics certificates through a cert-manager `Issuer`. cert-manager is therefore required, and the chart provides no option to disable it.

## Install the operator

The chart is published as an OCI artifact in the [Stakater package registry](https://github.com/orgs/stakater/packages/container/package/public/charts/hibernation-operator). Install it into the `hibernation-operator-system` namespace:

```sh
helm install hibernation-operator \
  oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-operator-system \
  --create-namespace
```

For production clusters, pin the chart to a specific version by adding `--version <chart-version>`.

The release creates:

* The `hibernation-operator-controller-manager` deployment, which runs the `ResourceSupervisor` and `ClusterResourceSupervisor` controllers
* The `hibernation-operator-webhook` deployment, which serves the validating admission webhook
* The CRDs, RBAC resources, services and cert-manager certificates

For the available chart values, refer to [Configuration](../../reference/configuration.md).

## Verify the installation

```sh
kubectl get pods -n hibernation-operator-system --watch
```

Once both pods report `Running`, hibernation schedules can be created:

* [Hibernate Workloads in a Single Namespace](../../guides/create-resource-supervisor.md)
* [Hibernate Workloads Across Multiple Namespaces](../../guides/create-cluster-resource-supervisor.md)

The ArgoCD integration requires no installation option. It is enabled per resource through `spec.argocd` on a `ClusterResourceSupervisor`; refer to [ArgoCD Integration](../../integrations/argocd.md).
