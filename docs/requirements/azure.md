# SpecificAI Platform — Azure requirements

SpecificAI Platform is a self-hosted, Kubernetes-native application for
creating task-specific models. It runs entirely inside your own Azure
subscription: your datasets, your trained models, and your inference traffic
never leave it.

This page lists the Azure infrastructure to provision before installing the
platform Helm chart — the cluster, the node pools, the networking, and the
managed services the platform connects to. Provision what is described here,
then install the chart with the values file that matches your cluster.

Everything on this page is customer-owned. SpecificAI supplies the container
images, the Helm chart, and updates; you operate the infrastructure they run
on.

## Platform compute floor

The platform core must have this much capacity schedulable at all times.

| Resource | Floor |
|---|---|
| vCPU | 26 |
| Memory | 50 GB |

The floor covers the always-on platform services only. It excludes GPU
capacity and anything running outside the cluster. Training, evaluation, and
Playground inference run on GPU nodes that are provisioned on demand and
released when the work finishes, so GPU capacity is not part of the standing
footprint and is not billed while the platform is idle.

## Managed Kubernetes cluster

The platform is deployed onto an Azure Kubernetes Service cluster in your
subscription, new or existing. Both AKS modes are supported, and each has its
own values file.

| Cluster mode | Values file | Node provisioning |
|---|---|---|
| AKS Automatic | `azure-automatic.values.yaml` | Built-in node auto-provisioning reconciles the `NodePool` and `AKSNodeClass` the chart renders. |
| AKS Standard | `azure-standard.values.yaml` | Node pools you create ahead of time, labeled `workload: <tier>`. |

!!! warning "The chart cannot detect which mode your cluster runs in"

    Applying the wrong file leaves GPU pods pending indefinitely, because the
    node auto-provisioning controller the rendered `NodePool` needs — or the
    pre-built pool the node selectors point at — will not exist. Confirm the
    cluster mode before you choose.

On the standard path the chart also installs the NVIDIA device plugin
DaemonSet, because pre-built pools have no provisioning layer installing GPU
drivers on your behalf. On AKS Automatic, auto-provisioned GPU nodes come with
the driver and device plugin preinstalled, so the chart's own DaemonSet stays
disabled there. Each values file already sets this correctly.

For the platform core, provision a dedicated node pool of `Standard_D16ds_v5`
with a minimum of two nodes and a 100 GB OS disk. That is 32 vCPU and 128 GiB
against a floor of 26 vCPU and 50 GB, which leaves room for rolling upgrades
and for the burst of short-lived pods the platform creates while a job starts.
A dedicated pool also keeps the platform isolated from your other workloads.
Use on-demand capacity; spot is workable only if an eviction during working
hours is acceptable to you. GPU capacity is separate and is covered below.

The cluster also needs Workload Identity and the OIDC issuer enabled — object
storage authentication depends on both, as described under **Cloud services**.

## Node pools

The chart schedules work onto six named tiers. Each tier is a `workload` label
value; the chart's node selectors and matching tolerations are already set in
the values files, so you do not assign these by hand.

| Node pool | Instance type | Accelerator | Purpose |
|---|---|---|---|
| `gpu-high-performance` | Any L40S SKU | NVIDIA L40S | Generative and high-VRAM training. |
| `gpu-basic` | `Standard_NV6ads_A10_v5` up to `Standard_NV36ads_A10_v5` | NVIDIA A10 | Playground inference and general GPU training. |
| `gpu-low-performance` | T4 SKUs, e.g. `Standard_NC8as_T4_v3` | NVIDIA T4 | Classification and NER training and evaluation. |
| `cpu-high-performance` | `D` family | — | Shares the `cpu-basic` pool. |
| `cpu-basic` | `D` family | — | CPU training, data processing, and evaluation. |
| `cpu-low-performance` | `D` family | — | Shares the `cpu-basic` pool. |

The three CPU tiers deliberately map to one pool on Azure; the platform
distinguishes them so that other clouds can separate them, and so you can
split them later by overriding the tier's node selector.

On AKS Automatic, node auto-provisioning launches these nodes on demand as
jobs are queued and consolidates them away five minutes after they go idle, so
no GPU node runs continuously just because the platform is online. The values
file pins which VM SKUs each GPU tier may use through the
`karpenter.azure.com/sku-name` node selectors and the matching
`common.karpenter.azure` SKU lists; adjust both together to match the quota in
your subscription.

On AKS Standard, create the pools ahead of time — `az aks nodepool add
--labels workload=<tier>`, Terraform, or the portal — and keep the labels
aligned with the node selectors in the values file. Pools you did not label
can be selected by their AKS agent-pool name instead; the values file shows
the alternative.

## Networking

The chart exposes the platform through a Kubernetes Ingress. You provide the
ingress controller; the chart does not install one.

| Helm value | Meaning |
|---|---|
| `global.optuneAddress` | The hostname the platform is served on. Used as the Ingress host and as the base of the single sign-on redirect URI. |
| `global.ingress.createIngress` | Whether the chart renders the Ingress. Default `true`. |
| `global.ingress.className` | The `ingressClassName` on the rendered Ingress. Default `nginx`. |

Create a DNS record for `global.optuneAddress` pointing at your ingress
controller's public IP. Reserve a static public IP for the controller so the
record does not go stale when the load balancer is recreated.

**TLS is terminated at the Ingress on Azure.** Either reference an existing
Kubernetes TLS Secret with `global.ingress.certificateSecretName`, or let the
chart create it: set `global.ingress.createCertificateSecret` to `true` and
pass your PEM certificate and key as `global.ingress.certificateTLSCrt` and
`global.ingress.certificateTLSKey`.

The cluster needs outbound connectivity to the storage account, to the
database, to the registry the platform images are pulled from, and to the
small set of third-party endpoints listed under **Outbound endpoints** below.
Gateway API is not used on Azure; leave `global.gateway.enabled` at `false`.

## Cloud services

### Object storage

One dedicated blob container in a dedicated storage account holds datasets,
checkpoints, and trained model artifacts.

| Helm value | Meaning |
|---|---|
| `global.bucketName` | The storage account and container as `<account>/<container>`. |
| `backend.storage.blob.accountName` | Storage account name. |
| `backend.storage.blob.containerName` | Blob container name. |
| `backend.storage.blob.resourceGroup` | Resource group of the storage account — not the AKS `MC_*` node resource group. |
| `global.roleId` | Client ID of the user-assigned managed identity the platform's service account federates with. |
| `specificai-inference.serviceAccount.roleId` | Client ID of a dedicated read-only identity for the Triton inference service account. Leave unset to reuse `global.roleId`. |

The container must contain a `deployed_models/` prefix. The Triton inference
server uses it as its model repository path and will not start without it; the
values file already points the inference mount
(`specificai-inference.models.blob`) at the same container.

Enable CORS on the storage account's blob service, allowing `PUT` from
`https://<your platform hostname>` with the `ETag` and `Content-Type` headers
exposed. The platform uploads and downloads large files directly from the
browser using pre-signed URLs, which fail without it. Add a lifecycle
management rule that deletes blobs under the `downloads/` prefix after one
day, so temporary download artifacts do not accumulate.

**Authentication is Workload Identity — no storage account key.** The chart
mounts the container with the Azure Blob CSI driver using a Workload Identity
token, and both the backend and the Triton inference server read it as a
normal filesystem path. The storage account may keep shared key access
disabled. Grant the backend identity **Storage Blob Data Contributor** on the
storage account; a dedicated inference identity needs only **Storage Blob Data
Reader**, because its mount is read-only.

Each identity above needs a federated credential whose subject is the exact
Kubernetes service account that mounts as it
(`system:serviceaccount:<namespace>:<service-account>`). A mismatch fails the
mount with `AADSTS700213: No matching federated identity record found for
presented assertion subject`. The chart sets blobfuse's `allow_other` mount
option on the backend models volume so the backend process (uid 1000) can
read it; keep an equivalent option if you manage that PersistentVolume
yourself.

### Database

The platform stores its operational data in a MongoDB-compatible database.
Either option below works; pick on your own operational preference.

| Service | Recommended tier |
|---|---|
| MongoDB Atlas | `M40` |
| Azure Cosmos DB for MongoDB | Sized comparably to Atlas `M40` |

A smaller tier is workable for a proof of concept, with a migration planned
before production use. Supply the connection string as the
`DB_CONNECTION_STRING` key described under **Secrets**.

### Message broker

**No managed message broker is required.** The platform runs RabbitMQ inside
the cluster by default, installed and managed by the Helm chart, and the Azure
values files leave it that way. You do not need to provision Azure Event Hubs.

If you would rather run a managed broker, the platform also speaks Kafka
through Event Hubs. Provision a dedicated Event Hubs namespace on the
**Premium** SKU — the platform creates topics dynamically, and lower tiers cap
the number of event hubs per namespace — with a shared access policy granting
**Send** and **Listen**. Then set `common.env.data.MESSAGE_BROKER_TYPE` to
`kafka`, point `common.env.data.KAFKA_BOOTSTRAP_SERVERS` at the namespace's
Kafka endpoint, set `KAFKA_ENV: eventhub` and `KAFKA_SASL_MECHANISM: PLAIN`,
and supply the policy's connection string as the `KAFKA_PASSWORD` key
described under **Secrets**. Only one broker is active at a time.

--8<-- "_snippets/secrets.md"

### Blob storage needs no secret on Azure

Object storage is deliberately absent from the table above. The platform
authenticates to the storage account through the federated managed identities
described under **Cloud services**, so there is no storage account key to
create, store, or rotate. A legacy account-key mode
(`global.azure.useWorkloadIdentityForBlob: false`) is retained for one release
so existing installs can upgrade before switching; it requires shared key
access enabled on the storage account and will be removed.

--8<-- "_snippets/outbound-endpoints.md"

## Next steps

Once the cluster, the storage account, and the database exist, choose the
values file that matches your cluster mode — `azure-automatic.values.yaml` or
`azure-standard.values.yaml` — and fill in the placeholders it marks.

The platform chart is distributed through the AWS Marketplace container
registry as
`709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform`.
The registry is an Amazon ECR even when your platform runs on Azure, so
SpecificAI DevOps issues you an AWS access key ID and secret access key with
read-only access to pull the chart and images. The
[registry access page](../registry-access.md) describes the exchange.

Then follow the [install guide](../install.md) to log in to the registry,
install the chart with your values file, and verify the deployment. The chart
version you install determines the platform version; see the changelog for
what each release contains.
