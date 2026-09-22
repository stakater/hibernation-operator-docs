# Webhooks

The Hibernation Operator provides **validating admission webhooks** for both
custom resources. They reject invalid resources at admission time — before they
are persisted — primarily to catch malformed cron schedules and conflicting
namespace targeting.

## Overview

- **Type:** validating only (no mutating webhooks).
- **Resources / operations:** `resourcesupervisors` and
  `clusterresourcesupervisors`, on `CREATE` and `UPDATE`.
- **Failure policy:** `Fail` — if the webhook is unavailable, the request is
  rejected (`sideEffects: None`).
- **Enablement:** off by default; enabled via the `ENABLE_WEBHOOKS` environment
  variable. The Helm chart runs a dedicated webhook deployment with webhooks
  enabled (see [Configuration](configuration.md)).
- **Port:** the webhook server listens on `9443`.

The webhook endpoints are:

- `/validate-hibernation-stakater-com-v1beta1-resourcesupervisor` (`vresourcesupervisor.kb.io`)
- `/validate-hibernation-stakater-com-v1beta1-clusterresourcesupervisor` (`vclusterresourcesupervisor.kb.io`)

## Schedule validation (both resources)

Cron schedules are parsed using standard cron syntax. The webhook enforces:

- **Empty schedules are allowed** — both `sleepSchedule` and `wakeSchedule`
  empty (immediate/permanent sleep use cases) passes validation.
- If a `wakeSchedule` is provided, both `sleepSchedule` and `wakeSchedule` must
  be **valid cron expressions**.
- `sleepSchedule` and `wakeSchedule` **must not be identical**.
- When neither schedule uses a wildcard (`*`), both must resolve to a time **in
  the future**, and the **wake time must be after the sleep time**. Schedules
  containing `*` are always considered valid.

## ClusterResourceSupervisor namespace validation

In addition to schedule validation, the `ClusterResourceSupervisor` webhook
enforces namespace uniqueness:

- **No duplicate namespace names** within a single resource's
  `spec.namespaces.names`.
- **No overlap across resources** — a namespace targeted by one
  `ClusterResourceSupervisor` cannot be targeted by another. This prevents two
  cluster supervisors from fighting over the same namespace.

## Certificates

The webhook server requires TLS. The operator integrates with
[cert-manager](https://cert-manager.io):

- A self-signed `Issuer` (`selfsigned-issuer`) issues a `Certificate` stored in
  the secret `hibernation-operator-webhook-server-cert`.
- DNS names cover the webhook service (`<service>.<namespace>.svc` and the
  `.svc.cluster.local` variant).

Alternatively, you can supply certificates directly via the
`--webhook-cert-path`, `--webhook-cert-name`, and `--webhook-cert-key` flags.

> cert-manager is a prerequisite only when webhooks are enabled. See
> [Installation › Kubernetes](../getting-started/installation/kubernetes.md).
