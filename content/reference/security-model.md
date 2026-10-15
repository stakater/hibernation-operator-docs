# Security Model

The Hibernation Operator is designed to run safely in shared and multi-tenant
clusters. Its security model rests on four pillars: hardened workloads,
least-privilege RBAC, namespace-level safety controls, and secured transport.

## Pod and container hardening

The controller manager deployment meets the Kubernetes
[restricted Pod Security Standard](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted):

- **Pod:** `runAsNonRoot: true`, `seccompProfile.type: RuntimeDefault`
- **Container:** `allowPrivilegeEscalation: false`, all Linux capabilities
  dropped (`capabilities.drop: ["ALL"]`)

The webhook deployment sets `runAsNonRoot: true` and
`allowPrivilegeEscalation: false`, but it sets no `seccompProfile` and does not
drop capabilities, so it meets the baseline standard rather than the
restricted one.

Both deployments set resource requests and limits.

## Least-privilege RBAC

Both deployments run under the dedicated `hibernation-operator-controller-manager`
ServiceAccount, bound to a tightly scoped `ClusterRole`:

- **Read-only** on `namespaces` and `customresourcedefinitions`.
- **Write** access limited to `Deployments` and `StatefulSets` (for scaling),
  the operator's own custom resources, and `events`.
- `status` and `finalizers` permissions are separated from the main resource
  rules.

No cluster-admin or wildcard permissions are granted. See [RBAC](rbac.md) for
the complete permission set.

## Namespace exclusion safety

Workloads can be protected from hibernation regardless of any matching
supervisor:

- A namespace annotated with `hibernation.stakater.com/exclude: "true"` is
  **always ignored**, and the operator never scales workloads in it.
- The **operator's own namespace is automatically excluded**.

This lets platform teams carve out critical namespaces (or let application teams
opt out of cluster-wide policies) safely. The operator only ever modifies
`Deployments` and `StatefulSets` it explicitly targets; all other resources are
left untouched.

## Transport security

- **Webhooks** are served over TLS. The certificate is issued by cert-manager
  for Helm installations, by OLM on OpenShift, or supplied through flags. See
  [Webhooks](webhooks.md).
- **Metrics** require a bearer token authorized through the Kubernetes API, but
  are served over plain http unless `--metrics-tls-cert-dir` is set. See
  [Metrics](metrics.md).
- **`HTTP/2` is disabled by default** (`--enable-http2=false`) to mitigate known
  `HTTP/2` denial-of-service vulnerabilities (e.g. the "Rapid Reset" class of
  CVEs). Enable it only if you understand the risk.
- No `NetworkPolicy` is shipped. To restrict access to the metrics endpoint,
  create one that limits ingress on port `6443` to your Prometheus instance.
