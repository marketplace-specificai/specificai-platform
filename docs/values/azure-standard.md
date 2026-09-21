# Azure AKS Standard values

`azure-standard.values.yaml` targets **Azure AKS Standard** clusters. Download the raw file
from [`values/azure-standard.values.yaml`](https://github.com/marketplace-specificai/specificai-platform/blob/main/values/azure-standard.values.yaml),
fill in the `<PLACEHOLDER>` values it marks, and pass it to
`helm upgrade --install` as described in the [install guide](../install.md).

```yaml
# DO NOT EDIT in the public repository. This file is generated from
# SpecificAI's private chart sources by
# tools/marketplace_mirror/transform_content.py and republished on every
# chart release. Copy it, fill in the placeholder values it marks, and keep
# the copy under your own version control.

# ==============================================================================
# Cluster type: Azure AKS — Standard (pre-built node pools, no Karpenter/NAP)
# ==============================================================================
# WARNING: This file is for AKS Standard clusters with manually-provisioned
# node pools ONLY. Helm has no way to detect which mode your cluster actually
# runs in. Applying azure-automatic.values.yaml to a cluster like this leaves
# GPU pods pending indefinitely, because there is no node auto-provisioning
# controller here to satisfy the NodePool/AKSNodeClass that file renders. Use
# azure-automatic.values.yaml instead if your cluster runs AKS Automatic.
# ==============================================================================
#
# AKS Standard has no built-in node auto-provisioning and this file does not
# install Karpenter for you, so GPU (and CPU) node pools must already exist —
# create them ahead of time (Terraform, az aks nodepool add, or the portal)
# with the labels referenced below. Because there is no managed provisioning
# layer installing GPU drivers on your behalf, the chart's own NVIDIA device
# plugin DaemonSet must be enabled.

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

  # Node selectors matching the labels on YOUR pre-built node pools (set via
  # `az aks nodepool add --labels workload=<tier>` or your Terraform module).
  # There is no chart-rendered NodePool to stamp a "workload" label here, so
  # these must match whatever your pools actually carry — the AKS-native
  # agent-pool name is shown below as an alternative when you did not apply a
  # custom "workload" label at pool-creation time. Leaving these unset on
  # Azure reaches job-runner as an empty selector and the Job is silently
  # never created, so they must be set explicitly.
  cpuHighPerformanceNodeSelector:
    workload: "cpu-basic"
  cpuBasicNodeSelector:
    workload: "cpu-basic"
  cpuLowPerformanceNodeSelector:
    workload: "cpu-basic"
  gpuHighPerformanceNodeSelector:
    workload: "gpu-high-performance"
  gpuBasicNodeSelector:
    workload: "gpu-basic"
  gpuLowPerformanceNodeSelector:
    workload: "gpu-low-performance"
  # Alternative when you did not apply a custom "workload" label: select by
  # the AKS-assigned agent-pool name directly.
  # gpuHighPerformanceNodeSelector:
  #   kubernetes.azure.com/agentpool: "<YOUR_GPU_HIGH_PERF_NODE_POOL_NAME>"

  # Pre-built pools have no chart-rendered taint to tolerate. Add tolerations
  # here ONLY if you tainted your pools yourself, matching that taint.
  # gpuHighPerformanceTolerations:
  #   - key: "<your-taint-key>"
  #     operator: "Equal"
  #     value: "<your-taint-value>"
  #     effect: "NoSchedule"

# S3-style storage is not used on Azure — Blob container settings live under
# backend.storage.blob instead.
backend:
  storage:
    blob:
      containerName: "<YOUR_CONTAINER_NAME>"
      accountName: "<YOUR_STORAGE_ACCOUNT_NAME>"
      # Resource group of the storage account (NOT the AKS MC_* group).
      resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"

# Workload Identity is the default Blob auth on Azure (global.azure.useWorkloadIdentityForBlob),
# so Triton needs the models container mounted directly rather than reading
# via an account key. Required on Azure regardless of cluster mode.
specificai-inference:
  models:
    blob:
      enabled: true
      accountName: "<YOUR_STORAGE_ACCOUNT_NAME>"
      containerName: "<YOUR_CONTAINER_NAME>"
      resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"

common:
  karpenter:
    # No Karpenter/NAP on this cluster mode — do not render NodePool/AKSNodeClass.
    createNodePools: false

  # No node auto-provisioning layer here to preinstall GPU drivers on your
  # behalf, so the chart must deploy its own device plugin DaemonSet.
  nvidiaDevicePlugin:
    enabled: true
```
