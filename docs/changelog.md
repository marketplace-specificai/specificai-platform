# Changelog

## 4.9.0

- Playground GPU pods now run one vLLM engine per base model a task needs, multiplexed on a single GPU. Idle engines park their weights in host RAM and exactly one is awake at a time, so a task whose trained models span several base models no longer needs a GPU each. LoRA adapters are always served on the base model they were trained on.
- Base model selection for generative tasks (summarization and content generation) is available under Train -> Advanced. The choice persists on the usecase and drives training. `POST /update_selected_base_distilled_model` now requires the Editor or Admin role.
- Capacity change for the Playground GPU pod: the vLLM wrapper moves to `vllm/vllm-openai:v0.28.0`, `specificai-vllm-inference.model.gpuMemoryUtilization` defaults to 0.65 per engine, and the pod's memory request/limit rises to 40Gi/52Gi so every sleeping engine's weights fit in RAM. On AWS the `gpu-basic` tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier you have provisioned before upgrading.
- New secret `common.secrets.supervisorAdminToken` (`SUPERVISOR_ADMIN_TOKEN`) protects the Playground supervisor's admin endpoints. It is generated on first install when the chart manages the secret; set it yourself if you manage that secret externally.
- New optional values `specificai-vllm-inference.models.pvc.s3.bucketName` and `.volumeHandle` let the vLLM models volume share the backend bucket on AWS and pin a unique cluster-scoped Mountpoint-S3 handle. `backend.storage.s3.volumeHandle` is the matching override on the backend side. Leave all three empty for a single-namespace, one-bucket install.
- AWS Karpenter node pools now carry the Mountpoint-S3 CSI startup taint, so a new node waits for the S3 CSI driver to register before workloads schedule onto it.
- The Playground base model is resolved per task from the model registry rather than from a single `global.vllmBaseModel`.

## 4.8.2 — 2026-09-16

- Fixed label generation.

## 4.8.1 — 2026-09-13

- The Data Overview dataset details modal shows "Info is loading..." while a dataset analysis is still running, instead of reporting that no saved configuration exists, and fills the configuration in as soon as it arrives.
- Training without a benchmark (auto-split) now copies the origin dataset's saved wizard configuration onto the promoted train and benchmark datasets, so the dataset details modal keeps showing it after the split.

## 4.8.0 — 2026-09-13

- Serving-performance evaluation runs on its own isolated Triton replica (`specificai-triton-perf-eval`), scaled 0..1 per slot by KEDA with three slots per GPU tier by default and no customer ingress. Concurrent runs on the same GPU tier each take their own slot, so they cannot overwrite each other.
- Serving-performance evaluation packages the trained encoder into an isolated model repository. It never writes your live `deployed_models` and does not require the model to be deployed first.
- Backend API pods are annotated so the cluster autoscaler and Karpenter avoid evicting them, which previously could interrupt an in-progress serving-performance run.
- The standing Triton image is pinned to `25.02-py3-r2`, which ships the classification and NER BLS backends. The previously pinned `25.02-py3` predates them and would not be re-pulled on a same-tag rebuild.

## 4.7.0

- Karpenter node pools no longer force-kill long-running training and evaluation Jobs. Node expiry and the node termination grace period now match the Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. Previously a node was reclaimed 24 hours after boot and the run failed with "Training job terminated before cleanup". Idle nodes are still recycled within five minutes. Upgrade note: NodeClaims are immutable, so nodes already running keep the old policy until they are replaced.

## 4.6.0

- New dataset collection flow: Retrieval Agent.
- New dataset collection flow: Generate Data with AI.

## 4.5.0

- Azure Blob storage no longer needs a storage account key. The backend models volume and the Triton inference model repository both mount the models container with the Blob CSI driver using Workload Identity, so the storage account can disable shared-key access entirely. Controlled by `global.azure.useWorkloadIdentityForBlob`, which defaults to true; set it to false to restore the previous account-key path.
- Azure installs now need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` (`enabled`, `accountName`, `containerName`, `resourceGroup`) — the resource group of the storage account, not the AKS `MC_*` group. The render fails with an actionable message when the Workload Identity path is on and those are missing.
- Each Blob PersistentVolume presents its own `clientID` alongside the mounting pod's ServiceAccount token, so the two must belong to the same identity; a mismatch fails the mount with `AADSTS700213`. Read-only mounts need only `Storage Blob Data Reader`, while the read-write backend mount needs `Storage Blob Data Contributor`.
- Upgrade note for existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on an existing PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading. The reclaim policy leaves the blob data untouched.
- AWS and GCP behaviour is unchanged.

## 4.4.2

- `common.secrets.kaggleApiToken` is an override rather than a requirement. The Retrieval Agent's public-dataset search and download work out of the box, and the value only needs setting to point at your own Kaggle account.

## 4.4.1

- Model deploy reliability: all Triton image references are aligned, Triton stays alive through long startups and model loads, and out-of-memory deploy failures are surfaced with an actionable message instead of a generic one.

## 4.4.0

- KEDA RabbitMQ auth can read the password from an existing Secret. Set `global.kedaRabbitmqAuth.existingSecret` (typically the same Secret as `global.envFromSecret`) and the chart renders a credential-free AMQP URI while the TriggerAuthentication reads the password by key (`passwordKey`, default `RABBITMQ_PASSWORD`; optional `usernameKey`). This covers both RabbitMQ-scaled subcharts and removes the need for `global.rabbitmqPassword`, so installs that manage the application Secret with External Secrets, Vault or Sealed Secrets no longer put a password in their values file. Opt-in: left unset, the render is unchanged.

## 4.3.0

- Vertex AI authentication mode is resolved from the running platform instead of a stored value. On GCP, Vertex defaults to Workload Identity even when no mode was ever saved, and the settings panel reports the effective mode rather than a stale or misleading toggle.

## 4.2.0

- Optional Okta OIDC as a first-class identity provider (`common.optune_auth.method=OKTA` plus the `web_app_okta_*` values). The values and secrets are empty by default, so an install that does not set them is unaffected. `BOTH` remains Google plus Azure only and never includes Okta.

## 4.1.1

- No customer-facing changes.

## 4.1.0

- Saving a model to disk as a Docker image no longer streams through the ingress: `GET /download-model` redirects to a signed object-storage URL and the browser's download manager fetches the `.tar`. Multi-gigabyte images previously failed with a network error after roughly 4GB or 600 seconds. The frontend ingress proxy read and send timeouts are raised from 600s to 3600s as a fallback.
- Playground and the Docker download path both take vLLM from the chart's appVersion, so they stay on the same build; vLLM is pinned to 0.27.1. The models volume is templated with GCS Fuse for GKE Autopilot.
- Image-builder Jobs inherit `global.podLabels`, which Azure Workload Identity requires. Without it the Job falls back to IMDS and blob list and get fail with `AuthorizationPermissionMismatch`.
- The Azure backend models volume mounts blobfuse with `-o allow_other` so the platform can list trained LoRA adapters. Without it the Playground returns an empty model catalog.
- On Azure, `specificai-vllm-inference.gpuTier` is the knob for the Playground GPU tier; it defaults to `gpu-basic`.
- `specificai-vllm-inference` and KEDA can be enabled on Azure, with the models volume on Blob CSI using Workload Identity tokens and no account key.

## 4.0.0

- New system design.
