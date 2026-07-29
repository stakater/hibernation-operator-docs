# Metrics

The Hibernation Operator exposes Prometheus-compatible metrics through the
[controller-runtime](https://book.kubebuilder.io/reference/metrics) framework.

## Metrics endpoint

The metrics server is **disabled by default** (`--metrics-bind-address=0`). When
enabled it serves on:

- **`:8443`** over https when `--metrics-secure=true` (the default), or
- **`:8080`** over http when `--metrics-secure=false`.

The endpoint path is `/metrics`. In secure mode, requests are authenticated and
authorized using the Kubernetes API (bearer token), so only principals granted
the metrics-reader permission can scrape it. See
[Configuration](configuration.md) for the relevant flags and
[RBAC](rbac.md) for the metrics roles.

## ServiceMonitor

A Prometheus Operator `ServiceMonitor` (`controller-manager-metrics-monitor`)
is provided to scrape the endpoint:

- **scheme:** https, **port:** `https`, **path:** `/metrics`
- **auth:** bearer token from `/var/run/secrets/kubernetes.io/serviceaccount/token`
- **selector:** matches the manager service labels
  (`control-plane: hibernation-controller-manager`,
  `app.kubernetes.io/name: hibernation-operator`)

> By default the `ServiceMonitor` uses `insecureSkipVerify: true`. For
> production, enable cert-manager and reference a managed certificate so
> Prometheus verifies the metrics server's TLS certificate.

## Available metrics

The operator exports the **standard controller-runtime metrics** — there are
currently no custom hibernation-specific metrics. These include, among others:

- `controller_runtime_reconcile_total` — reconciliations per controller and result
- `controller_runtime_reconcile_errors_total` — reconcile errors per controller
- `controller_runtime_reconcile_time_seconds` — reconcile latency histogram
- `workqueue_*` — depth, adds, latency, and retries of the controller work queues
- `rest_client_*` — Kubernetes API client request metrics

Use these to monitor reconciliation health, error rates, and API client
behavior for both the `ResourceSupervisor` and `ClusterResourceSupervisor`
controllers.
