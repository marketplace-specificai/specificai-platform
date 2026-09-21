# Azure AKS Automatic values

`azure-automatic.values.yaml` targets **Azure AKS Automatic** clusters. Download the raw file
from [`values/azure-automatic.values.yaml`](https://github.com/marketplace-specificai/specificai-platform/blob/main/values/azure-automatic.values.yaml),
fill in the `<PLACEHOLDER>` values it marks, and pass it to
`helm upgrade --install` as described in the [install guide](../install.md).

```yaml
# DO NOT EDIT in the public repository. This file is generated from
# SpecificAI's private chart sources by
# tools/marketplace_mirror/transform_content.py and republished on every
# chart release. Copy it, fill in the placeholder values it marks, and keep
# the copy under your own version control.

# ==============================================================================
# Cluster type: Azure AKS — Automatic (built-in node auto-provisioning)
# ==============================================================================
# WARNING: This file is for AKS Automatic ONLY. Helm has no way to detect which
# mode your cluster actually runs in. Applying this file to an AKS Standard
# cluster with pre-built node pools (or applying azure-standard.values.yaml
# here) can leave GPU pods pending indefinitely, because there is no
# controller in that cluster mode to satisfy the NodePool/AKSNodeClass this
# file relies on. Use azure-standard.values.yaml instead if your cluster has
# manually-provisioned GPU node pools.
# ==============================================================================
#
# AKS Automatic clusters run node auto-provisioning (NAP) out of the box, so
# the chart's own NodePool + AKSNodeClass resources are enough — no separate
# Karpenter install or IAM-style role is needed (unlike AWS self-managed
# Karpenter). NAP-provisioned GPU nodes come with the NVIDIA driver and device
# plugin preinstalled by the platform, so the chart's own device plugin
# DaemonSet must stay disabled. GPU tiers are selected by the provider-native
# `karpenter.azure.com/sku-name` label NAP applies to every node, rather than
# a pool label — there is no pre-built pool to label on this cluster mode.

global:
  cloudProvider: "azure"
  # Azure region your cluster runs in, e.g. southcentralus.
  cloudRegion: "<YOUR_AZURE_REGION>"
  # Storage account + container backing object storage, as "<account>/<container>".
  bucketName: "<YOUR_STORAGE_ACCOUNT_NAME>/<YOUR_CONTAINER_NAME>"
  # Workload Identity (UAMI) client ID the platform's ServiceAccount uses.
  roleId: "<YOUR_MANAGED_IDENTITY_CLIENT_ID>"
  # Hostname the platform will be served on.
  optuneAddress: "<YOUR_PLATFORM_HOSTNAME>" # e.g. specificai.yourcompany.com

  # CPU tiers key off the "workload" label the chart's NodePool template
  # applies. Leaving these unset on Azure reaches job-runner as an empty
  # selector and the Job is silently never created, so they must be set
  # explicitly.
  cpuHighPerformanceNodeSelector:
    workload: "cpu-basic"
  cpuBasicNodeSelector:
    workload: "cpu-basic"
  cpuLowPerformanceNodeSelector:
    workload: "cpu-basic"
  # GPU tiers key off the native SKU-name label NAP applies to every node it
  # provisions, so you can pin exactly which VM SKU serves each tier without
  # depending on a pool label. Must match (or be a subset of)
  # common.karpenter.azure.gpu*SkuNames below.
  gpuHighPerformanceNodeSelector:
    karpenter.azure.com/sku-name: "<YOUR_HIGH_PERF_GPU_SKU>" # e.g. Standard_NC24ads_A100_v4
  gpuBasicNodeSelector:
    karpenter.azure.com/sku-name: "<YOUR_BASIC_GPU_SKU>" # e.g. Standard_NV36ads_A10_v5
  gpuLowPerformanceNodeSelector:
    karpenter.azure.com/sku-name: "<YOUR_LOW_PERF_GPU_SKU>" # e.g. Standard_NC8as_T4_v3

# Workload Identity is the default Blob auth on Azure (global.azure.useWorkloadIdentityForBlob),
# so both the backend models volume and Triton need the models container
# mounted directly rather than reading via an account key. Required on Azure
# regardless of cluster mode.
backend:
  storage:
    blob:
      containerName: "<YOUR_CONTAINER_NAME>"
      accountName: "<YOUR_STORAGE_ACCOUNT_NAME>"
      # Resource group of the storage account (NOT the AKS MC_* group).
      resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"

specificai-inference:
  models:
    blob:
      enabled: true
      accountName: "<YOUR_STORAGE_ACCOUNT_NAME>"
      containerName: "<YOUR_CONTAINER_NAME>"
      resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"

common:
  karpenter:
    # Render the chart's NodePool + AKSNodeClass (see
    # charts/common/templates/nodepool.yaml + nodeclasses.yaml). AKS
    # Automatic's built-in NAP controller reconciles them directly — no
    # roleArn/discoveryTag needed (those are AWS-only concepts).
    createNodePools: true
    # SKUs NAP is allowed to provision per GPU tier — keep in sync with the
    # sku-name node selectors above. Empty lists fall back to the chart's
    # defaults (L40S for high, several A10 variants for basic, T4 for low).
    azure:
      gpuHighSkuNames:
        - "<YOUR_HIGH_PERF_GPU_SKU>"
      gpuBasicSkuNames:
        - "<YOUR_BASIC_GPU_SKU>"
      gpuLowSkuNames:
        - "<YOUR_LOW_PERF_GPU_SKU>"

  # NAP-provisioned GPU nodes come with the NVIDIA driver and device plugin
  # preinstalled by the platform — leave the chart's own DaemonSet disabled to
  # avoid double-managing the same device.
  nvidiaDevicePlugin:
    enabled: false
```
