# Changelog

## 4.10.2 — 2026-09-23

`playground` `infrastructure`

- The chart is now also distributed through a public GHCR channel: anonymous `helm pull` from `oci://ghcr.io/marketplace-specificai` with no registry login required, and every published chart version is signed with cosign so its provenance can be verified. The install documentation is restructured around a channel chooser, with the public GHCR channel first and AWS Marketplace ECR as the alternative. `infrastructure`
- Playground GPU inference (summarization / vLLM) is now enabled on GCP. The KEDA operator remains a cluster-wide singleton so multi-namespace installs share one controller. `playground` `infrastructure`

## 4.10.0 — 2026-09-22

`summarization` `content-generation` `playground` `infrastructure` `security`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> The Playground GPU pod's memory request/limit rises to 40Gi/52Gi so all sleeping vLLM engines' weights fit in RAM, and on AWS the gpu-basic tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier and node sizes you have provisioned before upgrading. See the requirements pages for details.

- New opt-in `common.karpenter.s3CsiStartupTaint` (off by default) makes AWS Karpenter nodes wait for the Mountpoint-S3 CSI driver to register before workloads schedule onto them. Turn it on only after upgrading Mountpoint-S3 CSI to 2.1.0 or later — the pinned v1.15 driver never clears the taint, which leaves workloads Pending. `infrastructure`
- Playground GPU pods now run one vLLM engine per base model a task needs, multiplexed on a single GPU. Idle engines park their weights in host RAM and exactly one is awake at a time, so a task whose trained models span several base models no longer needs a GPU each. LoRA adapters are always served on the base model they were trained on. `playground` `infrastructure`
- Base model selection for generative tasks (summarization and content generation) is available under Train -> Advanced. The choice persists on the usecase and drives training. `POST /update_selected_base_distilled_model` now requires the Editor or Admin role. `summarization` `content-generation`
- Capacity change for the Playground GPU pod: the vLLM wrapper moves to `vllm/vllm-openai:v0.28.0`, `specificai-vllm-inference.model.gpuMemoryUtilization` defaults to 0.65 per engine, and the pod's memory request/limit rises to 40Gi/52Gi so every sleeping engine's weights fit in RAM. On AWS the `gpu-basic` tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier you have provisioned before upgrading. `playground` `infrastructure`
- New secret `common.secrets.supervisorAdminToken` (`SUPERVISOR_ADMIN_TOKEN`) protects the Playground supervisor's admin endpoints. It is generated on first install when the chart manages the secret; set it yourself if you manage that secret externally. `security` `infrastructure`
- New optional values `specificai-vllm-inference.models.pvc.s3.bucketName` and `.volumeHandle` let the vLLM models volume share the backend bucket on AWS and pin a unique cluster-scoped Mountpoint-S3 handle. `backend.storage.s3.volumeHandle` is the matching override on the backend side. Leave all three empty for a single-namespace, one-bucket install. `infrastructure`
- The Playground base model is resolved per task from the model registry rather than from a single `global.vllmBaseModel`. `playground`
- AWS installs can opt in to serving the platform through the Kubernetes Gateway API (an ALB via the AWS Load Balancer Controller) instead of the nginx Ingress, with `global.gateway.enabled` and the new `global.gateway.aws` block (`scheme`, `targetType`, `idleTimeoutSeconds`, `loadBalancerName`). Installs that do not opt in keep the nginx Ingress; Azure and GCP are unchanged. The upstream Gateway API CRDs (standard channel v1.4.0) are vendored under `crds/`, so a plain `helm install` lands them; in-place CRD upgrades still need `kubectl apply -f crds/`. Still required on the cluster: AWS Load Balancer Controller >= 3.0.0 with an IAM role, a GatewayClass for `gateway.k8s.aws/alb`, and an ACM certificate covering your hostnames — TLS is discovered from ACM, so `global.gateway.tls.secretName` stays empty on AWS. EKS Auto Mode's built-in load balancer controller does not serve Gateway API. `infrastructure`

## 4.8.2 — 2026-09-16

`classification` `data-preparation`

- Fixed label generation. `classification` `data-preparation`

## 4.8.1 — 2026-09-13

`data-preparation` `ui`

- The Data Overview dataset details modal shows "Info is loading..." while a dataset analysis is still running, instead of reporting that no saved configuration exists, and fills the configuration in as soon as it arrives. `ui` `data-preparation`
- Training without a benchmark (auto-split) now copies the origin dataset's saved wizard configuration onto the promoted train and benchmark datasets, so the dataset details modal keeps showing it after the split. `data-preparation`

## 4.8.0 — 2026-09-13

`inference` `infrastructure`

- Serving-performance evaluation runs on its own isolated Triton replica (`specificai-triton-perf-eval`), scaled 0..1 per slot by KEDA with three slots per GPU tier by default and no customer ingress. Concurrent runs on the same GPU tier each take their own slot, so they cannot overwrite each other. `inference` `infrastructure`
- Serving-performance evaluation packages the trained encoder into an isolated model repository. It never writes your live `deployed_models` and does not require the model to be deployed first. `inference`
- Backend API pods are annotated so the cluster autoscaler and Karpenter avoid evicting them, which previously could interrupt an in-progress serving-performance run. `infrastructure`
- The standing Triton image is pinned to `25.02-py3-r2`, which ships the classification and NER BLS backends. The previously pinned `25.02-py3` predates them and would not be re-pulled on a same-tag rebuild. `inference` `infrastructure`

## 4.7.0 — 2026-09-12

`infrastructure`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> Karpenter node expiry and the node termination grace period now match the training and evaluation Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. NodeClaims are immutable, so nodes already running keep the old policy until they are replaced. See the requirements pages for details.

- Karpenter node pools no longer force-kill long-running training and evaluation Jobs. Node expiry and the node termination grace period now match the Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. Previously a node was reclaimed 24 hours after boot and the run failed with "Training job terminated before cleanup". Idle nodes are still recycled within five minutes. Upgrade note: NodeClaims are immutable, so nodes already running keep the old policy until they are replaced. `infrastructure`

## 4.6.0 — 2026-09-09

`data-preparation`

- New dataset collection flow: Retrieval Agent. `data-preparation`
- New dataset collection flow: Generate Data with AI. `data-preparation`

## 4.5.0 — 2026-09-08

`infrastructure` `security`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> Existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on a bound PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading; the reclaim policy leaves the blob data untouched. Azure installs also need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` set. AWS and GCP are unaffected. See the requirements pages for details.

- Azure Blob storage no longer needs a storage account key. The backend models volume and the Triton inference model repository both mount the models container with the Blob CSI driver using Workload Identity, so the storage account can disable shared-key access entirely. Controlled by `global.azure.useWorkloadIdentityForBlob`, which defaults to true; set it to false to restore the previous account-key path. `infrastructure` `security`
- Azure installs now need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` (`enabled`, `accountName`, `containerName`, `resourceGroup`) — the resource group of the storage account, not the AKS `MC_*` group. The render fails with an actionable message when the Workload Identity path is on and those are missing. `infrastructure`
- Each Blob PersistentVolume presents its own `clientID` alongside the mounting pod's ServiceAccount token, so the two must belong to the same identity; a mismatch fails the mount with `AADSTS700213`. Read-only mounts need only `Storage Blob Data Reader`, while the read-write backend mount needs `Storage Blob Data Contributor`. `infrastructure` `security`
- Upgrade note for existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on an existing PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading. The reclaim policy leaves the blob data untouched. `infrastructure`
- AWS and GCP behaviour is unchanged. `infrastructure`

## 4.4.2 — 2026-09-10

`data-preparation` `infrastructure`

- `common.secrets.kaggleApiToken` is an override rather than a requirement. The Retrieval Agent's public-dataset search and download work out of the box, and the value only needs setting to point at your own Kaggle account. `data-preparation` `infrastructure`

## 4.4.1 — 2026-09-09

`inference`

- Model deploy reliability: all Triton image references are aligned, Triton stays alive through long startups and model loads, and out-of-memory deploy failures are surfaced with an actionable message instead of a generic one. `inference`

## 4.4.0 — 2026-09-08

`infrastructure` `security`

- KEDA RabbitMQ auth can read the password from an existing Secret. Set `global.kedaRabbitmqAuth.existingSecret` (typically the same Secret as `global.envFromSecret`) and the chart renders a credential-free AMQP URI while the TriggerAuthentication reads the password by key (`passwordKey`, default `RABBITMQ_PASSWORD`; optional `usernameKey`). This covers both RabbitMQ-scaled subcharts and removes the need for `global.rabbitmqPassword`, so installs that manage the application Secret with External Secrets, Vault or Sealed Secrets no longer put a password in their values file. Opt-in: left unset, the render is unchanged. `infrastructure` `security`

## 4.3.0 — 2026-09-08

`model-providers`

- Vertex AI authentication mode is resolved from the running platform instead of a stored value. On GCP, Vertex defaults to Workload Identity even when no mode was ever saved, and the settings panel reports the effective mode rather than a stale or misleading toggle. `model-providers`

## 4.2.0 — 2026-09-08

`infrastructure` `security`

- Optional Okta OIDC as a first-class identity provider (`common.optune_auth.method=OKTA` plus the `web_app_okta_*` values). The values and secrets are empty by default, so an install that does not set them is unaffected. `BOTH` remains Google plus Azure only and never includes Okta. `security` `infrastructure`

## 4.1.1 — 2026-09-06

- No customer-facing changes.

## 4.1.0 — 2026-08-31

`playground` `inference` `infrastructure`

- Saving a model to disk as a Docker image no longer streams through the ingress: `GET /download-model` redirects to a signed object-storage URL and the browser's download manager fetches the `.tar`. Multi-gigabyte images previously failed with a network error after roughly 4GB or 600 seconds. The frontend ingress proxy read and send timeouts are raised from 600s to 3600s as a fallback. `inference` `infrastructure`
- Playground and the Docker download path both take vLLM from the chart's appVersion, so they stay on the same build; vLLM is pinned to 0.27.1. The models volume is templated with GCS Fuse for GKE Autopilot. `playground` `infrastructure`
- Image-builder Jobs inherit `global.podLabels`, which Azure Workload Identity requires. Without it the Job falls back to IMDS and blob list and get fail with `AuthorizationPermissionMismatch`. `infrastructure`
- The Azure backend models volume mounts blobfuse with `-o allow_other` so the platform can list trained LoRA adapters. Without it the Playground returns an empty model catalog. `playground` `infrastructure`
- On Azure, `specificai-vllm-inference.gpuTier` is the knob for the Playground GPU tier; it defaults to `gpu-basic`. `playground` `infrastructure`
- `specificai-vllm-inference` and KEDA can be enabled on Azure, with the models volume on Blob CSI using Workload Identity tokens and no account key. `playground` `infrastructure`

## 4.0.0 — 2026-08-19

`infrastructure`

- New system design. `infrastructure`
