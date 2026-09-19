# **![][image1]**

# **AWS Requirements for SpecificAI Auto Distillation Platform Deployment**

SpecificAI Platform is a self-hosted kubernetes native application for creating task-specific models.  
SpecificAI is meant to be running on a cloud provider environment, such as AWS / Azure/ GCP.

This document outlines the essential Amazon Web Services (AWS) requirements necessary for the successful deployment and operation of the SpecificAI auto distillation platform on a customer's infrastructure. The implementation is intended to be performed using Terraform, and the solution itself is deployed via Helm.

# **Infrastructure and Compute Requirements**

The SpecificAI Inference solution requires dedicated compute resources to ensure isolation and performance.

## **Platform compute requirements**

* CPU: 26vCPU  
* RAM: 50GB

## **Amazon Elastic Kubernetes Service (EKS)**

The solution will be deployed onto an existing or new Amazon EKS cluster within the customer's AWS account.

| Requirement | Details | Rationale |
| :---- | :---- | :---- |
| Dedicated Nodegroup | A single dedicated nodegroup is required for isolation purposes. | Customer requirement for strict resource isolation. |
| Instance Type | `m7i.8xlarge` | Specified host type for performance and compatibility. |
| Minimum Node Count | 2 | Single host deployment as per requirement. |

## **Kubernetes Node Pools**

The up-mentioned nodegroup is dedicated for running the auto distillation platform core, but it could, in case preferred, be set as a node pool.

Model training itself will be performed based on the GPU resources using GPU cards variations \- 

| AWS Instance family | GPU Card | Node Pool Name |
| :---- | :---- | :---- |
| g6e | `L40S` | gpu-high-performance |
| g5 | `A10G` | gpu-basic |
| g4dn | `T4` | gpu-low-performance |
| c6i |  | cpu-basic |

GPU Instance won’t be running constantly while the platform is online, but will be running On Demand, as training takes place, therefore, no GPU massive cost is expected.

## **Networking**

The EKS cluster and its nodegroup must have the necessary network configuration for external and internal communication (e.g., access to S3, MSK, DocumentDB, etc.).

# **AWS Service Requirements**

The following AWS services and resources are required to support the SpecificAI Inference solution's functionality.

## **Amazon Simple Storage Service (S3)**

An S3 bucket is necessary for hosting the core inference models used by the solution.

* **S3 Bucket:** A dedicated S3 bucket must be created for model hosting.  
  1.  **This bucket must include a dedicated directory for the models to be served (called deployed\_models), as the inference solution uses this as the repository path for serving models.**  
  2. **The bucket should enable CORS (required for upload and download large files from the platform, using sign-url), with the following configuration**

```
# CORS on models bucket
cors_rule {
  allowed_headers = ["*"]
  allowed_methods = ["PUT"]
  allowed_origins = [<frontend_url_or_https_domain>]
  expose_headers  = ["ETag", "Content-Type"]
  max_age_seconds = 3000
}
```

     **C. The bucket should include the following lifecycle rule**

```
# Lifecycle cleanup for temporary downloads
rule {
  id     = "expire-downloads"
  status = "Enabled"
  filter { prefix = "downloads/" }
  expiration { days = 1 }
  noncurrent_version_expiration { noncurrent_days = 1 }
  abort_incomplete_multipart_upload { days_after_initiation = 1 }
}
```

* **Access Policy:** The EKS Worker Node IAM Role or the Kubernetes Service Account (via IAM Roles for Service Accounts \- IRSA) must have the following permissions \-


```
# Required S3 actions for upload/download flow
actions = [
  "s3:GetObject",
  "s3:PutObject",
  "s3:DeleteObject",
  "s3:AbortMultipartUpload",
  "s3:ListMultipartUploadParts"
]
resources = ["arn:aws:s3:::specificai-*/*"]

actions   = ["s3:ListBucket"]
resources = ["arn:aws:s3:::specificai-*"]
```

## **MongoDB (AWS DocumentDB/MongoDBAtlas)**

The platform persistent data is stored based on MongoDB Compatible solution, either DocumentDB or MongoDBAtlas

Hereby, this is the recommended compute solution to serve the platform \- 

| Solution | Compute size/tier |
| :---- | :---- |
| MongoDB Atlas | `M40` |
| AWS DocumentDB | `db.r5.xlarge` |

# **Deployment Requirements**

The deployment process relies on specific tooling and authentication methods.

## **Helm**

The SpecificAI Inference solution is distributed as a Helm chart.

* **Helm Installation:** Helm must be installed on the deployment environment.  
* **AWS Credentials for Helm:** The Helm installation process requires trust policy or as fallback a valid pair of AWS credentials (Access Key ID and Secret Access Key) to facilitate any required AWS interactions during the chart deployment (e.g., dynamic resource creation, integration with cloud services).

	The credentials will be given by SpecificAI DevOps

* **Helm Chart OCI: 709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform:X.X.X**

### Secrets

The platform relies on secret for several cases, such as SSO configuration, services authentication (Kafka, MongoDB), SSL configuration, etc.

SpecificAI helm chart support injecting such values using its values file. However, the customer may like to create the secrets on his own, skipping secrets creation by the chart installation.

Here are the list of secrets and expected keys

| Secret Name | Key | Purpose |
| :---- | :---- | :---- |
| docker-ro-creds | .dockerconfigjson | DockerHub credentials for pulling the platform images from SpecificAI private DockerHub Registry.<br><br>This secret will be provided by SpecificAI Devops prior the installation.<br><br>This secret is set as deployments \- imagePullSecrets |
| specificai-secrets | DB\_CONNECTION\_STRING KAFKA\_PASSWORD | Used by the pod to authenticate third parties such as Kafka and MongoDB.<br><br>Both should contain valid connectionString. |
| specificai-auth-secrets | WEB\_APP\_GOOGLE\_CLIENT\_ID WEB\_APP\_GOOGLE\_CLIENT\_SECRET WEB\_APP\_GOOGLE\_REDEIRECT\_URI | This configuration is required for setting up SSO configuration based on Azure Web App registration. ([docs](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)) |
| coralogix-keys | PRIVATE\_KEY | SpecificAI helm chart contains with installation of coralogix opentelemetry agent. Therefore, private api key is required to report metadata logs to SpecificAI. |

## 

### Egresses

SpecificAI platform uses third parties applications to ensure proper functionality. Hence, in order to track how the platform takes place, we used the following providers:

*Note*  
Any data sent by our platform to any third party is metadata only. No PII or any sensitive data will be reported, and will remain as a private asset hosted on your end.

| Provider | Endpoint | Purpose |
| :---- | :---- | :---- |
| Coralogix | eu2.coralogix.com | Logs monitoring. |
| Weights & Biases | api.wandb.ai | Model Training metrics and insights. |
| FullStory | \*.fullstory.com | Browser session recorder.<br><br>Following the previous note, sensitive data is masked hence won’t be delivered either to fullstory or SpecificAI. |

[image1]: assets/specificai-logo.png