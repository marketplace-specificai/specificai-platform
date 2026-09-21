## Outbound endpoints

The platform calls a small number of third-party services to report how it is
running. If your cluster restricts egress, allow the destinations below.

Everything sent to these providers is metadata about platform operation.
**No personally identifiable information, no dataset content, and no model
weights leave your environment** — your data and your trained models stay in
the object storage and database you provisioned.

| Provider | Endpoint | Purpose |
|---|---|---|
| Coralogix | `eu2.coralogix.com` | Platform log and metric monitoring. |
| Weights & Biases | `api.wandb.ai` | Model training metrics and insights. |
| FullStory | `*.fullstory.com` | Browser session recording in the web interface. Sensitive fields are masked in the browser, so they reach neither FullStory nor SpecificAI. |

These are the only outbound destinations the platform itself requires. Your
cluster will additionally need normal egress to your cloud provider's APIs, to
the container registry the platform images are pulled from, and to any model
weights source you configure.
