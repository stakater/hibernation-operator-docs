<!-- markdownlint-disable -->
# API Reference

## Packages
- [hibernation.stakater.com/v1beta1](#hibernationstakatercomv1beta1)


## hibernation.stakater.com/v1beta1


### Resource Types
- [ClusterResourceSupervisor](#clusterresourcesupervisor)
- [ResourceSupervisor](#resourcesupervisor)



#### ClusterResourceSupervisor



ClusterResourceSupervisor is the Schema for the resourcesupervisors API





| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `hibernation.stakater.com/v1beta1` | | |
| `kind` _string_ | `ClusterResourceSupervisor` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#objectmeta-v1-meta)_ | Refer to Kubernetes API documentation for fields of `metadata`. |  |  |
| `spec` _[ClusterResourceSupervisorSpec](#clusterresourcesupervisorspec)_ |  |  |  |
| `status` _[ClusterResourceSupervisorStatus](#clusterresourcesupervisorstatus)_ |  |  |  |




#### ClusterResourceSupervisorSpec



ClusterResourceSupervisorSpec defines the desired state of ClusterResourceSupervisor



_Appears in:_
- [ClusterResourceSupervisor](#clusterresourcesupervisor)

| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `schedule` _Hibernation_ |  |  | Required: \{\} <br /> |
| `namespaces` _Namespaces_ | Namespaces is a list of namespaces to which the schedule will be applied |  | Optional: \{\} <br /> |
| `argocd` _ArgoCDHibernation_ | ArgoCD contains details about ArgoCD to which the schedule will be applied |  |  |


#### ClusterResourceSupervisorStatus



ClusterResourceSupervisorStatus defines the observed state of ClusterResourceSupervisor



_Appears in:_
- [ClusterResourceSupervisor](#clusterresourcesupervisor)

| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `nextReconcileTime` _[Time](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#time-v1-meta)_ | NextReconcileTime contains the next time at which the namespace resources will sleep or wake up |  |  |
| `currentStatus` _[Status](#status)_ | CurrentStatus shows the state the tenant's resources |  | Enum: [sleeping running error] <br /> |
| `sleepingNamespaces` _SleepingNamespace array_ | SleepingResources contains the previous states for each of the deployments currently scaled down |  |  |
| `watchedNamespaces` _string array_ | WatchedNamespaces contains the list of namespaces that are being watched by the ClusterResourceSupervisor |  |  |
| `ignoreNamespaces` _string array_ | IgnoreNamespaces contains the list of namespaces that are being ignored by the ClusterResourceSupervisor |  |  |


#### ResourceSupervisor



ResourceSupervisor is the Schema for the resourcesupervisors API





| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `hibernation.stakater.com/v1beta1` | | |
| `kind` _string_ | `ResourceSupervisor` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#objectmeta-v1-meta)_ | Refer to Kubernetes API documentation for fields of `metadata`. |  |  |
| `spec` _[ResourceSupervisorSpec](#resourcesupervisorspec)_ |  |  |  |
| `status` _[ResourceSupervisorStatus](#resourcesupervisorstatus)_ |  |  |  |




#### ResourceSupervisorSpec



ResourceSupervisorSpec defines the desired state of ResourceSupervisor API



_Appears in:_
- [ResourceSupervisor](#resourcesupervisor)

| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `schedule` _Hibernation_ |  |  | Required: \{\} <br /> |


#### ResourceSupervisorStatus



ResourceSupervisorStatus defines the observed state of ResourceSupervisor



_Appears in:_
- [ResourceSupervisor](#resourcesupervisor)

| Field | Description | Default | Validation |
| --- | --- | --- | --- |
| `nextReconcileTime` _[Time](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#time-v1-meta)_ | NextReconcileTime contains the next time at which the namespace resources will sleep or wake up |  |  |
| `currentStatus` _[Status](#status)_ | CurrentStatus shows the state the tenant's resources |  | Enum: [sleeping running error] <br /> |


#### Status

_Underlying type:_ _string_



_Validation:_
- Enum: [sleeping running error]

_Appears in:_
- [ClusterResourceSupervisorStatus](#clusterresourcesupervisorstatus)
- [ResourceSupervisorStatus](#resourcesupervisorstatus)



