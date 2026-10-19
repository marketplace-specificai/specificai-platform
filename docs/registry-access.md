# SpecificAI Platform — Registry access

You need exactly one credential from SpecificAI to install the platform: a
read-only **Docker token** for the platform's container images.

| What | Where it lives | Credential |
|---|---|---|
| Helm chart | Public GitHub Container Registry: `oci://ghcr.io/marketplace-specificai/specificai-platform` | None — anonymous pull |
| Container images | SpecificAI's private Docker Hub repository `specificai/platform` (`docker.io`) | The Docker token SpecificAI issues you |

The same applies on AWS, Azure, and GCP. No cloud-account access, AWS keys,
or registry login are involved in getting the chart.

Chart versions before 4.13.0 are available on request from
[SpecificAI support](support.md).

## Getting your Docker token

| You send | You receive |
|---|---|
| A request for platform access, through your SpecificAI contact or `devops@specific.ai` | A Docker username and a read-only access token, delivered over a secure channel agreed with your contact. |

The token can only pull the platform's images; it cannot push or read
anything else.

## Using it

The [install guide](install.md#3-add-your-docker-token) turns the username
and token into `common.secrets.dockerConfigJson`, from which the chart
creates the `docker-ro-creds` image pull Secret. Every platform workload
references that Secret, including the Jobs the platform starts for training,
evaluation and model image builds. If a pod reports `ImagePullBackOff`, the
token is missing, mistyped, or revoked.

If you manage Secrets outside Helm (`global.createSecret: false`), create a
`kubernetes.io/dockerconfigjson` Secret with the name set in
`global.imagePullSecretsName` (default `docker-ro-creds`) in the platform
namespace instead.

## Rotating or revoking it

To rotate the token, or to revoke it when it is no longer needed, contact
`devops@specific.ai`. SpecificAI may also send you a new token, together with
the chart version to upgrade to.

When you receive a new token:

1. Regenerate `secrets.values.yaml` with it, as in
   [step 3 of the install guide](install.md#3-add-your-docker-token).
2. Run your `helm upgrade --install` command with the updated file. When the
   token arrives with a new chart version, pass both in the **same**
   upgrade — the new version's images are only readable with the new token.

If you manage `docker-ro-creds` yourself, replace its `.dockerconfigjson`
before upgrading instead. Pods that are already running keep their images;
only new pulls use the new token.
