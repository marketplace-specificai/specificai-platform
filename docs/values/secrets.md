# Secrets values

`secrets.values.example.yaml` lists every credential the chart reads, for all cluster
types. Download the raw file from
[`values/secrets.values.example.yaml`](https://github.com/marketplace-specificai/specificai-platform/blob/main/values/secrets.values.example.yaml),
save it as `secrets.values.yaml`, fill in the `<PLACEHOLDER>` values it marks,
and pass it to `helm upgrade --install` after your cluster's values file, as
described in the [install guide](../install.md). Keep the filled copy out of
version control.

```yaml
# DO NOT EDIT in the public repository. This file is generated from
# SpecificAI's private chart sources by
# tools/marketplace_mirror/transform_content.py and republished on every
# chart release. Copy it, fill in the placeholder values it marks, and keep
# the filled copy out of version control: it holds credentials.

# ==============================================================================
# Secrets for the SpecificAI platform chart
# ==============================================================================
# Copy this file to secrets.values.yaml, replace every <YOUR_*> placeholder,
# and pass it to helm next to your cluster's values file:
#
#   helm upgrade --install ... --values <cluster>.values.yaml --values secrets.values.yaml
#
# Keep secrets.values.yaml out of version control. Commented keys are only
# needed for the features their description names; uncomment the ones you use.
# Values that must match are marked with the same placeholder.

# global:
#   ingress:
#     certificateTLSCrt: ""
#     certificateTLSKey: ""
#   gateway:
#     tls:
#       certificateTLSCrt: ""
#       certificateTLSKey: ""
#   rabbitmqPassword: ""

common:
  secrets:
    # -- Base64-encoded Docker Hub .dockerconfigjson.
    dockerConfigJson: "<YOUR_DOCKER_CONFIG_JSON>"
    # -- Database connection string (DocumentDB / Cosmos DB / MongoDB Atlas).
    dbConnectionString: "<YOUR_MONGODB_CONNECTION_STRING>"
    # -- Kafka SASL password.
    # kafkaPassword: ""
    # -- RabbitMQ password.
    rabbitmqPassword: "<YOUR_RABBITMQ_PASSWORD>"
    # -- Kaggle API token for agents_handler public-dataset search/download. Optional: the agents_handler image already ships SpecificAI's token, and this value only needs setting to override it with your own Kaggle account.
    # kaggleApiToken: ""
    # -- Token guarding the Playground inference supervisor's admin endpoints. Leave empty: the chart generates one on first install and keeps it across upgrades. Set it only to pin a value.
    # supervisorAdminToken: ""

  # -- SSO configuration. Set method and the matching WEB_APP_* keys for that IDP. BOTH is Google+Azure only and never includes Okta.
  # optune_auth:
  #   web_app_okta_client_secret: ""
  #   web_app_google_client_secret: ""
  #   web_app_azure_client_secret: ""

  # -- Coralogix app-side secret configuration.
  coralogix:
    secret:
      # -- Coralogix private key (send logs/metrics).
      privateKey: "<YOUR_CORALOGIX_PRIVATE_KEY>"

# backend:
#   storage:
#     blob:
#       accountKey: ""

rabbitmq:
  auth:
    # -- Password of the in-cluster RabbitMQ user. Must match `common.secrets.rabbitmqPassword`.
    password: "<YOUR_RABBITMQ_PASSWORD>"
    # -- Erlang cookie of the in-cluster RabbitMQ. Leave empty to let the RabbitMQ chart generate one.
    # erlangCookie: ""

# mongodb:
#   auth:
#     rootPassword: ""
#     passwords:
#       - ""
```
