# SpecificAI Platform — GCP requirements

SpecificAI Platform is a self-hosted, Kubernetes-native application for
creating task-specific models. It runs entirely inside your own Google Cloud
project: your datasets, your trained models, and your inference traffic never
leave it.

This page lists the Google Cloud infrastructure to provision before installing
the platform Helm chart — the cluster, the node pools, the networking, and the
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

The platform is deployed onto a Google Kubernetes Engine cluster in your
project, new or existing. Both GKE modes are supported, and each has its own
values file.

| Cluster mode | Values file | Node provisioning |
|---|---|---|
| GKE Standard | `gcp-standard.values.yaml` | The chart renders a GKE `ComputeClass` per GPU tier and GKE auto-creates matching node pools on demand. Requires GKE `1.33.3-gke.1136000` or later. |
| GKE Autopilot | `gcp-autopilot.values.yaml` | Google-managed nodes, selected through Autopilot's own compute classes and accelerator labels. |

!!! warning "The chart cannot detect which mode your cluster runs in"

    Applying the wrong file leaves GPU pods pending indefinitely — Autopilot
    already manages its own compute classes, so the Standard file's
    `ComputeClass` objects do not do what you expect there, and the Autopilot
    file provisions nothing on a Standard cluster. Confirm the cluster mode
    before you choose.

On GKE Standard the chart also installs the NVIDIA device plugin DaemonSet,
because Standard nodes are not guaranteed to ship one preinstalled. Autopilot
installs the GPU driver and device plugin itself, so the chart's own DaemonSet
stays disabled there. Each values file already sets this correctly.

For the platform core, provision a dedicated node pool of `c3-standard-32`
with a minimum of two nodes. That is 64 vCPU and 256 GB against a floor of
26 vCPU and 50 GB, which leaves room for rolling upgrades and for the burst of
short-lived pods the platform creates while a job starts. A dedicated pool
also keeps the platform isolated from your other workloads. GPU capacity is
separate and is covered below.

The cluster needs Workload Identity Federation for GKE enabled — object
storage authentication depends on it, as described under **Cloud services**.

## Node pools

The chart schedules work onto six named tiers. Each tier is a `workload` label
value; the chart's node selectors and matching tolerations are already set in
the values files, so you do not assign these by hand.

| Node pool | Instance type | Accelerator | Purpose |
|---|---|---|---|
| `gpu-high-performance` | Per region — see below | NVIDIA L40S | Generative and high-VRAM training. |
| `gpu-basic` | `g2-standard-8` | NVIDIA L4 | Playground inference and general GPU training. |
| `gpu-low-performance` | `n1-standard-8` | NVIDIA T4 | Classification and NER training and evaluation. |
| `cpu-high-performance` | Platform core pool | — | Shares the platform core capacity. |
| `cpu-basic` | Platform core pool | — | CPU training, data processing, and evaluation. |
| `cpu-low-performance` | Platform core pool | — | Shares the platform core capacity. |

The three CPU tiers run on the platform core pool on GCP rather than on a
separate CPU pool; the platform distinguishes them so that other clouds can
separate them, and so you can split them later by overriding the tier's node
selector. On Autopilot the values file maps them to Autopilot's built-in
`Performance` and `Balanced` compute classes instead, because there is no
customer-named pool to share.

GPU accelerator availability varies by region and project quota. The
`ComputeClass` tiers the chart renders on GKE Standard default to
`nvidia-l4` on `g2-standard-8` for `gpu-basic` and `nvidia-tesla-t4` on
`n1-standard-8` for `gpu-low-performance`; set the accelerator for each tier
under `global.computeClasses.tiers` to match what your project can provision,
including the high-performance tier. On Autopilot, the tiers are selected with
the `cloud.google.com/gke-accelerator` node label instead, as the values file
shows.

If you prefer to pre-create GPU node pools yourself instead of letting GKE
auto-create them, set `global.computeClasses.create` to `false` and name the
pools `gpu-high-performance`, `gpu-basic`, and `gpu-low-performance` — the
chart then targets them by pool name automatically.

## Networking

From chart version 3.1.x the platform is exposed on GCP through the Kubernetes
Gateway API rather than a legacy Ingress. The chart renders the `Gateway` and
`HTTPRoute` objects; you provide the Gateway infrastructure.

1. Install the Gateway API CRDs (`gateway.networking.k8s.io/v1`) and verify
   with `kubectl get gatewayclass`.
2. Ensure a Gateway controller serves the cluster — the chart does not install
   one. On GKE, enable the built-in Gateway controller and use the
   `gke-l7-global-external-managed` GatewayClass.
3. Create a DNS record for `global.optuneAddress` pointing at the gateway's
   IP address. Reserving a named static IP keeps the record stable.

| Helm value | Meaning |
|---|---|
| `global.optuneAddress` | The hostname the platform is served on. Used as the Gateway host and as the base of the single sign-on redirect URI. |
| `global.gateway.enabled` | Use Gateway API. Set `true` on GCP, together with `global.ingress.createIngress: false` — the chart refuses to render both. |
| `global.gateway.gatewayClassName` | The GatewayClass the Gateway binds to. `gke-l7-global-external-managed` on GKE. |
| `global.gateway.namedAddress` | Name of a reserved static IP for the Gateway load balancer. Optional. |

TLS on GKE is handled by Certificate Manager: attach your certificate map
through the `networking.gke.io/certmap` annotation via
`global.gateway.annotations`.

The cluster needs outbound connectivity to the GCS bucket, to the database, to
the registry the platform images are pulled from, and to the small set of
third-party endpoints listed under **Outbound endpoints** below.

## Cloud services

### Object storage

One dedicated GCS bucket holds datasets, checkpoints, and trained model
artifacts. Set it on `global.bucketName`.

The bucket must contain a `deployed_models/` prefix. The Triton inference
server uses it as its model repository path and will not start without it.

Enable CORS on the bucket. The platform uploads and downloads large files
directly from the browser using signed URLs, which fail without it.

```terraform
cors = [
  {
    origin          = ["https://<your platform hostname>"]
    method          = ["PUT"]
    response_header = ["ETag", "Content-Type"]
    max_age_seconds = 3600
  }
]
```

Add a lifecycle rule so temporary download artifacts do not accumulate.

```terraform
lifecycle_rules = [
  {
    action = { type = "Delete" }
    condition = {
      age            = 1
      matches_prefix = "downloads/"
    }
  }
]
```

Access is granted through Workload Identity Federation for GKE: create a
Google Cloud service account, grant it the **Storage Bucket Admin** role on
the bucket, and pass its email as `global.roleId`. The Kubernetes service
accounts the chart creates authenticate as that service account.

The platform mounts the bucket with the Cloud Storage FUSE CSI driver, which
ships with GKE but is enabled per cluster. Autopilot enables it by default; on
GKE Standard, enable the add-on:

```bash
gcloud container clusters update <your-cluster> --region <your-region> \
  --project <your-project> --update-addons=GcsFuseCsiDriver=ENABLED
```

### Database

The platform stores its operational data in a MongoDB-compatible database.

| Service | Recommended tier |
|---|---|
| MongoDB Atlas | `M40` |

A smaller tier is workable for a proof of concept, with a migration planned
before production use. Supply the connection string as the
`DB_CONNECTION_STRING` key described under **Secrets**.

### Message broker

**No managed message broker is required.** The platform runs RabbitMQ inside
the cluster by default, installed and managed by the Helm chart, and the GCP
values files leave it that way.

If you would rather run a managed broker, the platform also speaks Kafka. Set
`common.env.data.MESSAGE_BROKER_TYPE` to `kafka`, point
`common.env.data.KAFKA_BOOTSTRAP_SERVERS` at your Kafka service, set
`KAFKA_SASL_MECHANISM` to what that service uses (`PLAIN` for most
Kafka-compatible services on GCP), and supply the `KAFKA_PASSWORD` key
described under **Secrets**. The default `KAFKA_ENV: cloud` is already correct
on GCP. Only one broker is active at a time.

--8<-- "_snippets/secrets.md"

--8<-- "_snippets/outbound-endpoints.md"

## Next steps

Once the cluster, the bucket, and the database exist, choose the values file
that matches your cluster mode — `gcp-standard.values.yaml` or
`gcp-autopilot.values.yaml` — and fill in the placeholders it marks.

Then follow the [install guide](../install.md) to pull the chart — it is public
on GitHub Container Registry and needs no credentials — install it with your
values file and your Docker token, and verify the deployment. The chart version you install determines the platform version;
see the changelog for what each release contains.
