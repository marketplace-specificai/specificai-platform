# SpecificAI Platform — Registry access

The platform Helm chart and its container images are distributed through the
AWS Marketplace container registry:

```text
709825985650.dkr.ecr.us-east-1.amazonaws.com/specific-ai/specificai-platform
```

That registry is an Amazon ECR regardless of which cloud your platform runs
on — AWS, Azure, and GCP installations all pull the same chart from it. What
changes per cloud is how you get access: AWS customers are granted access at
the account level, while Azure and GCP customers are issued credentials.

This page describes both exchanges — what you send SpecificAI, what you
receive back, and how to log in with it. Once you can log in, continue with
the [install guide](install.md).

## AWS customers

Access is granted to your AWS account as a whole, so no credentials change
hands.

| You send | You receive |
|---|---|
| Your 12-digit AWS account ID, to `devops@specific.ai` | Confirmation that your account has pull access. |

SpecificAI adds your account ID to the registry policy's `AllowCustomerPull`
statement. That statement grants exactly three read-only actions —
`ecr:BatchCheckLayerAvailability`, `ecr:BatchGetImage`, and
`ecr:GetDownloadUrlForLayer` — so your account can pull the chart and images
and nothing else.

Once your account is added, any IAM principal in it that is permitted to call
ECR can authenticate with your own AWS credentials; SpecificAI issues nothing
further.

## Azure and GCP customers

The registry is an Amazon ECR even when nothing else in your deployment
touches AWS, so SpecificAI issues you a dedicated AWS access key to
authenticate against it. Nothing else in the platform uses this key.

| You send | You receive |
|---|---|
| A request for registry access, through your SpecificAI contact or `devops@specific.ai` | An AWS access key ID and secret access key with read-only pull access to the registry, delivered over a secure channel agreed with your contact. |

The key is scoped to pulling from this registry and cannot read or change
anything else. To rotate the key, or to revoke it when it is no longer
needed, contact `devops@specific.ai`.

## Log in with your access

Logging in requires the AWS CLI and Helm 3.8 or later. Azure and GCP
customers first make the issued key visible to the AWS CLI:

```bash
export AWS_ACCESS_KEY_ID=<your issued access key ID>
export AWS_SECRET_ACCESS_KEY=<your issued secret access key>
```

AWS customers skip that step and use whatever AWS credentials they normally
work with.

Then log Helm in to the registry:

```bash
aws ecr get-login-password --region us-east-1 \
  | helm registry login 709825985650.dkr.ecr.us-east-1.amazonaws.com \
      --username AWS --password-stdin
```

The same password also works for `docker login` with username `AWS`, if you
want to inspect images directly. ECR authorization tokens expire after
12 hours; rerun the command to get a fresh one.

If a pull is denied after login, the usual causes are an account ID not yet
added to the registry policy, or a region other than `us-east-1` in the
`get-login-password` call. Write to `devops@specific.ai` if access still
fails.
