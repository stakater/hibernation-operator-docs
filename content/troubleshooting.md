# Troubleshooting Guide

## Finding why a sleep or wake failed

When `status.currentStatus` is `error`, the reason is on the resource itself, no operator logs needed.

```sh
kubectl get clusterresourcesupervisor <name> -o jsonpath='{.status.conditions}'
kubectl get resourcesupervisor <name> -n <namespace> -o jsonpath='{.status.conditions}'
```

The `Ready` condition is `True` when the workloads are in the state the schedule asks for, and `False` with one of these reasons when they are not:

| Reason | Meaning |
| --- | --- |
| `Reconciled` | The last sleep or wake succeeded (`True`) |
| `Scheduled` | The resource is new and nothing is due yet (`True`) |
| `InvalidSleepSchedule` | `sleepSchedule` is not a valid cron. The operator stops acting on the resource until it is fixed |
| `InvalidWakeSchedule` | `wakeSchedule` is not a valid cron. Same as above |
| `SleepFailed` | At least one workload could not be scaled down. The operator retries automatically |
| `WakeFailed` | At least one workload could not be restored. The operator retries automatically |
| `NamespacesUnresolved` | The governed namespaces could not be listed |

The condition `message` carries the underlying error. After a failed wake, each application still asleep also carries its own `errorMessage`, and its namespace counts them:

```yaml
status:
  currentStatus: error
  conditions:
    - type: Ready
      status: "False"
      reason: WakeFailed
      message: 'namespace team-a, application web: deployments.apps "web" is forbidden: ...'
  sleepingNamespaces:
    - Namespace: team-a
      status: error
      errorMessage: 1 of 2 applications failed to wake
      sleepingApplications:
        - name: web
          kind: Deployment
          replicas: 3
          status: error
          errorMessage: 'deployments.apps "web" is forbidden: ...'
```

The next sleep clears both `errorMessage` fields.

## Workloads stay at zero after a wake

Check the resource's Events for a `LedgerLost` Warning. The replica counts live only in `status.sleepingNamespaces`, so if the status was lost the operator has nothing to restore from and leaves the workloads at zero rather than guessing. Scale them back by hand.

A wake only restores workloads the operator slept. A workload that was already at zero when the sleep ran, or was created while the namespace slept, is left alone.

## Schedules fire at the wrong time

Both cron schedules are evaluated in UTC, never in the cluster's or the browser's timezone. `sleepSchedule: "0 18 * * *"` sleeps at 18:00 UTC.
