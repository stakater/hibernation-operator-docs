# Changelog

## v0.1.104

_**September 16, 2026**_

### Breaking Changes

- `ResourceSupervisor` now records hibernated replica counts in `status.sleepingNamespaces` instead of the `hibernation.stakater.com/original-replicas` annotation on each workload, so GitOps controllers no longer report drift on hibernated workloads. Upgrading requires no action, as the first wake after upgrade restores anything still carrying the old annotation. Downgrading below this version while namespaces are asleep will leave workloads scaled to zero, so wait for a wake window or delete the `ResourceSupervisor` first.

### Bug Fixes

- Fixed StatefulSets never being woken, which also discarded the only record of their original replica count.
- Fixed a Deployment and a StatefulSet sharing a name colliding, so that only one of them was ever woken.
- Fixed `ResourceSupervisor` waking workloads it never put to sleep, and restoring unannotated workloads to a single replica.
- Fixed a failed wake being reported as success, so the affected workloads were never retried.
- Fixed deletion removing the finalizer even when the pre-delete wake failed, leaving namespaces asleep with nothing left to retry.
- Fixed supervisors that reference only ArgoCD applications being rejected by the validating webhook.
- Fixed an invalid sleep schedule being accepted when no wake schedule was set, after which the supervisor stopped reconciling.
- Fixed a failed namespace lookup waking every sleeping namespace.
- Fixed transient errors permanently freezing a supervisor until the operator restarted.
- Fixed workloads rescaled between sleep passes being woken at a stale replica count.

### Changes to behavior

- Logging now defaults to production mode (JSON, Info level) instead of debug console output. Pass `-zap-devel` for the previous behavior.
- The `pprof` endpoint no longer listens by default. Pass `--pprof-bind-address` to enable it.

### Enhancements

- `ResourceSupervisor` and `ClusterResourceSupervisor` now share one reconciliation path, so both behave identically.
- ArgoCD upgraded to `v3.3.12` and Go to 1.25, along with security updates.

## v0.1.103

_**October 2, 2025**_

Dependency and security updates.

## v0.1.102

_**September 3, 2025**_

Initial release.
