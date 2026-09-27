## Secrets

The platform reads its credentials from four Kubernetes Secrets: registry
access, service credentials, single sign-on, and the observability key.

The Helm chart creates all four for you from values you pass at install time.
If your organization manages Secrets outside Helm, set `global.createSecret` to
`false` and create them yourself with the names and keys below. The names are
chart defaults and can be changed — each row names the value that renames it.

| Secret | Keys | Purpose |
|---|---|---|
| `docker-ro-creds` | `.dockerconfigjson` | Registry credentials used as `imagePullSecrets` by every platform workload. Renamed by `global.imagePullSecretsName`.<br><br>See the registry note below — this is not always needed. |
| `specificai-secrets` | `DB_CONNECTION_STRING`<br>`RABBITMQ_PASSWORD`<br>`KAFKA_PASSWORD`<br>`SUPERVISOR_ADMIN_TOKEN`<br>`KAGGLE_API_TOKEN` | Service credentials, mounted into every pod with `envFrom`. Renamed by `global.envFromSecret`.<br><br>Supply `DB_CONNECTION_STRING` and the password for whichever message broker you run. See the key notes below. |
| `specificai-auth-secret` | Depends on your identity provider | Single sign-on credentials. Renamed by `global.optuneAuthSecretName`. See the table below. |
| `coralogix-keys` | `PRIVATE_KEY` | Coralogix send-your-data key. The chart installs a Coralogix OpenTelemetry agent that reports platform telemetry to SpecificAI; the key is issued to you by SpecificAI. |

### Registry credentials

`docker-ro-creds` holds credentials for the registry your platform images are
pulled from, and which registry that is depends on how you obtained the chart.

The standard distribution pulls images from SpecificAI's Docker Hub
organization and needs a read-only Docker Hub credential, which SpecificAI
DevOps issues before installation. The **AWS Marketplace** distribution is
published with its image references already rewritten to the marketplace
container registry, and access to that registry is granted to your cloud
account rather than to a username and password.

Every workload still carries the `imagePullSecrets` reference either way, so
keep the Secret name configured; only whether it holds a credential changes.

### Key notes

- `DB_CONNECTION_STRING` is the full MongoDB connection string for the database
  described under **Cloud services**, including credentials.
- `RABBITMQ_PASSWORD` is required on the default in-cluster broker. It must
  match `rabbitmq.auth.password`, which the chart sets from the same install
  value.
- `KAFKA_PASSWORD` is required **only** if you switch the broker to managed
  Kafka with `common.env.data.MESSAGE_BROKER_TYPE: kafka`. On the default
  in-cluster broker, omit it.
- `SUPERVISOR_ADMIN_TOKEN` guards the Playground inference supervisor's admin
  endpoints. The chart generates one on first install and preserves it across
  upgrades, so you only supply it when you create this Secret yourself — any
  random 40-character string works, and the same value must be visible to every
  pod.
- `KAGGLE_API_TOKEN` is optional. It overrides the built-in token the platform
  uses to search and download public Kaggle datasets.

### Single sign-on keys

The identity provider is independent of your cloud — any of the three works on
AWS, Azure, and GCP. Populate the keys for the one you use and leave the others
unset.

| Identity provider | Keys |
|---|---|
| Google | `WEB_APP_GOOGLE_CLIENT_ID`<br>`WEB_APP_GOOGLE_CLIENT_SECRET`<br>`WEB_APP_GOOGLE_REDIRECT_URI` |
| Microsoft Entra ID | `WEB_APP_AZURE_CLIENT_ID`<br>`WEB_APP_AZURE_CLIENT_SECRET`<br>`WEB_APP_AZURE_REDIRECT_URI`<br>`WEB_APP_AZURE_TENANT_ID` |
| Okta | `WEB_APP_OKTA_CLIENT_ID`<br>`WEB_APP_OKTA_CLIENT_SECRET`<br>`WEB_APP_OKTA_REDIRECT_URI`<br>`WEB_APP_OKTA_ISSUER` |

The redirect URI is `https://<your platform hostname>/api/auth/<provider>/callback`,
using the hostname you set as `global.optuneAddress`. Register that exact URI
with your identity provider.
