# Feature availability

The platform's features, the chart version each first shipped in,
and its cloud availability — generated from the release catalog for
platform version **4.10.2**.

**Clouds** reads as follows:

- **All clouds** — available on AWS, Azure, and GCP.
- **Partial** — available on a subset of the supported clouds; ask in
  your [support channel](support.md) whether yours is covered.
- **On request** — contact us through your [support channel](support.md).

A version older than the [changelog](changelog.md)'s first entry
means the feature predates the current platform generation and is
available everywhere the platform installs.

## Training and evaluation

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Classification task type | 3.9.1 | All clouds |
| NER task type | 3.9.1 | All clouds |
| Summarization task type | 3.9.1 | All clouds |
| Generative base model selection | 4.9.0 | Partial |
| Subtasks | 3.9.1 | All clouds |
| ↳ Smart stratification for multi-task datasets | 3.9.1 | All clouds |
| ↳ Multiple tasks in single model | 3.9.1 | All clouds |
| Improved classification | 3.9.1 | All clouds |
| Per-label confidence threshold tuning | 3.9.1 | All clouds |
| Per-entity-type confidence threshold tuning | 4.0.0 | All clouds |
| CPU and GPU training and evaluation | 3.9.1 | All clouds |
| Default training hardware from account settings | 3.9.1 | All clouds |
| Training GPU SKUs by task | 3.9.1 | All clouds |
| ↳ Encoder training GPUs | 3.9.1 | All clouds |
| ↳ Decoder training GPUs | 3.9.1 | All clouds |
| Evaluation GPU SKUs by task | 3.9.1 | All clouds |
| ↳ Encoder evaluation GPUs | 3.9.1 | All clouds |
| ↳ Decoder evaluation GPUs | 3.9.1 | All clouds |
| Re-evaluation after data changes | 3.9.1 | All clouds |
| Serving-performance evaluation | 4.1.2 | Partial |
| Versions Diff modal | 4.1.1 | All clouds |

## Data preparation

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Auto-labeling unlabeled datasets | 3.9.1 | All clouds |
| Data generation | 3.9.1 | All clouds |
| Retrieval Agent | 4.0.0 | All clouds |
| Online data search (Hugging Face) | 3.9.1 | All clouds |
| Manual upload (CSV, JSON, JSONL, Parquet) | 3.9.1 | All clouds |
| Link existing datasets across tasks | 3.9.1 | All clouds |
| Task data export | 3.9.1 | All clouds |
| Task samples archiving | 3.9.1 | All clouds |
| Auto-split benchmark from a single dataset | 3.9.1 | All clouds |

## Exploration

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Playground for encoder models | 3.9.1 | All clouds |
| Playground for testing generation models | 3.9.1 | All clouds |
| Playground GPU inference (summarization / vLLM) | 3.9.1 | All clouds |

## Inference

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Deploy encoder models on Triton | 3.9.1 | All clouds |
| Model export after training | 3.9.1 | All clouds |
| Inference tracing (SDK request and response) | 3.9.1 | All clouds |

## Model providers

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Native third-party providers | 3.9.1 | All clouds |
| ↳ OpenAI | 3.9.1 | All clouds |
| ↳ Google | 3.9.1 | All clouds |
| ↳ Anthropic | 3.9.1 | All clouds |
| Cloud provider model services | 3.9.1 | All clouds |
| ↳ Amazon Bedrock | 3.9.1 | Partial |
| ↳ Google Vertex AI | 3.9.1 | Partial |
| ↳ Azure AI (OpenAI project and Foundry project) | 3.9.1 | Partial |

## Infrastructure

Since **3.9.1** · Clouds: **All clouds**

| Feature | Since | Clouds |
|---|---|---|
| Coralogix logging | 3.9.1 | All clouds |
| FullStory browser analytics | 3.9.1 | All clouds |
| W&B model training analytics | 3.9.1 | All clouds |
| Okta OIDC login | 4.2.0 | Partial |
| Arena customer clients vs internal environments | 4.8.1 | All clouds |
