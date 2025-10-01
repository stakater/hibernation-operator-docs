# Overview

The **Hibernation Operator** supports two installation methods: **Operator Lifecycle Manager (OLM)** and **Helm Chart**. These methods ensure flexibility and compatibility across various Kubernetes environments, including **OpenShift**, **Azure Kubernetes Service (AKS)**, **Amazon Elastic Kubernetes Service (EKS)**, **Google Kubernetes Engine (GKE)**, and other standard Kubernetes distributions.

## Installation Methods

### 1. Operator Lifecycle Manager (OLM)

OLM is the recommended installation method for **OpenShift** clusters. The Hibernation Operator is [Red Hat Certified](https://catalog.redhat.com/en/software/container-stacks/detail/687efb0f70dee945827cfe24) and available in the **OpenShift OperatorHub**. This method provides native lifecycle management, automatic upgrades (when configured), and seamless integration with OpenShift’s operator framework.

### 2. Helm Chart

Helm is the preferred installation method for **all other Kubernetes platforms**, including AKS, EKS, GKE, and vanilla Kubernetes. The Helm chart offers a lightweight, customizable, and GitOps-friendly deployment with support for optional features like **ArgoCD integration** and **admission webhooks**.

## Prerequisites

Before proceeding, ensure the following:

* Access to an **OpenShift cluster** (for OLM) or a **Kubernetes cluster v1.24+** (for Helm).
* **Cluster administrator** or equivalent permissions.
* Familiarity with `kubectl` (or `oc` for OpenShift).
* **Helm CLI installed locally** if using Helm-based installation.
* *(Optional)* **cert-manager** installed if enabling the admission webhook (required only for validation of cron schedules and CR structure).

## Next Steps

Choose the installation guide that matches your environment:

* [Installing with OLM on OpenShift](openshift.md)  
* [Installing with Helm on Kubernetes (generic)](kubernetes.md)

By following the appropriate guide, you’ll be able to deploy the Hibernation Operator and start automating cost-saving hibernation of your workloads—whether you're a platform team managing hundreds of namespaces or a developer optimizing a single staging environment.
