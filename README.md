<!--
  DO NOT EDIT in the public repository. This file is generated from
  .devops/docs/customer-facing/public/README.md in SpecificAI's private
  sources by tools/marketplace_mirror/transform_content.py and republished
  on every chart release, so a hand edit here is overwritten by the next
  sync.

  The one hand-managed file in the public repository is
  .github/workflows/pages.yml. Pushing a workflow file requires the sync
  App's `workflows` permission, which the mirror App is not granted — it is
  scoped to repository contents only — so the pipeline deliberately never
  writes under .github/workflows/. Change that file by hand in the public
  repository, and only that file.
-->

# SpecificAI Platform

<!--
  Static shields.io badges only — no external services beyond img.shields.io.
  The 4.9.0 token is stamped with the released chart
  version by transform_content.py at staging time.
-->
[![Chart version](https://img.shields.io/badge/chart-4.9.0-1753ff)](https://marketplace-specificai.github.io/specificai-platform/changelog/)
[![License](https://img.shields.io/badge/license-Apache--2.0%20%2B%20CC%20BY%204.0-252a5c)](LICENSE)
[![Documentation](https://img.shields.io/badge/docs-github.io-1753ff)](https://marketplace-specificai.github.io/specificai-platform/)

Customer-facing documentation and Helm values for deploying the SpecificAI
Platform on your own cloud account.

This repository is **generated**. A sync bot republishes every file here from
SpecificAI's private sources on each chart release, which is why pull requests
are not accepted: any change merged here would be overwritten by the next
sync. Report issues or request changes through your SpecificAI contact or
`devops@specific.ai`.

## Documentation

Hosted docs: <https://marketplace-specificai.github.io/specificai-platform/>

| Page | Covers |
|---|---|
| [AWS requirements](docs/requirements/aws.md) | Cloud prerequisites for EKS |
| [Azure requirements](docs/requirements/azure.md) | Cloud prerequisites for AKS |
| [GCP requirements](docs/requirements/gcp.md) | Cloud prerequisites for GKE |
| [Registry access](docs/registry-access.md) | How chart and image pull access is granted, per cloud |
| [Install guide](docs/install.md) | Registry login, `helm install`, verification |
| [Upgrade guide](docs/upgrade.md) | Upgrade procedure, per-version pre-upgrade checklist, rollback |
| [Feature availability](docs/features.md) | Features by version and cloud (generated per release) |
| [Support](docs/support.md) | How to reach the SpecificAI team |
| [Changelog](CHANGELOG.md) | Chart versions from 4.0.0 onward |

## Helm chart

The chart and its container images are distributed through the AWS
Marketplace container registry as
`709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform`.
[Registry access](docs/registry-access.md) describes how access is granted.

## Helm values templates

Copy the file that matches your cluster type, replace the `<PLACEHOLDER>`
values, and pass it to `helm upgrade --install` as described in the
[install guide](docs/install.md).

| File | Cluster type |
|---|---|
| [`values/aws-auto-mode.values.yaml`](values/aws-auto-mode.values.yaml) | Amazon EKS Auto Mode |
| [`values/aws-standard.values.yaml`](values/aws-standard.values.yaml) | Amazon EKS with self-managed Karpenter |
| [`values/azure-automatic.values.yaml`](values/azure-automatic.values.yaml) | Azure AKS Automatic |
| [`values/azure-standard.values.yaml`](values/azure-standard.values.yaml) | Azure AKS Standard |
| [`values/gcp-autopilot.values.yaml`](values/gcp-autopilot.values.yaml) | Google GKE Autopilot |
| [`values/gcp-standard.values.yaml`](values/gcp-standard.values.yaml) | Google GKE Standard |

Using the wrong values file for your cluster type can leave GPU workloads
pending — each file's header comments name the exact mode it is for.

## Security

Report suspected vulnerabilities privately to `devops@specific.ai` — see
[SECURITY.md](SECURITY.md).

## License

The Helm values templates under `values/` are licensed under
[Apache-2.0](LICENSE). The documentation under `docs/` is licensed under
[CC BY 4.0](LICENSE-docs).
