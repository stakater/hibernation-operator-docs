# Quick Start

Get the Hibernation Operator running and put your first workload on a sleep
schedule in a few minutes.

## Prerequisites

- A Kubernetes cluster (v1.24+) with `kubectl` access
- [Helm CLI](https://helm.sh/docs/intro/install/)
- *(Optional)* [cert-manager](https://cert-manager.io/docs/installation/) —
  required only if you enable the validating webhooks

For OpenShift OperatorHub and detailed options, see
[Installation](installation/overview.md).

## 1. Install the operator

```sh
helm install hibernation-operator \
  oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-operator-system \
  --create-namespace
```

Wait for the pods to become ready:

```sh
kubectl get pods -n hibernation-operator-system --watch
```

## 2. Create your first schedule

Create a namespace-scoped `ResourceSupervisor` that scales the workloads in a
namespace down at night and back up in the morning (UTC):

```yaml
# my-first-schedule.yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: nightly-hibernation
  namespace: my-app-staging   # must match the target namespace
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # sleep daily at 20:00 UTC
    wakeSchedule: "0 8 * * *"     # wake daily at 08:00 UTC
```

Apply it:

```sh
kubectl apply -f my-first-schedule.yaml
```

## 3. Verify

```sh
kubectl get resourcesupervisor nightly-hibernation -n my-app-staging \
  -o jsonpath='{.status.currentStatus}'
```

The `currentStatus` field reports `running`, `sleeping`, or `error`.

## Next steps

- [Hibernate Workloads in a Single Namespace](../guides/create-resource-supervisor.md)
- [Hibernate Workloads Across Multiple Namespaces](../guides/create-cluster-resource-supervisor.md)
- [Concepts › Architecture](../concepts/architecture.md) — how it works
- [Reference › API Reference](../reference/api.md) — every field
