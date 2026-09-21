# Helm values templates

The chart ships one starter values file per supported cluster type. Pick
the file that matches your cluster, fill in the `<PLACEHOLDER>` values it
marks, and pass it to `helm upgrade --install` as described in the
[install guide](../install.md).

| Values file | Cluster type |
|---|---|
| [`aws-auto-mode.values.yaml`](aws-auto-mode.md) | Amazon EKS Auto Mode |
| [`aws-standard.values.yaml`](aws-standard.md) | Amazon EKS with self-managed Karpenter |
| [`azure-automatic.values.yaml`](azure-automatic.md) | Azure AKS Automatic |
| [`azure-standard.values.yaml`](azure-standard.md) | Azure AKS Standard |
| [`gcp-autopilot.values.yaml`](gcp-autopilot.md) | Google GKE Autopilot |
| [`gcp-standard.values.yaml`](gcp-standard.md) | Google GKE Standard |

The chart cannot detect which mode your cluster runs in, and applying the
wrong file can leave GPU pods pending indefinitely — each file's header
comments name the exact mode it is for. The raw files live in the
[`values/` directory](https://github.com/marketplace-specificai/specificai-platform/tree/main/values) of this
repository.
