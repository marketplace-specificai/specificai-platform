# Changelog

## 4.14.7 — 2026-10-20

`infrastructure`

- The message broker is no longer restarted when a Karpenter-managed cluster (including EKS Auto Mode) consolidates its nodes. The broker pod was protected only from the Cluster Autoscaler, so Karpenter could drain the node it ran on, and every service lost its broker connection until the pod came back. The broker pod now also carries the annotation Karpenter honors. Upgrading restarts the broker once. `infrastructure`

## 4.14.6 — 2026-10-19

`infrastructure` `security`

- With Traefik, the new `global.ingress.traefik.middlewareNamespace` makes the chart create its `Middleware` objects in a namespace that Traefik's CRD provider watches. Set it when `providers.kubernetesCRD.namespaces` leaves out the platform namespace, where Traefik otherwise ignores the Middlewares and the public API paths stop working. The Middleware names then start with the platform namespace, and the Ingresses reference them there. Left empty, nothing changes. The Traefik requirements now also say to include the platform namespace in `providers.kubernetesIngress.crossProviderNamespaces` if you set it, and to keep `providers.kubernetesCRD.safeNaming` off. `infrastructure`
- `global.runtimeEnv` is now the one value to set for the prefix of message broker topic and queue names. The platform ConfigMap's `RUNTIME_ENV` follows it when `common.env.data.RUNTIME_ENV` is empty, and so do the KEDA queues of `specificai-vllm-inference`, `data-processor` and `specificai-triton-perf-eval`, which used `local` unless set per subchart; a subchart's own `runtimeEnv` still overrides it. If you set both `global.runtimeEnv` and `common.env.data.RUNTIME_ENV`, they must be equal, or the render fails. `infrastructure`
- The backend and the batch-poller, teacher-fetcher, judgment-fetcher and generation-fetcher jobs now pull their images with the Secret named in `global.imagePullSecretsName`, like every other platform workload. `backend.imagePullSecrets` and the batch subcharts' `imagePullSecrets` values are no longer read; if you set them, make sure `global.imagePullSecretsName` names that Secret. With the default `docker-ro-creds`, nothing changes. `infrastructure`
- The chart README has a new section, "Secrets managed outside Helm (GitOps, External Secrets)", listing the Secrets and values an install needs when External Secrets, Vault or Sealed Secrets provides its credentials. It covers the RabbitMQ Erlang cookie Secret to create when the broker password lives in a Secret, and the `coralogix-keys` Secret, which `global.createSecret: false` does not cover: set `common.coralogix.secret.create: false` as well and keep that name, because the observability agent always reads it. `infrastructure` `security`

## 4.14.5 — 2026-10-15

`classification`

- Classification training no longer tries a model that is not available in your installation. On a task with no stored languages, each attempt to load that model spent about 30 seconds and logged a failure before training continued with the other models. Training results are unchanged. `classification`

## 4.14.4 — 2026-10-14

`ner`

- Entity recognition training on very large datasets is no longer killed while it evaluates. Evaluation now scores each batch as it goes instead of holding every prediction for the whole evaluation set in memory, so its memory use no longer grows with the dataset. Reported metrics are unchanged. `ner`

## 4.14.3 — 2026-10-13

`data-preparation`

- Generate data with AI no longer fails immediately when the goal description, a label or entity description, or an example contains template syntax such as `${{ secrets.AWS_SECRET_ACCESS_KEY }}` or `{{ .Values.key }}`. That syntax is now preserved rather than interpreted, so the run generates data instead of ending with "No samples were generated" without generating anything. `data-preparation`

## 4.14.2 — 2026-10-12

`data-preparation` `model-providers`

- Long teacher evaluations no longer restart from the beginning when they outlast the message broker's acknowledgement timeout. The handler now acknowledges an evaluation when it receives it, so a realtime evaluation (a model provider without batch support) runs once instead of being redelivered and rerun from zero. An evaluation that is in progress when the handler restarts is no longer retried automatically; start it again. `model-providers`
- Retrieval Agent runs get the same change: a run that outlasts the acknowledgement timeout is no longer rerun from scratch, and a run that is in progress when the handler restarts is no longer retried automatically. `data-preparation`

## 4.14.1 — 2026-10-12

`infrastructure`

- The chart can serve the platform through Traefik's Kubernetes Ingress provider. Set `global.ingress.controller: traefik` and `global.ingress.className` to your Traefik IngressClass: the chart then renders Prefix paths and Traefik `Middleware` objects for the `/public/api`, `/public/triton` and `/public/playground` path rewrites instead of ingress-nginx annotations. Traefik's Kubernetes CRD provider must be enabled. The default, `nginx`, renders the same Ingress objects as before. `infrastructure`

## 4.14.0 — 2026-10-08

`classification` `ner` `summarization` `content-generation` `data-preparation` `inference` `model-providers` `infrastructure` `security`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> Platform container images are now served from the `specificai/platform` Docker Hub repository. Set `common.secrets.dockerConfigJson` (the `docker-ro-creds` image pull Secret) to the registry token SpecificAI provides, in the same `helm upgrade` that installs this version. If you manage `docker-ro-creds` outside Helm (`global.createSecret: false`), update that Secret before upgrading. Without the new token, pods fail with `ImagePullBackOff`. The chart's values schema now also fails the upgrade when `global.cloudRegion`, `global.bucketName`, `global.roleId` or `global.optuneAddress` still holds a `<...>` placeholder, when `global.bucketName` or `global.optuneAddress` is empty, or when `global.roleId` is empty on Azure or GCP. Set those values first; the upgrade guide's "Moving to the generated values templates" section covers the rest of the move. [Step-by-step instructions](https://marketplace-specificai.github.io/specificai-platform/infra-changelog/#4140-2026-10-08).

- Generate labels with AI on 100 or more unlabeled records now runs through the model provider's batch API when the selected model supports it, preferring a batch-capable model among the task's configured models; smaller runs keep realtime calls. The progress bar shows labeled / remaining counts and the current phase, and cancelling the run while the provider processes the batch also stops the provider's batch job. The threshold is `common.env.data.LABEL_GENERATION_BATCH_MIN_RECORDS` (default 100) and the batch path needs the `batch-poller` and `generation-fetcher` subcharts. `classification` `data-preparation`
- A label generation run that stops early (for example on a database cursor error) now ends as failed and reports how many labels were saved, instead of reporting success. `classification` `data-preparation`
- On AWS with Amazon DocumentDB, the `batch-poller` KEDA trigger can now open the database's TLS CA file: the chart renders the `specificai-rds-ca-bundle` ConfigMap with the Amazon RDS root CAs of `global.cloudRegion` and mounts it into its KEDA operator at `/app/certs`. A KEDA installed outside the chart needs the same mount (chart README, "KEDA and Amazon DocumentDB"); without it the poller never runs and batch jobs stay in provider processing. `infrastructure`
- Platform container images are now served from the `specificai/platform` Docker Hub repository. Set `common.secrets.dockerConfigJson` to the registry token SpecificAI provides, in the same `helm upgrade` that installs this version. `infrastructure` `security`
- The job-runner Deployment now sets `imagePullSecrets` from `global.imagePullSecretsName`, like every other platform workload. `infrastructure`
- Model image builds authenticate the base-image pull with the image pull Secret, mounted read-only into the image-builder Job. Pushes to ECR, GCR / Artifact Registry and ACR use it alongside the destination credential; a Docker Hub push builds to a local tarball with the pull credential and then pushes with your Docker Hub token alone. `inference` `infrastructure`
- Base model selection under Train -> Advanced is now a searchable dropdown available for every task type. It defaults to "Automatic", which lets the platform pick the base model, or can pin one model for that training only; the choice is no longer saved on the task. Classification also offers MiniLM, a small and fast model. Generative tasks on Automatic now train the model the training handler's configuration names. `POST /update_selected_base_distilled_model` was removed. `classification` `ner` `summarization` `content-generation`
- New batch processing services (batch-poller, teacher-fetcher, judgment-fetcher, generation-fetcher) split vendor LLM batch submission, polling, and result fetching into independently scaled KEDA jobs. All four are enabled by default and each can be disabled per subchart; batch label generation needs batch-poller and generation-fetcher. `infrastructure`
- The six values templates are now generated from the chart itself. Each sets live only what its cluster type needs and lists every other setting as a commented line showing the chart default, so nothing you leave commented is pinned. Credentials moved to a separate `secrets.values.example.yaml` for a two-file install. `infrastructure`
- The chart's values schema rejects an install that leaves a `<...>` placeholder in the core `global` values or leaves `global.bucketName`, `global.optuneAddress` or (on Azure and GCP) `global.roleId` empty, instead of failing later at runtime. On AWS an empty `global.roleId` still installs and leaves pods on the EKS node role or Pod Identity; the AWS service account no longer gets an empty `eks.amazonaws.com/role-arn` annotation in that case. `infrastructure`
- Each cloud's requirements page now has a Required values table and the Secrets section a table of credential values, both generated from the chart. The upgrade guide covers moving an existing values file to the new templates, the Azure storage-account-key upgrade path, and keys the chart no longer reads. `infrastructure`
- A new Infrastructure changelog page lists every chart version with the infrastructure change it needs, if any, as step-by-step Console, CLI and Verify instructions for each affected cloud. Every "an infrastructure change is required" warning in the changelog and the upgrade guide now links to that version's section. `infrastructure`
- Agentic tasks (Autonomy Level 2-4) now train on the uploaded tool-calling trajectories. Summarization and content-generation agents fine-tune on the full user / assistant / tool conversation, including each tool call's name and arguments, instead of the input and expected output alone. Rows whose trajectory has no assistant turn are left out of training and counted in the dataset reduction summary, and a training run with nothing left to train on fails with an explicit reason that tells apart missing trajectories from trajectories without an assistant turn. `summarization` `content-generation`
- Generate Data with AI now supports content-generation tasks with their own examples and settings. The Short / Medium / Long text length now sizes the generated output, and the task prompt and output schema are applied to every generated record, both for new datasets and for generating responses on existing ones. `content-generation` `data-preparation`
- Retrieval Agent, Generate data with AI and label generation now work when only one model provider is configured: a selected or stored provider without credentials falls back to a configured provider (same model family preferred) instead of failing with 'Provider … not registered'. `model-providers` `data-preparation`
- If the model provider's credentials are removed while a label batch is still processing, the run now fails within about five minutes with a clear message instead of polling forever; labels saved so far are kept. `model-providers` `data-preparation`
- Single-label classification can optimize training on one label. When that label is set, training keeps the checkpoint with the best PR AUC for that label, and samples that label as 20% of each epoch when it is rarer than that. Metrics for both labels are still reported. `classification`
- When training data holds more than one language, the validation split, the automatically created benchmark and the hyperparameter-search validation data keep the same language mix as the training data, as closely as each language's row count allows. `classification`

## 4.11.4 — 2026-10-08

`infrastructure`

- The RabbitMQ Erlang cookie no longer changes between renders. When `rabbitmq.auth.erlangCookie` is empty, the chart derives a stable cookie for each install from the broker password, the release name and the namespace, and stores it in its own Secret (`rabbitmq.auth.existingErlangSecret`). Tools that render the chart with `helm template`, such as Argo CD, no longer restart the broker on every sync. Set `rabbitmq.auth.erlangCookie` to pin your own value, or point `rabbitmq.auth.existingErlangSecret` at a Secret you manage. The broker restarts once during this upgrade. `infrastructure`

## 4.11.3 — 2026-10-03

`infrastructure` `ui`

- Telemetry export gains metadata-only fields for the user-journey funnel: the task's last creation-wizard step, the improvement method of a training run (AI, human or both), whether a run is a re-evaluation, per-task training / improvement run counters, and a per-version quality-delta gauge that carries the previous version's score as the baseline. No task content or identities are exported. `infrastructure`
- The task creation wizard stamps the current step on the task (POST /task-wizard-step) so observability can show where drafts stall; nothing in the product reads the value back. `infrastructure` `ui`

## 4.11.2 — 2026-10-01

`classification`

- Classification training for a Chinese task can select hfl/chinese-roberta-wwm-ext, hfl/chinese-macbert-base, FacebookAI/xlm-roberta-base, google-bert/bert-base-multilingual-uncased, and distilbert/distilbert-base-multilingual-cased. A task that is both English and Chinese selects the multilingual models, which cover both languages. `classification`

## 4.11.1 — 2026-10-01

`classification` `ner` `summarization` `content-generation` `infrastructure` `ui`

- Evaluation metrics are exported to observability per section (general metrics, per-class metrics, span-level metrics, threshold calibration, generative classic and LLM-judge scores), one row per training run, model kind and language, each redacted and size-capped on its own. Large label sets are no longer trimmed out of the export, and re-evaluation runs are attributed to the training version they re-evaluated. `infrastructure` `classification` `ner` `summarization` `content-generation`
- Quality telemetry (accuracy, precision, recall, F-score, confusion matrix, model-response matrix, summarization metrics and judge criteria) is exported per language and per label, matching the selections available on the Evaluation screen. `infrastructure` `classification` `ner` `summarization` `content-generation`
- Starting a training run with languages the selected base model does not support now emits a structured warning log (`alert_fields` are lifted to top-level log keys) so it can be alerted on. `infrastructure` `classification` `ner`
- Two-label classification tasks ask "What matters most for these labels?" (equally important, don't miss a label, or avoid its false detections) in Business Goals and in Edit Task. The choice sets the Fallback label and the training defaults (F-score beta and the "Optimize on" label in Train > Advanced); Data Collection and Train block when the chosen label is no longer one of the task's labels. Existing two-label tasks default to "Both labels are equally important". `ui` `classification`
- The Evaluation Threshold card gains a False Positive Rate section: FPR = FP / (FP + TN) for the selected label, previewed live, with a slider and numeric input linked to the threshold slider. `ui` `classification`

## 4.11.0 — 2026-09-27

`classification` `ner` `summarization` `content-generation` `playground` `infrastructure` `ui`

- Playground GPU inference (summarization / vLLM) is now enabled on GCP. The KEDA operator remains a cluster-wide singleton so multi-namespace installs share one controller. `playground` `infrastructure`
- Model Configuration is redesigned. Generative tasks (summarization and content generation) get Commercial model / SpecificAI model tabs: the SpecificAI tab configures the student system prompt, user prompt, and output format per field as Same as commercial, Disable, or Customize, replacing the Train -> Advanced "Include prompt in training & evaluation" checkbox. The user prompt editor shows template tokens as pills with a Template/Preview switch and an Additional fields rail for inserting dataset columns as context. New "When evaluating" settings (Normalize casing, Strip whitespace, Fix JSON format) clean model outputs before evaluation comparison for all task types (POST /save_evaluation_normalization, Editor or Admin role). `ui` `summarization` `content-generation` `classification` `ner`

## 4.10.2 — 2026-09-23

`playground` `data-preparation` `infrastructure` `ui`

- The chart is now also distributed through a public GHCR channel: anonymous `helm pull` from `oci://ghcr.io/marketplace-specificai` with no registry login required, and every published chart version is signed with cosign so its provenance can be verified. The install documentation is restructured around a channel chooser, with the public GHCR channel first and AWS Marketplace ECR as the alternative. `infrastructure`
- Playground GPU inference (summarization / vLLM) is now enabled on GCP. The KEDA operator remains a cluster-wide singleton so multi-namespace installs share one controller. `playground` `infrastructure`
- Retrieval Agent can research Hugging Face and Kaggle dataset candidates with Tavily before searching those hubs. Editors and Admins set a per-user Tavily API key under Settings → Tavily API Key. `data-preparation` `ui`

## 4.10.0 — 2026-09-22

`summarization` `content-generation` `playground` `infrastructure` `security`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> The Playground GPU pod's memory request/limit rises to 40Gi/52Gi so all sleeping vLLM engines' weights fit in RAM, and on AWS the gpu-basic tier promotes g5.2xlarge to g5.4xlarge. Review the GPU tier and node sizes you have provisioned before upgrading. [Step-by-step instructions](https://marketplace-specificai.github.io/specificai-platform/infra-changelog/#4100-2026-09-22).

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
> Karpenter node expiry and the node termination grace period now match the training and evaluation Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. NodeClaims are immutable, so nodes already running keep the old policy until they are replaced. [Step-by-step instructions](https://marketplace-specificai.github.io/specificai-platform/infra-changelog/#470-2026-09-12).

- Karpenter node pools no longer force-kill long-running training and evaluation Jobs. Node expiry and the node termination grace period now match the Jobs' own deadlines (72h on GPU pools, 168h on CPU pools) on AWS and Azure. Previously a node was reclaimed 24 hours after boot and the run failed with "Training job terminated before cleanup". Idle nodes are still recycled within five minutes. Upgrade note: NodeClaims are immutable, so nodes already running keep the old policy until they are replaced. `infrastructure`

## 4.6.0 — 2026-09-09

`data-preparation`

- New dataset collection flow: Retrieval Agent. `data-preparation`
- New dataset collection flow: Generate Data with AI. `data-preparation`

## 4.5.0 — 2026-09-08

`infrastructure` `security`

> ⚠️ **Before upgrading, an infrastructure change is required.**
> Existing Azure installs: `spec.csi.volumeAttributes` and `mountOptions` are immutable on a bound PersistentVolume, so scale the backend down and delete `<namespace>-backend-pvc` and `<namespace>-backend-pv` before upgrading; the reclaim policy leaves the blob data untouched. Azure installs also need `backend.storage.blob.resourceGroup` and `specificai-inference.models.blob` set. AWS and GCP are unaffected. [Step-by-step instructions](https://marketplace-specificai.github.io/specificai-platform/infra-changelog/#450-2026-09-08).

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
