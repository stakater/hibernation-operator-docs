# RBAC

The Hibernation Operator ships with a least-privilege set of RBAC roles: the
permissions the controller itself needs to operate, and a set of user-facing
roles you can bind to your own users and groups for self-service access to the
custom resources.

## Operator (manager) permissions

The operator's `manager-role` `ClusterRole` grants only what is required to
watch the custom resources and scale workloads:

| API group | Resources | Verbs |
|-----------|-----------|-------|
| `""` (core) | `namespaces` | get, list, watch |
| `apiextensions.k8s.io` | `customresourcedefinitions` | get, list, watch |
| `apps` | `deployments`, `statefulsets` | get, list, patch, update, watch |
| `hibernation.stakater.com` | `clusterresourcesupervisors`, `resourcesupervisors` | create, delete, get, list, patch, update, watch |
| `hibernation.stakater.com` | `clusterresourcesupervisors/status`, `resourcesupervisors/status` | get, patch, update |
| `hibernation.stakater.com` | `clusterresourcesupervisors/finalizers`, `resourcesupervisors/finalizers` | update |

Note the operator has **read-only** access to namespaces and CRDs, and can
**only** modify `Deployments` and `StatefulSets` (for scaling) — it never
touches other workload types or arbitrary resources. See the
[Security Model](security-model.md) for how this supports safe operation.

## User-facing roles

For each custom resource the chart provides admin, editor, and viewer
`ClusterRole`s intended for binding to your users:

| Role | ResourceSupervisor | ClusterResourceSupervisor |
|------|--------------------|---------------------------|
| **admin** | full access (`*`) to the resource; `get` on status | full access (`*`) to the resource; `get` on status |
| **editor** | create, delete, get, list, patch, update, watch; `get` on status | create, delete, get, list, patch, update, watch; `get` on status |
| **viewer** | get, list, watch | get, list, watch |

Bind one of these to a user or group, for example:

```sh
kubectl create rolebinding alice-rs-editor \
  --clusterrole=resourcesupervisor-editor-role \
  --user=alice \
  --namespace=my-app-staging
```

## Leader election and metrics roles

- **Leader election** — a namespaced `Role` granting access to `coordination.k8s.io`
  leases and events, used only when `--leader-elect` is enabled.
- **Metrics access** — a `metrics-reader` role (read the `/metrics` endpoint) and
  a `metrics-auth` role used by the authentication/authorization filter that
  protects the metrics endpoint. See [Metrics](metrics.md).
