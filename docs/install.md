# SpecificAI Platform — Install guide

This page takes you from provisioned infrastructure to a running platform:
log in to the chart registry, install the Helm chart with the values file
that matches your cluster, and verify the deployment.

Before you start, you need:

- The infrastructure described on your cloud's requirements page —
  [AWS](requirements/aws.md), [Azure](requirements/azure.md),
  or [GCP](requirements/gcp.md).
- Pull access to the chart registry, described on the
  [registry access page](registry-access.md).
- Helm 3.8 or later (the chart is distributed as an OCI artifact), `kubectl`
  pointed at your cluster, and the AWS CLI for the registry login.

## Choose your values file

The chart ships a starter values file per supported cluster type. Pick the
one that matches your cluster, fill in the `<PLACEHOLDER>` values it marks,
and keep your copy under your own version control.

| Values file | Cluster type |
|---|---|
| `aws-auto-mode.values.yaml` | Amazon EKS Auto Mode |
| `aws-standard.values.yaml` | Amazon EKS with self-managed Karpenter |
| `azure-automatic.values.yaml` | Azure AKS Automatic |
| `azure-standard.values.yaml` | Azure AKS Standard |
| `gcp-autopilot.values.yaml` | Google GKE Autopilot |
| `gcp-standard.values.yaml` | Google GKE Standard |

The files live in the
[`values/` directory](https://github.com/marketplace-specificai/specificai-platform/tree/main/values)
of the public platform repository.

!!! warning "The chart cannot detect which mode your cluster runs in"

    Applying the wrong file for your cluster mode leaves GPU pods pending
    indefinitely. Your cloud's requirements page explains how the two modes
    differ; confirm the mode before you choose.

Two defaults worth knowing before you install:

- **Message broker.** The chart runs RabbitMQ inside the cluster by default,
  installed and managed by the chart itself — you provision no managed broker
  unless you deliberately switch to Kafka, as described on the requirements
  pages.
- **Secrets.** The chart creates the platform's Kubernetes Secrets from
  values you pass at install time. Have the secret material listed under
  **Secrets** on your requirements page ready — or pre-create the Secrets
  yourself and set `global.createSecret` to `false`.

## Log in to the registry

Authenticate Helm against the AWS Marketplace registry with the access
arranged on the [registry access page](registry-access.md):

```bash
aws ecr get-login-password --region us-east-1 \
  | helm registry login 709825985650.dkr.ecr.us-east-1.amazonaws.com \
      --username AWS --password-stdin
```

Azure and GCP customers export the issued key as `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` first; AWS customers use their own credentials.

## Install the chart

Install straight from the registry, passing your filled-in values file. Any
namespace works; the examples use `specificai`.

```bash
helm upgrade --install specificai \
  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform \
  --version <CHART_VERSION> \
  --namespace specificai --create-namespace \
  --values <your-values-file>.values.yaml
```

The chart version determines the platform version; the
[changelog](https://github.com/marketplace-specificai/specificai-platform/blob/main/CHANGELOG.md)
lists what each release contains.

To inspect the chart before installing, pull it first:

```bash
helm pull \
  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform \
  --version <CHART_VERSION>
```

## Verify the installation

Check the release and its workloads:

```bash
helm status specificai --namespace specificai
kubectl get pods --namespace specificai
```

Within a few minutes every pod should be `Running` or `Completed`. GPU nodes
are provisioned on demand, so seeing no GPU nodes while the platform is idle
is normal, not a failure.

Then check the entry point:

```bash
kubectl get ingress --namespace specificai
```

Once the ingress has an address, point the DNS record for your platform
hostname (the value you set as `global.optuneAddress`) at it as described in
the **Networking** section of your requirements page. Open
`https://<your platform hostname>` and sign in with the identity provider you
configured.

If a training or inference pod stays `Pending` and no GPU node appears, the
values file almost always does not match the cluster mode — recheck the
mapping table on your requirements page and in **Choose your values file**
above.

## Upgrade and uninstall

Upgrades are the same `helm upgrade --install` command with a newer
`--version`; the [upgrade guide](upgrade.md) covers the full procedure,
including the per-version pre-upgrade checklist and rollback. Review the
[changelog](https://github.com/marketplace-specificai/specificai-platform/blob/main/CHANGELOG.md)
first.

To remove the platform, run `helm uninstall specificai --namespace
specificai`. The data in your object storage and database is customer-owned
infrastructure and is not touched.
