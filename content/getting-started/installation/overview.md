# Overview

The Hibernation Operator can be installed with the Operator Lifecycle Manager (OLM) or with Helm. Both methods deploy the same controller, admission webhook, CRDs and RBAC resources.

## Installation methods

### Operator Lifecycle Manager (OLM)

OLM is the recommended method on OpenShift. The Hibernation Operator is [Red Hat Certified](https://catalog.redhat.com/en/software/container-stacks/detail/687efb0f70dee945827cfe24) and available in the OpenShift OperatorHub, and OLM manages its upgrades and the certificate of its admission webhook.

### Helm

Helm is the recommended method on all other Kubernetes distributions, including AKS, EKS, GKE and upstream Kubernetes. The chart is published as an OCI artifact and suits GitOps workflows. It requires cert-manager to issue the webhook and metrics certificates.

## Prerequisites

* An OpenShift cluster for OLM, or a Kubernetes cluster at version 1.24 or later for Helm
* Cluster administrator permissions
* `kubectl`, or `oc` on OpenShift
* The Helm CLI, when installing with Helm
* cert-manager, when installing with Helm

## Next steps

* [On OpenShift](openshift.md): installation with OLM
* [On Kubernetes](kubernetes.md): installation with Helm
