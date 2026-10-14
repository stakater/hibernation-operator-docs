# Configuration

This page documents the configuration surface of the Hibernation Operator: the
**Helm chart values** you set at install time, the **environment variables**
that toggle controllers and webhooks, and the **manager command-line flags**
that control the metrics, health, and webhook endpoints.

For the full installation flow, see [Installation › Kubernetes](../getting-started/installation/kubernetes.md).

## Helm values

The operator is installed from the OCI chart
`oci://ghcr.io/stakater/public/charts/hibernation-operator`. The table below lists the values most commonly tuned. Inspect the full set with:

```sh
helm show values oci://ghcr.io/stakater/public/charts/hibernation-operator
```

| Value | Default | Description |
|-------|---------|-------------|
| `controllerManager.controllerManager.env.enableRsController` | `"true"` | Enable the `ResourceSupervisor` controller. |
| `controllerManager.controllerManager.env.enableCrsController` | `"true"` | Enable the cluster-scoped `ClusterResourceSupervisor` controller. |
| `controllerManager.controllerManager.env.enableWebhooks` | `"false"` | Enable validating webhooks on the controller deployment. |
| `controllerManager.controllerManager.env.leaderElectionId` | `hibernation-operator-election-id` | Leader-election lock identifier. |
| `controllerManager.controllerManager.image.repository` | `ghcr.io/stakater/public/hibernation-operator` | Operator image repository. |
| `controllerManager.controllerManager.image.tag` | the release tag, for example `v0.1.104` | Operator image tag. |
| `controllerManager.controllerManager.resources` | requests `10m`/`64Mi`, limits `500m`/`128Mi` | Manager container resource requests/limits. |
| `controllerManager.replicas` | `1` | Number of manager replicas. |
| `webhook.manager.env.enableWebhooks` | `"true"` | Enable webhooks on the dedicated webhook deployment. |
| `webhook.manager.resources` | requests `100m`/`20Mi`, limits `2`/`2Gi` | Webhook container resource requests/limits. |
| `webhook.replicas` | `1` | Number of webhook deployment replicas. |
| `imagePullSecrets` | `[]` | Image pull secrets for private registries. |
| `kubernetesClusterDomain` | `cluster.local` | Cluster DNS domain (used for in-cluster service URLs). |

For example, to disable the `ResourceSupervisor` controller at install time:

```sh
helm install hibernation-operator \
  oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-operator-system --create-namespace \
  --set controllerManager.controllerManager.env.enableRsController="false"
```

## Environment variables

These are consumed by the operator process directly (the Helm chart sets them
on the deployment). Flag variables are enabled only when set to the exact
string `"true"`.

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_RS_CONTROLLER` | unset (off) | Enable the `ResourceSupervisor` controller. |
| `ENABLE_CRS_CONTROLLER` | unset (off) | Enable the `ClusterResourceSupervisor` controller. |
| `ENABLE_WEBHOOKS` | unset (off) | Enable the validating webhooks. |
| `OPERATOR_NAME_PREFIX` | `hibernation-operator` | Name prefix used for operator-owned resources. |
| `OPERATOR_NAMESPACE` | auto-detected in-cluster, falling back to `hibernation-operator-system` | Namespace the operator runs in. Workloads in this namespace are never hibernated. |
| `LEADER_ELECTION_ID` | unset | Identifier for the leader-election lock. When unset, leader election is disabled even if `--leader-elect` is passed. |

## Manager flags

Command-line flags accepted by the manager binary (set via the deployment's
container `args`).

| Flag | Default | Description |
|------|---------|-------------|
| `--metrics-bind-address` | `0` (disabled) | Address the metrics endpoint binds to, or `0` to disable it. The Helm chart and the OLM bundle set `:6443`. |
| `--metrics-tls-cert-dir` | `""` | Directory containing `tls.crt` and `tls.key`. When set, metrics are served over HTTPS; otherwise over HTTP. |
| `--metrics-secure` | `true` | Accepted but has no effect. Use `--metrics-tls-cert-dir` to enable HTTPS. |
| `--health-probe-bind-address` | `:8081` | Address for liveness/readiness probes. |
| `--leader-elect` | `false` | Enable leader election for high-availability deployments. |
| `--enable-http2` | `false` | Enable `HTTP/2` for the metrics and webhook servers. Disabled by default to mitigate `HTTP/2` CVEs. |
| `--webhook-cert-path` | `""` | Directory containing the webhook certificate. |
| `--webhook-cert-name` | `tls.crt` | Webhook certificate file name. |
| `--webhook-cert-key` | `tls.key` | Webhook key file name. |
| `--metrics-cert-path` | `""` | Directory containing the metrics server certificate. |
| `--metrics-cert-name` | `tls.crt` | Metrics certificate file name. |
| `--metrics-cert-key` | `tls.key` | Metrics key file name. |

> See [Metrics](metrics.md) for how the metrics endpoint is exposed, and
> [Webhooks](webhooks.md) for webhook certificate management.
