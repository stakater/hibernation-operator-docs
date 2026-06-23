# Security Model

The Hibernation Operator is designed to run safely in shared and multi-tenant
clusters. Its security model rests on four pillars: hardened workloads,
least-privilege RBAC, namespace-level safety controls, and secured transport.

## Pod and container hardening

The operator and webhook deployments adhere to the Kubernetes
[restricted Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted):

- **Pod:** `runAsNonRoot: true`, `seccompProfile.type: RuntimeDefault`
- **Container:** `allowPrivilegeEscalation: false`, all Linux capabilities
  dropped (`capabilities.drop: ["ALL"]`)
- Resource requests and limits are set to bound resource consumption.

## Least-privilege RBAC

The operator runs under a dedicated `controller-manager` ServiceAccount with a
tightly scoped `ClusterRole`:

- **Read-only** on `namespaces` and `customresourcedefinitions`.
- **Write** access limited to `Deployments` and `StatefulSets` (for scaling) and
  the operator's own custom resources.
- `status` and `finalizers` permissions are separated from the main resource
  rules.

No cluster-admin or wildcard permissions are granted. See [RBAC](rbac.md) for
the complete permission set.

## Namespace exclusion safety

Workloads can be protected from hibernation regardless of any matching
supervisor:

- A namespace annotated with `hibernation.stakater.com/exclude: "true"` is
  **always ignored** — the operator never scales workloads in it.
- The **operator's own namespace is automatically excluded**.

This lets platform teams carve out critical namespaces (or let application teams
opt out of cluster-wide policies) safely. The operator only ever modifies
`Deployments` and `StatefulSets` it explicitly targets; all other resources are
left untouched.

## Transport security

- **Webhooks** are served over TLS, with certificates managed by cert-manager
  (or supplied via flags). See [Webhooks](webhooks.md).
- **Metrics** are served over HTTPS with bearer-token authentication by default.
  See [Metrics](metrics.md).
- **HTTP/2 is disabled by default** (`--enable-http2=false`) to mitigate known
  HTTP/2 denial-of-service vulnerabilities (e.g. the "Rapid Reset" class of
  CVEs). Enable it only if you understand the risk.
- An optional `NetworkPolicy` is provided to restrict ingress to the metrics
  endpoint.
