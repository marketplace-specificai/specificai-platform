# Changelog

## 4.9.0

- Playground GPU pod runs an in-pod supervisor/router (`specificai-vllm-inference` 0.3.3 -> 0.6.0): one vLLM engine per base model a task's trained models need, multiplexed on a single GPU with vLLM level-1 sleep mode (weights parked in host RAM, exactly one engine awake behind a GPU lock). LoRAs are always served on the base they were trained on; the backend routes each call with the `X-SpecificAI-Base` header and gates session readiness per task on `/admin/healthz`.
- Train -> Advanced "Base model selection" for generative tasks (summarization / content generation), backed by the decoder registry via `GET /generative-base-models`; the per-task choice persists on the usecase and drives training. `POST /update_selected_base_distilled_model` now requires Editor/Admin and validates against the registry.
- vLLM wrapper image moves to upstream `vllm/vllm-openai:v0.28.0` (sleep mode requires >= 0.28.0). `specificai-vllm-inference.model.gpuMemoryUtilization` defaults to 0.65 (flat per engine; fits K=5 on a 16 GB T4) and the pod memory request/limit is raised to 40Gi/52Gi so all sleeping engines' weights fit in RAM (AWS gpu-basic promotes g5.2xlarge -> g5.4xlarge).
- New `common.secrets.supervisorAdminToken` (`SUPERVISOR_ADMIN_TOKEN`) gates the supervisor's `/admin/prepare` and `/admin/healthz`; auto-generated on first install when the chart manages the secret, shared with the backend via the same Secret. `/admin/ping` stays open for probes.
- Optional `specificai-vllm-inference.models.pvc.s3.volumeHandle` / `bucketName` so the vLLM models PV can share the backend bucket and pin a unique cluster-scoped Mountpoint-S3 handle (empty keeps the namespace-prefixed template default). `backend.storage.s3.volumeHandle` is documented as the matching optional override (default stays `s3-csi-driver-volume` for single-namespace / one-bucket installs).
- AWS Karpenter NodePools apply the official Mountpoint-S3 CSI startup taint `s3.csi.aws.com/agent-not-ready:NoExecute` so nodes wait until s3-csi-node registers before workloads schedule.
- `global.vllmBaseModel` and the deployment-handler `VLLM_BASE_MODEL` default now come from the decoder registry's inference weights id; the live Playground pod no longer reads a single global base.

## 4.8.1

- Coralogix Customer filter: remap AWS/Azure namespace `integration` to application `aws-integration` / `azure-integration` via transform/app_override (same pattern as GCP `gcp-integration`). K8s namespace stays `integration`.
- Data Overview dataset details modal ("AI-generated Dataset" / "Dataset Retrieval Agent") shows "Info is loading..." while the dataset analysis is still loading, instead of the misleading "No saved configuration is available for this dataset.", and fills in the saved configuration as soon as it arrives.
- Auto-train progress banner status copy reads "Data flows finished, starting training…" (removed the em dash).
- Auto-split (training without a benchmark) now copies the origin dataset's saved wizard configuration (``source_config``, ``created_by_user``, ``created_datetime``) onto the promoted train and benchmark datasets, so the "Generate Data with AI" / "Dataset Retrieval Agent" details modal keeps showing the configuration after the split instead of "No saved configuration is available for this dataset."

## 4.8.0

- Restore the 4.2.0-4.7.0 changelog and chart version after PR #2586 overwrote Chart.yaml with a develop-lineage copy (version regressed to 4.1.3); no feature code was affected.
- Isolated Triton serving-perf eval replica (specificai-triton-perf-eval): KEDA 0..1 per slot (default 3 slots per GPU tier), no customer ingress, used by on-demand model-perf-runs.
- Serving-perf eval packages a trained encoder into an isolated ``perf_eval_models`` Triton repo and never writes live ``deployed_models`` or requires Deploy.
- Serving-perf eval concurrent runs on the same GPU tier each get their own KEDA slot and ``perf_eval_models/slots/<tier>/<slot>`` Triton repo so they cannot overwrite each other.
- Serving-perf eval Triton loads BLS backends from the inference image (Helm emptyDir overlay) and writes TensorRT cache under ``/tmp/trt_cache`` so ``gs://`` / object-store model repos can start.
- Serving-perf eval init copies BLS backends onto emptyDir with plain ``cp -R`` (no ``--preserve``) so uid 1000 can install ``classification_bls`` / ``ner_bls`` without chmod/utime CrashLoop.
- Serving-perf eval Triton uses a startupProbe and liveness on ``/v2/health/live`` so the first TensorRT compile after a cold emptyDir cache is not a liveness CrashLoop.
- Serving-perf KEDA session TTL covers unbilled replica startup (Ready + first-infer warmup) plus the billed load, not estimate+5, so Quick no longer dies on a cold cluster.
- Backend API pods are annotated ``safe-to-evict=false`` / ``do-not-disrupt`` so Autopilot/Karpenter is less likely to SIGTERM the in-process serving-perf runner.
- Pin the standing Triton image to 25.02-py3-r2 (Hub SHA 35b8723480) so classification_bls / ner_bls backends ship. Frozen 25.02-py3 is pre-BLS and IfNotPresent will not roll a same-tag rebuild.
- Align job-runner BASE_TRITON_IMAGE and compose publish tag with the chart ref (specificai-inference-25.02-py3-r2) so Helm and the image-builder agree.

## 4.7.0

- Karpenter NodePools no longer force-kill long-running training/evaluation Jobs. GPU pools had `expireAfter: 5m` (CPU: 30m), so every node started draining minutes after boot; the Jobs' `karpenter.sh/do-not-disrupt` pods blocked the drain, but EKS Auto Mode applies a default 24h `terminationGracePeriod` to NodeClaims, forcibly reclaiming the node — and killing the Job — exactly 24h after boot ("Training job terminated before cleanup"). NodePools now set `expireAfter` and an explicit `terminationGracePeriod` matching the Jobs' `activeDeadlineSeconds` (72h GPU, 168h CPU) on AWS and Azure. Idle nodes are still recycled within 5 minutes by `WhenEmpty` consolidation. Note: NodeClaims are immutable, so existing nodes keep the old policy until they are replaced.

## 4.6.0

- Add new dataset collection flow: Retrieval Agent.
- Add new dataset collection flow: Generate Data with AI.

## 4.5.0

- Azure Blob storage no longer needs a storage account key. The backend models volume and the Triton inference model repository both mount the models container with the Blob CSI driver using Workload Identity (`mountWithWorkloadIdentityToken` + the identity behind the mounting pod's ServiceAccount), so the storage account can disable shared-key access entirely. Controlled by `global.azure.useWorkloadIdentityForBlob`, which defaults to true; set it to false to restore the previous account-key path (`optune-storage-account-secret`, `nodeStageSecretRef`, and Triton's `as://` model repository with AZURE_STORAGE_KEY).
- Azure installs now need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` (`enabled`, `accountName`, `containerName`, `resourceGroup`) — the resource group of the storage account, not the AKS `MC_*` group. The render fails with an actionable message when the Workload Identity path is on and those are missing.
- Each Blob PV presents its own `clientID` alongside the mounting pod's ServiceAccount token, so the two must belong to the same identity. Triton's PV resolves `clientID` from `specificai-inference.serviceAccount.roleId` (falling back to `global.roleId`) through the same expression that annotates its ServiceAccount; a mismatch fails the mount with `AADSTS700213`. Read-only mounts need only `Storage Blob Data Reader`; the read-write backend mount needs `Storage Blob Data Contributor`.
- Operator note for upgrades: `spec.csi.volumeAttributes` and `mountOptions` are immutable on an existing PersistentVolume, so an existing Azure install must scale the backend down and delete `<namespace>-backend-pvc` / `<namespace>-backend-pv` before upgrading. The reclaim policy leaves the blob data untouched.
- AWS and GCP behaviour is unchanged.

## 4.4.2

- `common.secrets.kaggleApiToken` is now an override rather than a requirement: the agents_handler image ships SpecificAI's Kaggle token, so the Retrieval Agent's public-dataset search and download work out of the box and the value only needs setting to point at your own Kaggle account.

## 4.4.1

- Model deploy reliability: align all Triton image references on `specificai-inference`, keep Triton alive during long startup/model loads with a `/v2/health/live` liveness probe plus startup probe and `--strict-readiness=false`, and surface actionable out-of-memory deploy failures to users.

## 4.4.0

- KEDA RabbitMQ auth from an existing Secret: set `global.kedaRabbitmqAuth.existingSecret` (typically the same Secret as `global.envFromSecret`, `specificai-secrets`) and the chart renders a credential-free AMQP URI while the TriggerAuthentication reads the password from that Secret by key (`passwordKey`, default `RABBITMQ_PASSWORD`; optional `usernameKey`, empty means the username stays a plain value in the chart Secret). Covers both RabbitMQ-scaled subcharts, `specificai-vllm-inference` and `data-processor`, and removes the need for `global.rabbitmqPassword`. Installs that manage the app Secret with External Secrets, Vault or Sealed Secrets get this without adding a password to their values file. Opt-in: left unset, the password is still inlined into the URI and the render is identical to 4.1.1.

## 4.3.0

- Vertex AI auth mode is now resolved dynamically instead of trusting a stored value: on GCP platforms Vertex defaults to Workload Identity even when no auth mode was ever saved, and the settings API exposes the effective mode plus the platform's cloud provider so the frontend panel can honestly show "enabled by default on GCP", "unsupported off GCP", or the explicitly saved state instead of a stale or misleading toggle.

## 4.2.0

- Optional Okta OIDC as a first-class Helm IDP (common.optune_auth.method=OKTA plus web_app_okta_*). Values and secrets stay empty by default so hosted gcp/dev* remains Google. BOTH is Google+Azure only and never includes Okta.

## 4.1.1

- Coralogix Customer filter: remap the GCP `integration` namespace to application `gcp-integration` via transform/app_override. The Kubernetes namespace and its ingress host are unchanged.

## 4.1.0

- Docker Save-to-disk: GET /download-model 302s to a signed object-storage URL and the frontend uses the browser download manager for .tar files so a ~9GB bake is not streamed through nginx (was TypeError network error after ~4GB / 600s). Frontend ingress proxy-read/send-timeout 600 → 3600 as stream fallback.
- Playground and docker-download vLLM use Chart.AppVersion so both stay on the same Helm SHA. vLLM 0.27.1 is pinned in specificai_vllm_inference/Dockerfile (FROM vllm/vllm-openai:v0.27.1), not by a static wrapper tag. GCS Fuse models volume is templated for GKE Autopilot; ci/gcp.values.yaml keeps specificai-vllm-inference.enabled false.
- TEMP azure/dev: pin all GPU tiers (HIGH/BASIC/LOW/training/eval) to T4 NC8 and restrict Azure GPU NodePools (gpu-high, gpu-basic, gpu-low) to NC8/NC16 only via ci/azure.values.yaml while L40S/NV36 quota is blocked. Shared template defaults stay customer-facing (gpu-high sku-gpu-name L40S). Not a customer SKU-name list.
- Image-builder Jobs inherit global.podLabels (Azure Workload Identity). Without azure.workload.identity/use the Job falls back to IMDS and blob list/get returns AuthorizationPermissionMismatch while Playground CSI still works.
- Azure backend models PV: add blobfuse `-o allow_other` so uid 1000 can list trained LoRAs. Without it Playground POST /usecase-models returns an empty SpecificAI catalog.
- Azure Playground vLLM: gpuTier is the operator knob (default gpu-basic). azure/dev falls back to gpu-low-performance with T4 NC8/NC16 affinity while NV36 quota is blocked.
- Enable specificai-vllm-inference + KEDA on Azure with Blob CSI models volume (Workload Identity token-only, no account key).

## 4.0.0

- New system design
