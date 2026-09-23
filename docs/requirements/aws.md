# SpecificAI Platform — AWS requirements

SpecificAI Platform is a self-hosted, Kubernetes-native application for
creating task-specific models. It runs entirely inside your own AWS account:
your datasets, your trained models, and your inference traffic never leave it.

This page lists the AWS infrastructure to provision before installing the
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

The platform is deployed onto an Amazon EKS cluster in your account, new or
existing. Both EKS modes are supported, and each has its own values file.

| Cluster mode | Values file | Node provisioning |
|---|---|---|
| EKS Auto Mode | `aws-auto-mode.values.yaml` | Auto Mode's built-in Karpenter-compatible controller and NodeClass. |
| EKS standard | `aws-standard.values.yaml` | Your own Karpenter controller; the chart renders an `EC2NodeClass` for it. |

!!! warning "The chart cannot detect which mode your cluster runs in"

    Applying the wrong file leaves GPU pods pending indefinitely, because the
    NodeClass or IAM role Karpenter needs to launch GPU nodes will not exist.
    Confirm the cluster mode before you choose.

On the standard path the chart also installs the NVIDIA device plugin
DaemonSet, because plain EC2 nodes do not come with one. Auto Mode installs the
GPU driver and device plugin itself, so the chart's own DaemonSet stays
disabled there. Each values file already sets this correctly.

For the platform core, provision a dedicated node group of `m7i.8xlarge` with
a minimum of two nodes. That is 64 vCPU and 256 GiB against a floor of 26 vCPU
and 50 GB, which leaves room for rolling upgrades and for the burst of
short-lived pods the platform creates while a job starts. A dedicated group
also keeps the platform isolated from your other workloads. GPU capacity is
separate and is covered below.

## Node pools

The chart schedules work onto six named tiers. Each tier is a `workload` label
value; the chart's node selectors and matching tolerations are already set in
the values files, so you do not assign these by hand.

| Node pool | Instance types | Accelerator | Purpose |
|---|---|---|---|
| `gpu-high-performance` | `g6e.4xlarge` | NVIDIA L40S | Generative and high-VRAM training. |
| `gpu-basic` | `g5` family | NVIDIA A10G | Playground inference and general GPU training. |
| `gpu-low-performance` | `g4dn` family | NVIDIA T4 | Classification and NER training and evaluation. |
| `cpu-high-performance` | `c6i`, `c6a` families | — | Shares the `cpu-basic` pool. |
| `cpu-basic` | `c6i`, `c6a` families | — | CPU training, data processing, and evaluation. |
| `cpu-low-performance` | `c6i`, `c6a` families | — | Shares the `cpu-basic` pool. |

The three CPU tiers deliberately map to one pool on AWS; the platform
distinguishes them so that other clouds can separate them, and so you can
split them later by overriding the tier's node selector.

Karpenter provisions these nodes on demand as jobs are queued and consolidates
them away five minutes after they go idle, so no GPU node runs continuously
just because the platform is online.

## Networking

The chart exposes the platform through a Kubernetes Ingress. You provide the
ingress controller; the chart does not install one.

| Helm value | Meaning |
|---|---|
| `global.optuneAddress` | The hostname the platform is served on. Used as the Ingress host and as the base of the single sign-on redirect URI. |
| `global.ingress.createIngress` | Whether the chart renders the Ingress. Default `true`. |
| `global.ingress.className` | The `ingressClassName` on the rendered Ingress. Default `nginx`. |

Create a DNS record for `global.optuneAddress` pointing at your ingress
controller's load balancer. Use a CNAME to the load balancer's DNS name rather
than an A record to a resolved address, which changes.

**TLS is terminated in front of the cluster on AWS.** The Ingress the chart
renders on AWS carries no `tls` block, so bring your own certificate — an ACM
certificate on the load balancer is the usual arrangement. The chart's
`global.ingress.createCertificateSecret` path is Azure-only and does nothing
here.

The cluster needs outbound connectivity to the S3 bucket, to the database,
to the registry the platform images are pulled from, and to the small set of
third-party endpoints listed under **Outbound endpoints** below. Gateway API
is not used on AWS; leave `global.gateway.enabled` at `false`.

## Cloud services

### Object storage

One dedicated S3 bucket holds datasets, checkpoints, and trained model
artifacts. Set it on `global.bucketName` and on `backend.storage.s3.bucketName`,
with the region on `backend.storage.s3.region`.

The bucket must contain a `deployed_models/` prefix. The Triton inference
server uses it as its model repository path and will not start without it.

Enable CORS on the bucket. The platform uploads and downloads large files
directly from the browser using pre-signed URLs, which fail without it.

```terraform
cors_rule {
  allowed_headers = ["*"]
  allowed_methods = ["PUT"]
  allowed_origins = ["https://<your platform hostname>"]
  expose_headers  = ["ETag", "Content-Type"]
  max_age_seconds = 3000
}
```

Add a lifecycle rule so temporary download artifacts do not accumulate.

```terraform
rule {
  id     = "expire-downloads"
  status = "Enabled"
  filter { prefix = "downloads/" }
  expiration { days = 1 }
  noncurrent_version_expiration { noncurrent_days = 1 }
  abort_incomplete_multipart_upload { days_after_initiation = 1 }
}
```

Grant access with IAM Roles for Service Accounts and pass the role ARN as
`global.roleId`; the EKS worker node role works as a fallback. These are the
only S3 actions the platform performs, so the policy can stop here.

```terraform
actions = [
  "s3:GetObject",
  "s3:PutObject",
  "s3:DeleteObject",
  "s3:AbortMultipartUpload",
  "s3:ListMultipartUploadParts"
]
resources = ["arn:aws:s3:::<your-bucket>/*"]

actions   = ["s3:ListBucket"]
resources = ["arn:aws:s3:::<your-bucket>"]
```

### Database

The platform stores its operational data in a MongoDB-compatible database.
Either option below works; pick on your own operational preference.

| Service | Recommended tier |
|---|---|
| MongoDB Atlas | `M40` |
| Amazon DocumentDB | `db.r5.xlarge` |

A smaller tier is workable for a proof of concept, with a migration planned
before production use. Supply the connection string as the
`DB_CONNECTION_STRING` key described under **Secrets**.

### Message broker

**No managed message broker is required.** The platform runs RabbitMQ inside
the cluster by default, installed and managed by the Helm chart, and the AWS
values files leave it that way. You do not need to provision Amazon MSK.

If you would rather run a managed broker, the platform also speaks Kafka. Set
`common.env.data.MESSAGE_BROKER_TYPE` to `kafka`, point
`common.env.data.KAFKA_BOOTSTRAP_SERVERS` at your MSK cluster, and supply the
`KAFKA_PASSWORD` key described under **Secrets**. On MSK the defaults
`KAFKA_SASL_MECHANISM: SCRAM-SHA-512`, `KAFKA_SECURITY_PROTOCOL: SASL_SSL`, and
`KAFKA_ENV: cloud` are already correct. Only one broker is active at a time.

--8<-- "_snippets/secrets.md"

--8<-- "_snippets/outbound-endpoints.md"

## Next steps

Once the cluster, the bucket, and the database exist, choose the values file
that matches your cluster mode — `aws-auto-mode.values.yaml` or
`aws-standard.values.yaml` — and fill in the placeholders it marks.

Then follow the [install guide](../install.md) to pull the chart — it is public
on GitHub Container Registry and needs no credentials — install it with your
values file and your Docker token, and verify the deployment. The chart version you install
determines the platform version; see the changelog for what each release
contains.
