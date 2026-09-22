# ArgoCD Integration

When workloads are managed by ArgoCD, hibernation and GitOps pull in opposite directions. Scaling a `Deployment` to zero is drift from the desired state in Git, so ArgoCD syncs it back up and the workloads never stay asleep.

The Hibernation Operator resolves this by writing a `deny` sync window onto the ArgoCD `AppProject`s you name, covering the sleep period. ArgoCD then leaves those applications alone while they are hibernated, and resumes normal syncing afterwards.

!!! warning
    `spec.argocd` does not choose which namespaces to hibernate. It only suppresses ArgoCD syncing. The namespaces still come entirely from `spec.namespaces`, and a `ClusterResourceSupervisor` with an `argocd` block but no `namespaces` block writes sync windows while hibernating nothing.

The integration is optional and must be enabled at install time.

## How it works

When a `ClusterResourceSupervisor` sets `spec.argocd`, the operator:

1. Looks up each named `AppProject` in the namespace given by `spec.argocd.namespace`.
1. Computes the sleep duration from `sleepSchedule` and `wakeSchedule`.
1. Sets a single `deny` sync window on the AppProject, scheduled to match the sleep time and lasting that duration, with manual sync still permitted.

Hibernation of the workloads themselves proceeds independently, driven by `spec.namespaces` as usual.

!!! warning
    The operator replaces `spec.syncWindows` on each named AppProject rather than appending to it. Sync windows you maintain there by other means are overwritten. If you rely on existing sync windows, do not point the operator at that AppProject.

If the ArgoCD `AppProject` CRD is not present in the cluster, the operator logs that and continues hibernating without touching ArgoCD.

## Enabling the integration

Enable ArgoCD support and point the operator at the ArgoCD namespace during installation:

```sh
helm install hibernation-operator oci://ghcr.io/stakater/public/charts/hibernation-operator \
  --namespace hibernation-system \
  --create-namespace \
  --set argoCD.enabled=true \
  --set argoCD.namespace=argocd
```

## Using it

Select the namespaces to hibernate as normal, and list the AppProjects whose syncing should be suppressed while they sleep:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ClusterResourceSupervisor
metadata:
  name: argocd-hibernation-policy
spec:
  namespaces:
    labelSelector:
      matchLabels:
        env: dev
  argocd:
    namespace: argocd          # Namespace where the AppProjects live
    appProjects:
      - frontend-team
      - mobile-apps
  schedule:
    sleepSchedule: "0 18 * * 1-5"   # Sleep weekdays at 18:00 UTC
    wakeSchedule: "0 8 * * 1-5"     # Wake weekdays at 08:00 UTC
```

Apply it:

```sh
kubectl apply -f cluster-resource-supervisor-argocd.yaml
```

## Verifying

Check which namespaces the policy actually manages. These come from `spec.namespaces`, not from the AppProjects:

```sh
kubectl get clusterresourcesupervisor argocd-hibernation-policy -o jsonpath='{.status.watchedNamespaces}'
```

Then confirm the sync window landed on the AppProject:

```sh
kubectl get appproject frontend-team -n argocd -o jsonpath='{.spec.syncWindows}'
```

You should see a single entry with `kind: deny` and a schedule matching your `sleepSchedule`.

## Permissions

ArgoCD must be installed, and the operator's ServiceAccount needs to read and modify AppProjects in the ArgoCD namespace, since writing the sync window is a patch rather than a read:

```yaml
- apiGroups: ["argoproj.io"]
  resources: ["appprojects"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

## Troubleshooting

| Symptom | Check |
| --- | --- |
| No sync window appears on the AppProject | The name in `appProjects` matches the `AppProject` resource name exactly, and `spec.argocd.namespace` is the namespace it lives in. Both are case-sensitive. |
| Sync window exists but nothing hibernates | `spec.namespaces` is set. The `argocd` block alone selects no namespaces. |
| Namespace missing from `status.watchedNamespaces` | `status.ignoreNamespaces`, which lists namespaces filtered out by the `hibernation.stakater.com/exclude` annotation or because they are the operator's own namespace. |
| Workloads wake up during the sleep window | The sync window was overwritten, or the application belongs to an AppProject not listed in `spec.argocd.appProjects`. |
| Permission errors in the operator log | The ServiceAccount has `update` and `patch` on `appprojects.argoproj.io`, not only read verbs. |

## Related pages

- [ClusterResourceSupervisor](../concepts/cluster-resource-supervisor.md)
- [Hibernate Workloads Across Multiple Namespaces](../guides/create-cluster-resource-supervisor.md)
