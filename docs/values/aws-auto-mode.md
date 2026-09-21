# Amazon EKS Auto Mode values

`aws-auto-mode.values.yaml` targets **Amazon EKS Auto Mode** clusters. Download the raw file
from [`values/aws-auto-mode.values.yaml`](https://github.com/marketplace-specificai/specificai-platform/blob/main/values/aws-auto-mode.values.yaml),
fill in the `<PLACEHOLDER>` values it marks, and pass it to
`helm upgrade --install` as described in the [install guide](../install.md).

```yaml
# DO NOT EDIT in the public repository. This file is generated from
# SpecificAI's private chart sources by
# tools/marketplace_mirror/transform_content.py and republished on every
# chart release. Copy it, fill in the placeholder values it marks, and keep
# the copy under your own version control.

# ==============================================================================
# Cluster type: AWS EKS — Auto Mode
# ==============================================================================
# WARNING: This file is for EKS Auto Mode ONLY. Helm has no way to detect which
# mode your cluster actually runs in. Applying this file to a self-managed-
# Karpenter EKS cluster (or applying aws-standard.values.yaml to an Auto Mode
# cluster) can leave GPU pods pending indefinitely, because the NodeClass your
# cluster needs to provision GPU nodes will not exist. Use
# aws-standard.values.yaml instead if you run your own Karpenter controller.
# ==============================================================================
#
# EKS Auto Mode ships a built-in Karpenter-compatible controller and a default
# NodeClass, so the chart does not need to render an EC2NodeClass (or the IAM
# role / discovery tags that come with it) — it only renders NodePools that
# point at Auto Mode's own NodeClass. Auto Mode also installs the NVIDIA GPU
# driver and device plugin on GPU nodes itself, so the chart's own device
# plugin DaemonSet must stay disabled.

global:
  cloudProvider: "aws"
  # AWS region your cluster runs in, e.g. us-west-2.
  cloudRegion: "<YOUR_AWS_REGION>"
  # S3 bucket used for object storage (datasets, checkpoints, artifacts).
  bucketName: "<YOUR_S3_BUCKET_NAME>"
  # IRSA role ARN the platform's ServiceAccount assumes for S3 / other AWS APIs.
  roleId: "<YOUR_IRSA_ROLE_ARN>" # arn:aws:iam::<account-id>:role/<your-irsa-role>
  # Hostname the platform will be served on.
  optuneAddress: "<YOUR_PLATFORM_HOSTNAME>" # e.g. specificai.yourcompany.com

  # Node selectors the chart's own NodePools (rendered below) label their
  # nodes with, regardless of Auto Mode vs self-managed Karpenter — job-runner
  # stamps these onto training/evaluation/image-build Jobs. Leaving these unset
  # on AWS reaches job-runner as an empty selector and the Job is silently
  # never created, so they must be set explicitly (no chart-side default here).
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

  # Matching tolerations for the taints the chart's NodePools apply.
  cpuHighPerformanceTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "cpu-basic"
      effect: "NoSchedule"
  cpuBasicTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "cpu-basic"
      effect: "NoSchedule"
  cpuLowPerformanceTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "cpu-basic"
      effect: "NoSchedule"
  gpuHighPerformanceTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "gpu-high-performance"
      effect: "NoSchedule"
  gpuBasicTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "gpu-basic"
      effect: "NoSchedule"
  gpuLowPerformanceTolerations:
    - key: "instance-type"
      operator: "Equal"
      value: "gpu-low-performance"
      effect: "NoSchedule"

# S3 model storage backend (used by the backend service's model artifacts).
# Must be nested under `storage:` — the backend subchart reads
# .Values.storage.s3.*, so a bare `backend.s3.*` is silently ignored.
backend:
  storage:
    s3:
      region: "<YOUR_AWS_REGION>"
      bucketName: "<YOUR_S3_BUCKET_NAME>"

common:
  karpenter:
    # Render the chart's NodePools (see charts/common/templates/nodepool.yaml).
    createNodePools: true
    # Auto Mode: skip EC2NodeClass and point NodePools at Auto Mode's built-in
    # default NodeClass. Do NOT set karpenter.roleArn / karpenter.discoveryTag
    # in this mode — they are only read on the self-managed path.
    mode: "auto-mode"

  # Auto Mode installs the NVIDIA driver and device plugin on GPU nodes
  # itself — leave the chart's own DaemonSet disabled to avoid double-managing
  # the same device.
  nvidiaDevicePlugin:
    enabled: false

# Triton reads deployed models straight from object storage. The umbrella chart
# blanks this value, so it has to be set per cloud: without it the model
# repository URI renders with no scheme ("://<bucket>/deployed_models") and
# Triton fails to start.
specificai-inference:
  storageProvider: "s3"
```
