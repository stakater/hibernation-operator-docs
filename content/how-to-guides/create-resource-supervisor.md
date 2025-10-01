# Setup ResourceSupervisor

## Creating a `ResourceSupervisor`

This guide explains how to create a **namespace-scoped** `ResourceSupervisor` to hibernate workloads in a single namespace.

### Prerequisites

- The **Hibernation Operator** must be [installed](../installation/kubernetes.md) in your cluster.
- You have **edit** (or equivalent) permissions in the target namespace.

### Step 1: Create a `ResourceSupervisor` YAML

Create a file named `resource-supervisor.yaml`:

```yaml
apiVersion: hibernation.stakater.com/v1beta1
kind: ResourceSupervisor
metadata:
  name: my-namespace-hibernation
  namespace: my-app-staging  # ← Must match the namespace you want to hibernate
spec:
  schedule:
    sleepSchedule: "0 20 * * *"   # Sleep every day at 8 PM UTC
    wakeSchedule: "0 8 * * *"    # Wake up every day at 8 AM UTC
```

> 💡 **Cron Format**: Uses standard Unix cron syntax (`minute hour day month weekday`).  
> Timezone is **UTC**. Use tools like [crontab.guru](https://crontab.guru) to validate schedules.

### Step 2: Apply the Resource

```sh
kubectl apply -f resource-supervisor.yaml
```

### Step 3: Verify Status

Check the status to confirm the hibernation state:

```sh
kubectl get resourcesupervisor my-namespace-hibernation -n my-app-staging -o yaml
```

Look for:

- `status.currentStatus`: `running` or `sleeping`
- `status.nextReconcileTime`: Next scheduled action (ISO 8601)

### Notes

- Only `Deployments` and `StatefulSets` in the **same namespace** are affected.
- The operator **scales them to 0 replicas** during sleep and **restores original replica counts** on wake.
- If `wakeSchedule` is omitted, workloads stay asleep until manually deleted or updated.
[text](resource-supervisor.md)