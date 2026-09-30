<!--
  DO NOT EDIT in the public repository. This is the public site's landing
  page, generated from .devops/docs/customer-facing/public/index.md in
  SpecificAI's private sources by tools/marketplace_mirror/transform_content.py
  and republished on every chart release. The private repository's own docs
  build uses src/index.md instead; the two differ because only the public
  layout has the changelog and values pages.
-->

# SpecificAI Platform

SpecificAI Platform is a self-hosted, Kubernetes-native application for
creating task-specific models. It runs on your own cloud account: your
datasets, your trained models, and your inference traffic never leave it.

This site is generated from SpecificAI's private sources and republished on
each chart-version release.

## Deploying the platform

1. Provision the infrastructure for your cloud:
   [AWS](requirements/aws.md), [Azure](requirements/azure.md), or
   [GCP](requirements/gcp.md).
2. Get your Docker token for the platform images from SpecificAI — see
   [Registry access](registry-access.md). The Helm chart itself is public on GHCR and
   needs no credentials.
3. Pick the [Helm values template](values/index.md) that matches your cluster
   type and follow the [install guide](install.md).

The [changelog](changelog.md) lists what each chart release contains.
