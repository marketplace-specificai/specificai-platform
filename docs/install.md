# SpecificAI Platform — Install guide

This page lists every command you run to go from provisioned infrastructure
to a running platform, in order: pull the chart, fill in your values, add
your image pull credential, install, verify, and later upgrade.

## Before you start

You need:

- The infrastructure described on your cloud's requirements page —
  [AWS](requirements/aws.md), [Azure](requirements/azure.md),
  or [GCP](requirements/gcp.md).
- Helm 3.8 or later and `kubectl` pointed at your cluster.
- The **Docker username and read-only token** SpecificAI issued you. It is
  the only credential you need — see the [access page](registry-access.md).

The Helm chart itself is public on GitHub Container Registry
(`ghcr.io/marketplace-specificai/specificai-platform`) and needs no login.

## 1. Pull the chart

```bash
helm pull oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version 4.10.2
```

Optionally, render the chart locally to inspect it before installing. This
touches no cluster:

```bash
helm template specificai \
  specificai-platform-4.10.2.tgz \
  --set global.cloudProvider=gcp \
  --set global.bucketName=example-bucket \
  --set global.optuneAddress=platform.example.com \
  > /dev/null
```

## 2. Download and fill in your values file

Pick the values file that matches your cluster type:

| Values file | Cluster type |
|---|---|
| `aws-auto-mode.values.yaml` | Amazon EKS Auto Mode |
| `aws-standard.values.yaml` | Amazon EKS with self-managed Karpenter |
| `azure-automatic.values.yaml` | Azure AKS Automatic |
| `azure-standard.values.yaml` | Azure AKS Standard |
| `gcp-autopilot.values.yaml` | Google GKE Autopilot |
| `gcp-standard.values.yaml` | Google GKE Standard |

!!! warning "The chart cannot detect which mode your cluster runs in"

    Applying the wrong file for your cluster mode leaves GPU pods pending
    indefinitely. Your cloud's requirements page explains how the two modes
    differ; confirm the mode before you choose.

Download it — replace the file name with yours:

```bash
VALUES_FILE=gcp-standard.values.yaml
curl -fsSLO "https://raw.githubusercontent.com/marketplace-specificai/specificai-platform/main/values/${VALUES_FILE}"
```

Open the file and replace every `<PLACEHOLDER>` value (region, bucket,
identity, platform hostname). Keep your copy under your own version control.

## 3. Add your Docker token

The platform's container images live in SpecificAI's private Docker registry.
Turn the username and token SpecificAI issued you into the image pull
credential the chart expects, and write it to a separate secrets file:

```bash
DOCKER_CONFIG_JSON=$(kubectl create secret docker-registry docker-ro-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<DOCKER_USERNAME> \
  --docker-password=<DOCKER_TOKEN> \
  --dry-run=client -o jsonpath='{.data.\.dockerconfigjson}')

cat > secrets.values.yaml <<EOF
common:
  secrets:
    dockerConfigJson: "${DOCKER_CONFIG_JSON}"
EOF
```

`--dry-run=client` only encodes the credential locally; nothing is sent to
your cluster yet. Add the other secret values listed under **Secrets** on your
requirements page (database connection string, broker password, SSO) to the
same file. Keep `secrets.values.yaml` out of version control.

## 4. Install

Any namespace works; the examples use `specificai`.

```bash
helm upgrade --install specificai \
  oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version 4.10.2 \
  --namespace specificai --create-namespace \
  --values "${VALUES_FILE}" \
  --values secrets.values.yaml
```

The chart runs RabbitMQ inside the cluster by default, so there is no managed
message broker to provision unless you deliberately switch to Kafka.

## 5. Verify

```bash
helm status specificai --namespace specificai
kubectl get pods --namespace specificai
kubectl get ingress --namespace specificai
```

Within a few minutes every pod should be `Running` or `Completed`. GPU nodes
are provisioned on demand, so seeing no GPU nodes while the platform is idle
is normal.

Once the ingress has an address, point the DNS record for your platform
hostname (`global.optuneAddress`) at it, as described in the **Networking**
section of your requirements page, then open `https://<your platform hostname>`.

If a pod reports `ImagePullBackOff`, the Docker token is missing or wrong —
recheck step 3. If a training or inference pod stays `Pending` and no GPU node
appears, the values file does not match the cluster mode — recheck step 2.

## 6. Upgrade

Review the
[changelog](https://github.com/marketplace-specificai/specificai-platform/blob/main/CHANGELOG.md)
and the [upgrade guide](upgrade.md) first. Then run the same command with the
new version and the same two values files:

```bash
helm upgrade --install specificai \
  oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version <NEW_CHART_VERSION> \
  --namespace specificai \
  --values "${VALUES_FILE}" \
  --values secrets.values.yaml
```

## Optional: verify the chart signature

Every chart version is signed with [cosign](https://docs.sigstore.dev/). The
public key is
[`cosign.pub`](https://github.com/marketplace-specificai/specificai-platform/blob/main/cosign.pub)
at the root of the public repository:

```bash
cosign verify --key cosign.pub \
  ghcr.io/marketplace-specificai/specificai-platform:4.10.2
```

A failure means the artifact is not one SpecificAI released; do not install
it and contact `devops@specific.ai`.

## Uninstall

```bash
helm uninstall specificai --namespace specificai
```

The data in your object storage and database is customer-owned infrastructure
and is not touched.
