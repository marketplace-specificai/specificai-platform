## Secrets

The platform reads its credentials from four Kubernetes Secrets: registry
access, service credentials, single sign-on, and the observability key.

The Helm chart creates all four for you from values you pass at install time.
If your organization manages Secrets outside Helm, create them yourself with
the names and keys below and turn off the chart's own copies, as described
under **Creating the Secrets yourself**. The names are chart defaults that can
be changed, and each row names the value that renames it; only
`coralogix-keys` has a fixed name.

| Secret | Keys | Purpose |
|---|---|---|
| `docker-ro-creds` | `.dockerconfigjson` | Registry credentials that every platform workload, including the backend and the batch jobs, uses as its `imagePullSecrets`. Renamed by `global.imagePullSecretsName`.<br><br>Holds the Docker token SpecificAI issues you. See the registry note below. |
| `specificai-secrets` | `DB_CONNECTION_STRING`<br>`RABBITMQ_PASSWORD`<br>`KAFKA_PASSWORD`<br>`SUPERVISOR_ADMIN_TOKEN`<br>`KAGGLE_API_TOKEN` | Service credentials, mounted into every pod with `envFrom`. Renamed by `global.envFromSecret`.<br><br>Supply `DB_CONNECTION_STRING` and the password for whichever message broker you run. See the key notes below. |
| `specificai-auth-secret` | Depends on your identity provider | Single sign-on credentials. Renamed by `global.optuneAuthSecretName`. See the table below. |
| `coralogix-keys` | `PRIVATE_KEY` | Coralogix send-your-data key. The chart installs a Coralogix OpenTelemetry agent that reports platform telemetry to SpecificAI; the key is issued to you by SpecificAI. The agent always reads a Secret with this name. |

### Secret values

--8<-- "_snippets/secret-values.md"

### Registry credentials

`docker-ro-creds` holds the read-only Docker token SpecificAI issues you for
the platform's container images, which are stored in SpecificAI's private
Docker Hub repository `specificai/platform`. Every workload references it as
its `imagePullSecrets`, and model image builds also use it to pull their base
image. The chart creates it from `common.secrets.dockerConfigJson`; the
install guide shows how to generate that value from your username and token.
When SpecificAI sends you a new token, update this Secret in the same
`helm upgrade` — see the registry access page. The Helm chart itself is
public and needs no credential.

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

### Creating the Secrets yourself

With `global.createSecret` set to `false`, the chart creates none of
`docker-ro-creds`, `specificai-secrets` or `specificai-auth-secret`. Create
all three in the platform namespace before you install. The backend does not
start without `specificai-auth-secret`, so create it even if you do not use
single sign-on; it can be empty.

`global.createSecret` does not cover `coralogix-keys`. Also set
`common.coralogix.secret.create` to `false`, and create the Secret under
exactly that name with the key `PRIVATE_KEY`: the Coralogix agent always reads
`coralogix-keys`, whatever `common.coralogix.secret.secretName` is set to.

To keep the RabbitMQ password out of your values file as well, point the
broker and KEDA at `specificai-secrets`: set
`rabbitmq.auth.existingPasswordSecret` to its name,
`rabbitmq.auth.existingSecretPasswordKey` to `RABBITMQ_PASSWORD`, and
`global.kedaRabbitmqAuth.existingSecret` to its name. The chart derives the
broker's Erlang cookie from that password only when it is passed as a value,
so in this setup it cannot derive one. Create a Secret once with a random
alphanumeric `rabbitmq-erlang-cookie` key, for example the output of
`openssl rand -hex 32`, and set `rabbitmq.auth.existingErlangSecret` to its
name. Any name works except `<release>-rabbitmq-erlang-cookie`, which the
chart uses for its own.
