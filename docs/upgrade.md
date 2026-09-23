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
refers to.

### 4.10.0 — 2026-09-22

⚠️ The Playground GPU pod's memory request/limit rises to 40Gi/52Gi so all sleeping vLLM engines' weights fit in RAM, and on AWS the gpu-basic tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier and node sizes you have provisioned before upgrading.

### 4.7.0 — 2026-09-12

⚠️ Karpenter node expiry and the node termination grace period now match the training and evaluation Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. NodeClaims are immutable, so nodes already running keep the old policy until they are replaced.

### 4.5.0 — 2026-09-08

⚠️ Existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on a bound PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading; the reclaim policy leaves the blob data untouched. Azure installs also need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` set. AWS and GCP are unaffected.
