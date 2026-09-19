## SpecificAI Platform Requirements Note

### Intro

SpecificAI Platform is a self-hosted kubernetes native application for creating task-specific models.  
SpecificAI is meant to be running on a cloud provider environment, such as Azure and AWS.

The following services are required to be set prior of SpecificAI Platform installation:

* Storage  
  * AWS: S3 Bucket  
  * Azure: Storage Account with blob container  
* Kafka  
  * AWS: MSK  
  * Azure: Eventhub Namespace  
* MongoDB Compatible database cluster  
  * AWS: DocumentDB  
  * Azure: CosmosDB  
  * MongoDBAtlas  
* Kubernetes  
  * AWS: EKS  
  * Azure: AKS

Here’s the breakdown of each of the above services recommended configurations for Azure Cloud services.

### 

### Azure Cloud Services

#### Storage (Blob container)

SpecificAI Platform requires of a dedicated blob container included with a dedicated directory named `deployed_models`

Motivation: While inferring models, Triton server (installed as part of SpecificAI Helm solution) will set this directory as its repository-path.

**Authentication: Workload Identity, no storage account key.** From chart **4.5.0** the platform reaches the blob container only through the federated managed identities described under *Azure federated identity* below. The Helm chart mounts the container with the Azure Blob CSI driver using a Workload Identity token (`mountWithWorkloadIdentityToken`), and both the backend and the Triton inference server read it as a normal filesystem path. **No storage account key is requested, stored, or rendered into a Kubernetes Secret**, so the storage account may keep **shared key access disabled**.

The **platform (backend) identity** needs **Storage Blob Data Contributor** on the storage account (or its resource group) because its mount is read-write; a **dedicated read-only inference identity** needs only **Storage Blob Data Reader**. The container must be reachable from the AKS node subnet.

Values to supply at install time:

| Helm value | Meaning |
| :---- | :---- |
| `backend.storage.blob.accountName` | Storage account name. |
| `backend.storage.blob.containerName` | Blob container name. |
| `backend.storage.blob.resourceGroup` | Resource group **of the storage account** — **not** the AKS node resource group (`MC_*`). Required; the CSI driver uses it to resolve the account when mounting with a token. |
| `global.roleId` | Client ID (UUID) of the user-assigned managed identity federated with the platform service account. This is the identity the **backend** models volume mounts as. |
| `specificai-inference.serviceAccount.roleId` | Client ID of the identity federated with the **Triton inference** service account, when that component runs under its own service account. This is the identity the **Triton** models volume mounts as. Leave unset to reuse `global.roleId`. |

Each blob mount presents its PV's client ID together with the **mounting pod's own** service-account token, so every identity above needs a federated credential whose **subject is that exact service account** (`system:serviceaccount:<namespace>:<service-account>`). A mismatch fails the mount with `AADSTS700213: No matching federated identity record found for presented assertion subject ...`. **Read-only mounts need only `Storage Blob Data Reader`**; the read-write backend mount needs `Storage Blob Data Contributor`.

A legacy account-key mode (`global.azure.useWorkloadIdentityForBlob: false`) is retained for **one release** so existing installs can upgrade before switching. It requires a storage account key and shared key access enabled, and will be removed.

#### Models volume (Playground catalog)

Playground lists SpecificAI fine-tuned models from the **models blob/container CSI mount** on the **backend** pod (the same container used for trained artifacts). That mount must be readable by the backend process user (**uid 1000**).

Azure Blob CSI / blobfuse mounts as root by default. Set blobfuse **`-o allow_other`** on the backend models PersistentVolume (or StorageClass `mountOptions`). Without it, the backend cannot list the volume and Playground shows no SpecificAI model — even when training completed and the LoRA exists in the container.

The SpecificAI Helm chart sets this option on the backend models PV. If you manage the volume yourself, keep `allow_other` (or an equivalent that grants uid 1000 read/list access). After a change, remount the backend pods so the new options take effect.

After the mount is readable, Playground shows **trained versions** (for example V1.0). Recalculate-evaluation minor versions (for example 1.01) are evaluation-only and do not appear as a new Playground model.

Then click **Start GPU** to serve the trained model.

#### Kafka (Eventhub Namespace)

SpecificAI Platform requires a dedicated eventhub namespace. The platform will create the topics (referred as EventHubs) dynamically.

* SKU: Premium (Required for multiple hubs per namespace; [docs](https://learn.microsoft.com/en-us/azure/event-hubs/compare-tiers#:~:text=50%20per%20CU\)-,Number%20of%20event%20hubs%20per%20namespace,-10))  
* Capacity: 1  
* Authorization rule: Send, Listen  
* Authorization rule SAS \- Will be used by the platform components for authentication (connectionString).

#### MongoDB (MongoDBAtlas)

Due to sizing, we recommend using the M40 MongoDBAtlas tier. The equivalent sizing used by AWS is [r5.xlarge](https://instances.vantage.sh/aws/ec2/r5.xlarge?currency=USD).  
In case of deciding to start with a lower tier, we could set it so as a platform POC step, and along the way, determine a migration plan.

#### Kubernetes cluster (AKS)  The platform total resource allocation comes to 26 cores and 50GB of RAM. Therefore our official recommendation is as follows:

Node Pool: Optune \- Main Platform

* Purpose: Running the platform core components.  
* Num of nodes: (min) 2  
* Suggested VM type: Standard\_D16ds\_v5  
* Node Storage size: 100GB  
* Provisioning type: On-Demand\*  
  \*Spot instances may be also considerable, as long as it won’t affect the platform availability during working time.

Node Pool: gpu-basic (Playground inference)

* Purpose: Summarization / generative Playground vLLM (full 24 GB A10, matches AWS g5.2xlarge).  
* SKU: `Standard_NV36ads_A10_v5`  
* Karpenter label: `workload: gpu-basic`

Node Pool: gpu-low-performance (class / NER training and evaluation)

* Purpose: Classification and NER GPU training/eval. Cheaper A10 slices or T4 so these jobs are not forced onto NV36.  
* SKUs: `Standard_NV6ads_A10_v5`, `Standard_NV12ads_A10_v5` (optional `Standard_NV18ads_A10_v5`), plus T4 (`Standard_NC4as_T4_v3` / `NC8as` / `NC16as`)  
* Karpenter label: `workload: gpu-low-performance`

Node Pool: gpu-high-performance

* Purpose: Generative / high-VRAM training.  
* SKU GPU Name: L40S

Node Pool: cpu-training & cpu-inference

* Purpose: Fallback for GPU pools in case of quota/budget concerns.  
* SKU Family: D

AKS required features:

- Auto provisioning feature (such as NAP/Karpenter), in order to apply nodepool manifest.  
  Alternatively, you may skip node pools creation using helm values configuration.

Azure federated identity:  
Federated identity is required to bind an Azure identity with kubernetes service account.  
The identity should be granted with the following roles:

| Role name | Resource Name (Scope) |
| :---- | :---- |
| Managed Identity Operator | Resource group |
| Storage Blob Data Contributor | Resource group / Storage Account |
| Virtual Machine Contributor | Resource group |
| Network contributor | Resource group |
| Azure Event Hubs Data Owner | EventHub Namespace |
| Contributor | AKS cluster |

### Secrets

The platform relies on secret for several cases, such as SSO configuration, services authentication (Kafka, MongoDB), SSL configuration, etc.

SpecificAI helm chart support injecting such values using its values file. However, the customer may like to create the secrets on his own, skipping secrets creation by the chart installation.

Here are the list of secrets and expected keys.

**Blob storage is not in this list.** Object storage authenticates through the federated managed identity, so there is no storage account name/key secret to create or rotate.

| Secret Name | Keys | Purpose |
| :---- | :---- | :---- |
| docker-ro-creds | .dockerconfigjson | DockerHub credentials for pulling the platform images from SpecificAI private DockerHub Registry.<br><br>This secret will be provided by SpecificAI Devops prior the installation.<br><br>This secret is set as deployments \- *imagePullSecrets* |
| optune-secrets | DB\_CONNECTION\_STRING KAFKA\_PASSWORD | Used by the pod to authenticate third parties such as Kafka and MongoDB.<br><br>Both should contain valid connectionString. |
| optune-auth-secrets | WEB\_APP\_AZURE\_CLIENT\_ID WEB\_APP\_AZURE\_CLIENT\_SECRET WEB\_APP\_AZURE\_TENANT\_ID WEB\_APP\_AZURE\_REDEIRECT\_URI | This configuration is required for setting up SSO configuration based on Azure Web App registration. ([docs](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)) |
| coralogix-keys | PRIVATE\_KEY | SpecificAI helm chart contains with installation of coralogix opentelemetry agent. Therefore, private api key is required to report metadata logs to SpecificAI. |

### Egresses

SpecificAI platform uses third parties applications to ensure proper functionality. Hence, in order to track how the platform takes place, we used the following providers:

*Note*  
Any data sent by our platform to any third party is metadata only. No PII or any sensitive data will be reported, and will remain as a private asset hosted on your end.

| Provider | Endpoint | Purpose |
| :---- | :---- | :---- |
| Coralogix | eu2.coralogix.com | Logs monitoring. |
| Weights & Biases | api.wandb.ai | Model Training metrics and insights. |
| FullStory | \*.fullstory.com | Browser session recorder.<br><br>Following the previous note, sensitive data is masked hence won’t be delivered either to fullstory or SpecificAI. |

