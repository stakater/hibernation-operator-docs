# Metrics

The Hibernation Operator exposes Prometheus metrics through the [controller-runtime](https://book.kubebuilder.io/reference/metrics) metrics server.

## Metrics endpoint

The metrics server is controlled by the `--metrics-bind-address` flag. The binary disables it by default (`0`), but both the Helm chart and the OLM bundle start the controller manager with `--metrics-bind-address=:6443`, so an installed operator serves metrics on port `6443` at the path `/metrics`.

The server uses TLS only when `--metrics-tls-cert-dir` points to a directory containing `tls.crt` and `tls.key`. Neither installation method sets this flag, so the endpoint is served over plain http by default. The `--metrics-secure` flag is accepted but has no effect.

Requests pass through the controller-runtime authentication and authorization filter, which validates the caller's bearer token against the Kubernetes API. The scraping principal therefore needs the `hibernation-operator-metrics-reader` ClusterRole. Refer to [RBAC](rbac.md).

The webhook deployment does not serve metrics.

## Scraping

Neither installation method ships a `ServiceMonitor`. To scrape the operator with the Prometheus Operator, create a `Service` that targets port `6443` of the `hibernation-operator-controller-manager` pods, and a `ServiceMonitor` that selects it and authenticates with a token bound to the metrics reader role.

## Available metrics

The operator exports the standard controller-runtime metrics and defines no hibernation-specific metrics. These include:

* `controller_runtime_reconcile_total`: reconciles per controller and result
* `controller_runtime_reconcile_errors_total`: reconcile errors per controller
* `controller_runtime_reconcile_time_seconds`: reconcile latency histogram
* `workqueue_*`: depth, additions, latency and retries of the controller work queues
* `rest_client_*`: Kubernetes API client request metrics

These metrics describe the reconcile health, error rates and API client behaviour of both the `ResourceSupervisor` and `ClusterResourceSupervisor` controllers.
