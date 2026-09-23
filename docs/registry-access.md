# SpecificAI Platform — Registry access

You need exactly one credential from SpecificAI to install the platform: a
read-only **Docker token** for the platform's container images.

| What | Where it lives | Credential |
|---|---|---|
| Helm chart | Public GitHub Container Registry: `oci://ghcr.io/marketplace-specificai/specificai-platform` | None — anonymous pull |
| Container images | SpecificAI's private Docker registry | The Docker token SpecificAI issues you |

The same applies on AWS, Azure, and GCP. No cloud-account access, AWS keys,
or registry login are involved in getting the chart.

## Getting your Docker token

| You send | You receive |
|---|---|
| A request for platform access, through your SpecificAI contact or `devops@specific.ai` | A Docker username and a read-only access token, delivered over a secure channel agreed with your contact. |

The token can only pull the platform's images; it cannot push or read
anything else.

## Using it

The [install guide](install.md#3-add-your-docker-token) turns the username
and token into the `docker-ro-creds` image pull Secret the chart references
from every workload. If a pod reports `ImagePullBackOff`, the token is
missing, mistyped, or revoked.

## Rotating or revoking it

To rotate the token, or to revoke it when it is no longer needed, contact
`devops@specific.ai`. After rotation, regenerate `secrets.values.yaml` with the
new token as in the install guide and rerun your `helm upgrade --install`
command.
