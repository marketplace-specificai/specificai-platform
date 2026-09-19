# SpecificAI Platform

Customer-facing documentation and Helm values for deploying the SpecificAI Platform.

This repository is **generated** from SpecificAI's private monorepo. Do not open pull requests against it with content edits — they will be overwritten on the next chart-version sync. Report issues or request changes through your SpecificAI contact.

## Documentation

Hosted docs: <https://marketplace-specificai.github.io/specificai-platform/>

| Page | Source |
| --- | --- |
| [AWS requirements](docs/requirements/aws.md) | Cloud prerequisites for EKS |
| [Azure requirements](docs/requirements/azure.md) | Cloud prerequisites for AKS |
| [Changelog](CHANGELOG.md) | Chart versions from 4.9.0 onward |

GCP requirements are not published yet.

## Helm values templates

Copy the file that matches your cluster type, replace the `<PLACEHOLDER>` values, and pass it to `helm upgrade --install`.

| File | Cluster type |
| --- | --- |
| [`values/aws-auto-mode.values.yaml`](values/aws-auto-mode.values.yaml) | Amazon EKS Auto Mode |
| [`values/aws-standard.values.yaml`](values/aws-standard.values.yaml) | Amazon EKS with self-managed Karpenter |
| [`values/azure-automatic.values.yaml`](values/azure-automatic.values.yaml) | Azure AKS Automatic |
| [`values/azure-standard.values.yaml`](values/azure-standard.values.yaml) | Azure AKS Standard |
| [`values/gcp-autopilot.values.yaml`](values/gcp-autopilot.values.yaml) | Google GKE Autopilot |
| [`values/gcp-standard.values.yaml`](values/gcp-standard.values.yaml) | Google GKE Standard |

Using the wrong values file for your cluster type can leave GPU workloads pending — each file's header comments name the exact mode it is for.
