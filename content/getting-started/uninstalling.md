# Uninstalling

How to remove the Hibernation Operator. The method depends on how you installed
it. Uninstalling the operator does **not** wake any sleeping workloads on its
own — see [Clean up custom resources](#clean-up-custom-resources) below.

## Helm

Uninstall the Helm release:

```sh
helm uninstall hibernation-operator --namespace hibernation-operator-system
```

This removes the operator deployment, RBAC, and services. Optionally remove the
namespace once it is empty:

```sh
kubectl delete namespace hibernation-operator-system
```

## OpenShift (OLM)

If you installed via the OpenShift OperatorHub:

1. In the web console, go to **Operators → Installed Operators**.
2. Select **Hibernation Operator** and choose **Uninstall Operator**.

Or with the CLI, delete the `Subscription` and `ClusterServiceVersion`:

```sh
oc delete subscription hibernation-operator -n <install-namespace>
oc delete csv -n <install-namespace> -l operators.coreos.com/hibernation-operator.<install-namespace>
```

See [Installation › OpenShift OperatorHub](installation/openshift.md) for the
install-namespace you used.

## Clean up custom resources

Uninstalling the operator leaves your custom resources and their CRDs in place.
Before deleting the CRDs, wake any workloads you still need, then remove the
resources:

```sh
kubectl delete clusterresourcesupervisors.hibernation.stakater.com --all
kubectl delete resourcesupervisors.hibernation.stakater.com --all --all-namespaces
```

To remove the CRDs entirely (this deletes **all** remaining instances of these
resources):

```sh
kubectl delete crd \
  resourcesupervisors.hibernation.stakater.com \
  clusterresourcesupervisors.hibernation.stakater.com
```

> ⚠️ Deleting a `ResourceSupervisor` or `ClusterResourceSupervisor` stops it
> from managing its targets, but workloads currently scaled to zero are not
> automatically restored unless the resource is deleted while a wake is due.
> Scale any needed workloads back up manually if required.
