# SpecificAI Platform — Upgrade guide

This page describes how to move a running installation to a newer chart
version: pull the new chart, reconcile your values file against the freshly
published template, upgrade, and verify — plus a per-version checklist of
releases that need infrastructure attention before you upgrade.

Upgrades reuse the files from the [install guide](install.md): your
values file and your `secrets.values.yaml`. The chart is public on GitHub
Container Registry, so no login is involved.

## Check the pre-upgrade checklist

Before anything else, read the
[pre-upgrade checklist](#pre-upgrade-checklist-by-version) at the bottom of
this page. It lists every release that changes infrastructure in a way that
may need action on your side — GPU tier sizing, storage migrations, node-pool
policy changes. **When you skip versions, every entry between your current
version and the target applies**, not just the target's.

## Pull the new chart version

Pull the target version anonymously to inspect what you are
about to install, and to get that version's values templates:

```bash
helm pull oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version <TARGET_VERSION> --untar
```

Chart versions before 4.13.0 are available on request from
[SpecificAI support](support.md).

The
[changelog](https://github.com/marketplace-specificai/specificai-platform/blob/main/CHANGELOG.md)
lists what each release contains.

## Diff your values against the new template

Each release republishes the starter values files (the same six files the
[install guide](install.md) chooses from). Diff your customized copy against
the freshly published template for your cluster type:

```bash
diff your-values.yaml <freshly-published-template>.values.yaml
```

You are looking for keys the new version added (new features you may want,
new required values called out in the checklist below) and defaults that
changed. Fold anything relevant into your copy — your copy stays the source
of truth; never edit the published template in place.

## Moving to the generated values templates

From chart 4.13.0 the six values templates are generated from the chart's own
defaults instead of being written by hand, and a separate
`secrets.values.example.yaml` ships next to them. If your values file started
from a template published before 4.13.0, read this section once before your
first upgrade to 4.13.0 or later.

### What changed in the templates

Each template is now much longer, but sets far less than its length suggests:

- **Live keys** are only what your cluster type needs: the required values
  (with `<YOUR_*>` placeholders), the cloud-specific settings, and the
  `enabled` switch of each optional block.
- **Commented `# key: default` lines** list every other setting you can
  change, each with the chart's current default and a `# --` description.
  They are documentation, not configuration. **Do not copy them into your
  file.** Anything you leave out keeps following the chart default when a
  later release changes it; copying a default pins it.
- **Credentials are gone from the cluster templates.** They live in
  `secrets.values.example.yaml` instead.

### Compare your file with the new template

Because the new template is mostly comments, a plain `diff` is noisy.
Compare only what each file actually sets, with comments stripped and keys
sorted ([yq](https://github.com/mikefarah/yq) v4):

```bash
yq 'sort_keys(..) | ... comments=""' your-values.yaml > yours.flat.yaml
yq 'sort_keys(..) | ... comments=""' <cluster>.values.yaml > template.flat.yaml
diff yours.flat.yaml template.flat.yaml
```

- **Only in the template:** a required or cloud-specific value your file
  lacks. Add it. The **Required values** section of your cloud's requirements
  page lists every required value with its description.
- **Only in your file:** either a setting you changed on purpose (keep it),
  a default you copied (the template shows it commented with the same value;
  you may delete it so it tracks future defaults), a credential (move it, see
  below), or a key the chart no longer reads (see
  [Keys the chart no longer reads](#keys-the-chart-no-longer-reads)).

### Move credentials to a second file

Copy `secrets.values.example.yaml` to `secrets.values.yaml`, move every
credential out of your values file into it, and pass both files to `helm`,
values file first:

```bash
helm upgrade --install specificai \
  oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version <TARGET_VERSION> \
  --namespace specificai \
  --values your-values.yaml \
  --values secrets.values.yaml
```

Keys set live in the example are needed on every install; commented ones only
for the feature their description names. Keys that share a placeholder must
hold the same value — the RabbitMQ password appears as both
`common.secrets.rabbitmqPassword` and `rabbitmq.auth.password`. Keep
`secrets.values.yaml` out of version control.

### The chart now validates required values

The chart's values schema, which Helm applies on every `install`, `upgrade`,
`template` and `lint`, now fails the command instead of letting a broken
install through when:

- a `global.cloudProvider`, `global.cloudRegion`, `global.bucketName`,
  `global.roleId` or `global.optuneAddress` value still holds a `<...>`
  placeholder;
- `global.bucketName` or `global.optuneAddress` is empty;
- `global.cloudProvider` is `azure` or `gcp` and `global.roleId` is empty.
  On AWS an empty `global.roleId` still installs: pods then run with the EKS
  worker node role or an EKS Pod Identity association.

Set the value named in the error and rerun the command.

### Azure: storage account key installs

Since chart 4.5.0, `global.azure.useWorkloadIdentityForBlob` defaults to
`true`: the chart mounts Blob storage with Workload Identity and no longer
renders the storage account key Secret. An Azure values file that still
relies on a storage account key and does not set
`specificai-inference.models.blob.enabled` fails the upgrade with
`specificai-inference: global.azure.useWorkloadIdentityForBlob is true (the
default) on Azure, so Triton has no storage account key`. Pick one:

- **Move to Workload Identity (recommended).** Set up the identities,
  federated credentials and role assignments described under **Object
  storage** on the Azure requirements page, then add:

    ```yaml
    global:
      roleId: "<YOUR_MANAGED_IDENTITY_CLIENT_ID>"
    backend:
      storage:
        blob:
          resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"
    specificai-inference:
      models:
        blob:
          enabled: true
          accountName: "<YOUR_STORAGE_ACCOUNT_NAME>"
          containerName: "<YOUR_CONTAINER_NAME>"
          resourceGroup: "<YOUR_STORAGE_ACCOUNT_RESOURCE_GROUP>"
    ```

    and remove `backend.storage.blob.accountKey`. The backend's Blob
    PersistentVolume changes mount attributes, which Kubernetes does not
    allow on a bound volume: scale the backend down and delete
    `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading
    (the reclaim policy leaves the blob data untouched). Once the upgrade is
    healthy, the storage account can disable shared key access.

- **Keep the storage account key for now.** Set
  `global.azure.useWorkloadIdentityForBlob: false` and keep
  `backend.storage.blob.accountName` and `backend.storage.blob.accountKey`
  (in `secrets.values.yaml`). This legacy mode needs shared key access on the
  storage account and will be removed in a later release.

### Keys the chart no longer reads

Helm ignores keys the chart does not read, so these install without error
and silently do nothing. Delete them, and set the replacement where there is
one.

| Key in older values files | Replacement |
|---|---|
| `global.tag` | None. Image tags follow the chart version; install the chart version you want. |
| `backend.storage.s3.bucketName` | `global.bucketName`, which the chart already reads for the S3 bucket. |
| `common.karpenter.clusterName` | `common.karpenter.discoveryTag` (AWS standard): the `karpenter.sh/discovery` tag value on your subnets and security groups. |
| `specificai-inference.gpuTier` | None. The Triton inference server runs on the platform nodes (`global.nodeSelector`, `global.tolerations`). GPU Playground inference is placed by `specificai-vllm-inference.gpuTier`. |
| `specificai-inference.scheduleOnGpuNodes` | None, as for `specificai-inference.gpuTier`. |
| `job-runner.job.activeDeadlineSeconds` | `job-runner.job.gpuActiveDeadlineSeconds` and `job-runner.job.cpuActiveDeadlineSeconds`. |
| `global.gpuTrainingNodeSelector`, `global.gpuEvaluationNodeSelector` | The GPU tier selectors: `global.gpuHighPerformanceNodeSelector`, `global.gpuBasicNodeSelector`, `global.gpuLowPerformanceNodeSelector`. |
| `global.cpuTrainingNodeSelector`, `global.cpuEvaluationNodeSelector` | The CPU tier selectors: `global.cpuHighPerformanceNodeSelector`, `global.cpuBasicNodeSelector`, `global.cpuLowPerformanceNodeSelector`. |
| `global.trainingTolerations`, `global.evaluationTolerations` | The matching tier tolerations, such as `global.gpuBasicTolerations` or `global.cpuBasicTolerations`. |
| `global.gpuCustomNodeSelector`, `global.gpuCustomTolerations` | None. Map the workload to one of the three GPU tiers instead. |

## Upgrade

The upgrade is the install command with the new `--version`:

```bash
helm upgrade --install specificai \
  oci://ghcr.io/marketplace-specificai/specificai-platform \
  --version <TARGET_VERSION> \
  --namespace specificai \
  --values your-values.yaml \
  --values secrets.values.yaml
```

## Verify the rollout

```bash
helm status specificai --namespace specificai
kubectl get pods --namespace specificai
```

Within a few minutes every pod should be `Running` or `Completed`, same as
after an install. GPU nodes appear on demand, so an idle platform showing no
GPU nodes is normal.

## Roll back if needed

Helm keeps the previous release revisions:

```bash
helm history specificai --namespace specificai
helm rollback specificai <REVISION> --namespace specificai
```

A rollback restores the previous chart version and values. Note that it does
not undo infrastructure changes you made by hand for the upgrade (deleted
volumes, resized node pools) — the checklist entries below call out when a
change is one-way.

## Pre-upgrade checklist by version

Generated from the release catalog. Only versions that need
infrastructure attention are listed; a version that is absent needs
no action beyond the standard procedure above. When you skip
versions, apply every entry between your current version and the
target, newest last. The cloud requirements pages
([AWS](requirements/aws.md), [Azure](requirements/azure.md),
[GCP](requirements/gcp.md)) describe the infrastructure each note
refers to, and the [infrastructure changelog](infra-changelog.md)
has the per-cloud steps for each entry.

### 4.13.0 — 2026-09-30

⚠️ Platform container images are now served from the `specificai/platform` Docker Hub repository. Set `common.secrets.dockerConfigJson` (the `docker-ro-creds` image pull Secret) to the registry token SpecificAI provides, in the same `helm upgrade` that installs this version. If you manage `docker-ro-creds` outside Helm (`global.createSecret: false`), update that Secret before upgrading. Without the new token, pods fail with `ImagePullBackOff`. The chart's values schema now also fails the upgrade when `global.cloudRegion`, `global.bucketName`, `global.roleId` or `global.optuneAddress` still holds a `<...>` placeholder, when `global.bucketName` or `global.optuneAddress` is empty, or when `global.roleId` is empty on Azure or GCP. Set those values first; the upgrade guide's "Moving to the generated values templates" section covers the rest of the move. [Step-by-step instructions](infra-changelog.md#4130-2026-09-30).

### 4.10.0 — 2026-09-22

⚠️ The Playground GPU pod's memory request/limit rises to 40Gi/52Gi so all sleeping vLLM engines' weights fit in RAM, and on AWS the gpu-basic tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier and node sizes you have provisioned before upgrading. [Step-by-step instructions](infra-changelog.md#4100-2026-09-22).

### 4.7.0 — 2026-09-12

⚠️ Karpenter node expiry and the node termination grace period now match the training and evaluation Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. NodeClaims are immutable, so nodes already running keep the old policy until they are replaced. [Step-by-step instructions](infra-changelog.md#470-2026-09-12).

### 4.5.0 — 2026-09-08

⚠️ Existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on a bound PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading; the reclaim policy leaves the blob data untouched. Azure installs also need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` set. AWS and GCP are unaffected. [Step-by-step instructions](infra-changelog.md#450-2026-09-08).
