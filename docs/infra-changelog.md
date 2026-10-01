# Infrastructure changelog

The infrastructure work each platform version needs before you upgrade to it,
step by step for every affected cloud. A version marked "No infrastructure change."
needs nothing beyond the standard [upgrade procedure](upgrade.md). When you
skip versions, apply every entry between your current version and the target,
oldest first. The [changelog](changelog.md) lists everything else each version
changes.

## 4.13.0 — 2026-10-01

### What changes

From 4.13.0 the platform is distributed from new locations, and the chart
checks your values more strictly.

- **Container images** come from SpecificAI's private Docker Hub repository
  `specificai/platform`. The cluster pulls them with the `docker-ro-creds`
  image pull Secret, which must now hold the read-only Docker token that
  SpecificAI issues you (see [Registry access](registry-access.md)). The
  credentials you used with the previous registry do not work here.
- **The Helm chart** comes from the public GitHub Container Registry,
  `oci://ghcr.io/marketplace-specificai/specificai-platform`, and needs no
  login.
- **The values schema** now fails `helm upgrade` when `global.cloudProvider`,
  `global.cloudRegion`, `global.bucketName`, `global.roleId` or
  `global.optuneAddress` still holds a `<...>` placeholder, when
  `global.bucketName` or `global.optuneAddress` is empty, or when
  `global.roleId` is empty on Azure or GCP. The check runs before anything
  reaches the cluster, so fix the value and rerun. Values files that started
  from a template published before 4.13.0 should also go through
  [Moving to the generated values templates](upgrade.md#moving-to-the-generated-values-templates).

Every install is affected. Put the Docker token in the **same** `helm upgrade`
that installs 4.13.0, because this version's images can only be read with it.
If you skip it, every pod the upgrade replaces fails with `ImagePullBackOff`,
and so do the training and evaluation Jobs the platform starts afterwards.

If the cluster's outbound traffic is limited to an allowlist of domains, the
nodes also need HTTPS (TCP 443) to Docker Hub:

- `registry-1.docker.io`
- `auth.docker.io`
- `production.cloudflare.docker.com`
- `production.cloudfront.docker.com`

The machine that runs `helm` needs HTTPS to `ghcr.io` and
`pkg-containers.githubusercontent.com`, where the chart is downloaded from.
Clusters with unrestricted outbound HTTPS need no network change.

The steps assume the release name `specificai` and namespace `specificai`
from the [install guide](install.md).

=== "AWS"

    #### Console steps

    1. **Egress allowlist (only if you filter outbound domains).** If you
       filter with AWS Network Firewall, open **VPC > Network Firewall >
       Network Firewall rule groups** and select the stateful domain-list rule
       group that allows your cluster's outbound traffic. Edit its domain
       list, add the four Docker Hub hostnames above for HTTPS (TLS SNI), and
       save. If you filter with a proxy or another firewall, add the same
       hostnames there.
    2. **Required values.** Confirm your values file holds real values for
       these keys:
        - `global.cloudRegion`: the cluster's Region, shown on the **Amazon
          EKS > Clusters** page, for example `us-west-2`.
        - `global.bucketName`: the bucket name from **Amazon S3 > Buckets**.
        - `global.roleId`: the ARN from **IAM > Roles > your platform role**.
          Leave it empty only if the platform uses the node role or an EKS Pod
          Identity association.
        - `global.optuneAddress`: the hostname users open the platform on.
    3. **Docker token.** There is no console step for this. Build the pull
       Secret with the CLI below.

    #### CLI

    Check that the cluster can reach Docker Hub. A `401` means it can (the
    registry answers and asks for credentials). A timeout, or a pod that never
    starts because its own image cannot be pulled, means outbound access is
    blocked.

    ```bash
    NAMESPACE=specificai
    kubectl run dockerhub-check -n "$NAMESPACE" --rm -i --restart=Never \
      --image=curlimages/curl -- \
      curl -sS -o /dev/null -w '%{http_code}\n' https://registry-1.docker.io/v2/
    ```

    If you use AWS Network Firewall, check the domains in the rule group:

    ```bash
    REGION=<AWS_REGION>
    aws network-firewall describe-rule-group --rule-group-name <RULE_GROUP_NAME> \
      --type STATEFUL --region "$REGION" \
      --query "RuleGroup.RulesSource.RulesSourceList.Targets"
    ```

    Look up the required values:

    ```bash
    aws eks describe-cluster --name <CLUSTER_NAME> --region "$REGION" --query "cluster.arn" --output text
    aws s3api head-bucket --bucket <BUCKET_NAME>
    aws iam get-role --role-name <PLATFORM_ROLE_NAME> --query "Role.Arn" --output text
    ```

    The cluster ARN's fourth field is the Region, and `head-bucket` prints
    nothing when the bucket exists and you can reach it.

    Build the pull Secret from the Docker username and token SpecificAI sent
    you, as in [step 3 of the install guide](install.md#3-add-your-docker-token).
    The `docker login` check is optional and needs Docker on your machine:

    ```bash
    printf 'Docker username: '; read -r DOCKER_USERNAME
    printf 'Docker token: '; read -r -s DOCKER_TOKEN; echo

    printf '%s' "$DOCKER_TOKEN" | docker login --username "$DOCKER_USERNAME" --password-stdin
    docker logout

    export DOCKER_CONFIG_JSON=$(kubectl create secret docker-registry docker-ro-creds \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o jsonpath='{.data.\.dockerconfigjson}')

    yq -i '.common.secrets.dockerConfigJson = strenv(DOCKER_CONFIG_JSON)' secrets.values.yaml
    ```

    If you have no `secrets.values.yaml` yet, create it first as described in
    [Move credentials to a second file](upgrade.md#move-credentials-to-a-second-file),
    and remove any old `common.secrets.dockerConfigJson` from your values
    file.

    If you manage Secrets yourself (`global.createSecret: false`), replace
    the Secret in the cluster instead, before upgrading:

    ```bash
    kubectl create secret docker-registry docker-ro-creds -n "$NAMESPACE" \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

    Clear the token from your shell, render the chart once to catch schema
    errors without touching the cluster, then upgrade:

    ```bash
    unset DOCKER_TOKEN DOCKER_CONFIG_JSON

    helm template specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml > /dev/null

    helm upgrade --install specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml
    ```

    #### Verify

    ```bash
    kubectl get secret docker-ro-creds -n "$NAMESPACE" \
      -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'

    kubectl get pods -n "$NAMESPACE" \
      -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u

    kubectl get pods -n "$NAMESPACE" | grep -E 'ImagePullBackOff|ErrImagePull' || echo "no image pull errors"
    ```

    The Secret lists `https://index.docker.io/v1/`, which confirms the token
    is present without printing it. The platform's images start with
    `specificai/platform:`. No pod reports an image pull error, and every pod
    is `Running` or `Completed` within a few minutes.

=== "Azure"

    #### Console steps

    1. **Egress allowlist (only if you filter outbound domains).** If the
       cluster's outbound traffic goes through Azure Firewall, open **Firewall
       Policies**, select the policy attached to that firewall, and open
       **Settings > Application rules**. Select **Add a rule collection** of
       type **Application** with action **Allow**, and add a rule whose
       source is the AKS node subnet, protocol `Https:443`, and destination
       type **FQDN** with the four Docker Hub hostnames above. If you filter
       with a proxy or another firewall, add the same hostnames there.
    2. **Required values.** Confirm your values file holds real values for
       these keys:
        - `global.cloudRegion`: the **Location** on the AKS cluster's
          **Overview** page, written as a region name such as
          `southcentralus`.
        - `global.bucketName`: `<STORAGE_ACCOUNT_NAME>/<CONTAINER_NAME>`, from
          **Storage accounts > your account > Data storage > Containers**.
        - `global.roleId`: the **Client ID** on the platform identity's page
          under **Managed Identities**. It is required on Azure.
        - `global.optuneAddress`: the hostname users open the platform on.
    3. **Docker token.** There is no console step for this. Build the pull
       Secret with the CLI below.

    #### CLI

    Check that the cluster can reach Docker Hub. A `401` means it can (the
    registry answers and asks for credentials). A timeout, or a pod that never
    starts because its own image cannot be pulled, means outbound access is
    blocked.

    ```bash
    NAMESPACE=specificai
    kubectl run dockerhub-check -n "$NAMESPACE" --rm -i --restart=Never \
      --image=curlimages/curl -- \
      curl -sS -o /dev/null -w '%{http_code}\n' https://registry-1.docker.io/v2/
    ```

    If you use Azure Firewall, add the application rule. The command comes
    from the `azure-firewall` CLI extension:

    ```bash
    az extension add --name azure-firewall

    az network firewall policy rule-collection-group collection add-filter-collection \
      -g <FIREWALL_RESOURCE_GROUP> --policy-name <FIREWALL_POLICY_NAME> \
      --rule-collection-group-name <RULE_COLLECTION_GROUP_NAME> \
      --name allow-docker-hub --collection-priority <PRIORITY> --action Allow \
      --rule-name docker-hub --rule-type ApplicationRule \
      --source-addresses <AKS_NODE_SUBNET_CIDR> --protocols Https=443 \
      --target-fqdns registry-1.docker.io auth.docker.io production.cloudflare.docker.com production.cloudfront.docker.com
    ```

    Look up the required values:

    ```bash
    az aks show -g <AKS_RESOURCE_GROUP> -n <AKS_CLUSTER_NAME> --query location -o tsv
    az identity show -g <MANAGED_IDENTITY_RESOURCE_GROUP> -n <MANAGED_IDENTITY_NAME> --query clientId -o tsv
    az storage container exists --account-name <STORAGE_ACCOUNT_NAME> --name <CONTAINER_NAME> --auth-mode login
    ```

    Build the pull Secret from the Docker username and token SpecificAI sent
    you, as in [step 3 of the install guide](install.md#3-add-your-docker-token).
    The `docker login` check is optional and needs Docker on your machine:

    ```bash
    printf 'Docker username: '; read -r DOCKER_USERNAME
    printf 'Docker token: '; read -r -s DOCKER_TOKEN; echo

    printf '%s' "$DOCKER_TOKEN" | docker login --username "$DOCKER_USERNAME" --password-stdin
    docker logout

    export DOCKER_CONFIG_JSON=$(kubectl create secret docker-registry docker-ro-creds \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o jsonpath='{.data.\.dockerconfigjson}')

    yq -i '.common.secrets.dockerConfigJson = strenv(DOCKER_CONFIG_JSON)' secrets.values.yaml
    ```

    If you have no `secrets.values.yaml` yet, create it first as described in
    [Move credentials to a second file](upgrade.md#move-credentials-to-a-second-file),
    and remove any old `common.secrets.dockerConfigJson` from your values
    file.

    If you manage Secrets yourself (`global.createSecret: false`), replace
    the Secret in the cluster instead, before upgrading:

    ```bash
    kubectl create secret docker-registry docker-ro-creds -n "$NAMESPACE" \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

    Clear the token from your shell, render the chart once to catch schema
    errors without touching the cluster, then upgrade:

    ```bash
    unset DOCKER_TOKEN DOCKER_CONFIG_JSON

    helm template specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml > /dev/null

    helm upgrade --install specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml
    ```

    #### Verify

    ```bash
    kubectl get secret docker-ro-creds -n "$NAMESPACE" \
      -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'

    kubectl get pods -n "$NAMESPACE" \
      -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u

    kubectl get pods -n "$NAMESPACE" | grep -E 'ImagePullBackOff|ErrImagePull' || echo "no image pull errors"
    ```

    The Secret lists `https://index.docker.io/v1/`, which confirms the token
    is present without printing it. The platform's images start with
    `specificai/platform:`. No pod reports an image pull error, and every pod
    is `Running` or `Completed` within a few minutes.

=== "GCP"

    #### Console steps

    1. **Egress allowlist (only if you filter outbound domains).** If you
       restrict egress with Cloud Next Generation Firewall, open **VPC network
       > Firewall policies** and select the global network firewall policy
       associated with the cluster's VPC. Select **Create firewall rule**,
       with direction **Egress**, action **Allow**, a priority higher than
       your deny rule (a lower number), protocol `tcp:443`, and the four
       Docker Hub hostnames above as **Destination FQDNs**. If you filter
       with a proxy or another firewall, add the same hostnames there.
    2. **Required values.** Confirm your values file holds real values for
       these keys:
        - `global.cloudRegion`: the region in the cluster's **Location** under
          **Kubernetes Engine > Clusters**, such as `us-east1`. For a zonal
          cluster in `us-east1-b`, use `us-east1`.
        - `global.bucketName`: the bucket name from **Cloud Storage >
          Buckets**.
        - `global.roleId`: the platform service account's email from **IAM &
          Admin > Service Accounts**. It is required on GCP.
        - `global.optuneAddress`: the hostname users open the platform on.
    3. **Docker token.** There is no console step for this. Build the pull
       Secret with the CLI below.

    #### CLI

    Check that the cluster can reach Docker Hub. A `401` means it can (the
    registry answers and asks for credentials). A timeout, or a pod that never
    starts because its own image cannot be pulled, means outbound access is
    blocked.

    ```bash
    NAMESPACE=specificai
    kubectl run dockerhub-check -n "$NAMESPACE" --rm -i --restart=Never \
      --image=curlimages/curl -- \
      curl -sS -o /dev/null -w '%{http_code}\n' https://registry-1.docker.io/v2/
    ```

    If you use Cloud NGFW, add the egress rule:

    ```bash
    gcloud compute network-firewall-policies rules create <PRIORITY> \
      --firewall-policy=<FIREWALL_POLICY_NAME> --global-firewall-policy \
      --direction=EGRESS --action=allow --layer4-configs=tcp:443 \
      --dest-fqdns=registry-1.docker.io,auth.docker.io,production.cloudflare.docker.com,production.cloudfront.docker.com \
      --description="SpecificAI platform images from Docker Hub"
    ```

    Look up the required values:

    ```bash
    PROJECT=<GCP_PROJECT_ID>
    gcloud container clusters list --project "$PROJECT" --format="table(name,location)"
    gcloud storage buckets describe gs://<BUCKET_NAME> --format="value(name)"
    gcloud iam service-accounts list --project "$PROJECT" --format="value(email)"
    ```

    Build the pull Secret from the Docker username and token SpecificAI sent
    you, as in [step 3 of the install guide](install.md#3-add-your-docker-token).
    The `docker login` check is optional and needs Docker on your machine:

    ```bash
    printf 'Docker username: '; read -r DOCKER_USERNAME
    printf 'Docker token: '; read -r -s DOCKER_TOKEN; echo

    printf '%s' "$DOCKER_TOKEN" | docker login --username "$DOCKER_USERNAME" --password-stdin
    docker logout

    export DOCKER_CONFIG_JSON=$(kubectl create secret docker-registry docker-ro-creds \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o jsonpath='{.data.\.dockerconfigjson}')

    yq -i '.common.secrets.dockerConfigJson = strenv(DOCKER_CONFIG_JSON)' secrets.values.yaml
    ```

    If you have no `secrets.values.yaml` yet, create it first as described in
    [Move credentials to a second file](upgrade.md#move-credentials-to-a-second-file),
    and remove any old `common.secrets.dockerConfigJson` from your values
    file.

    If you manage Secrets yourself (`global.createSecret: false`), replace
    the Secret in the cluster instead, before upgrading:

    ```bash
    kubectl create secret docker-registry docker-ro-creds -n "$NAMESPACE" \
      --docker-server=https://index.docker.io/v1/ \
      --docker-username="$DOCKER_USERNAME" \
      --docker-password="$DOCKER_TOKEN" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

    Clear the token from your shell, render the chart once to catch schema
    errors without touching the cluster, then upgrade:

    ```bash
    unset DOCKER_TOKEN DOCKER_CONFIG_JSON

    helm template specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml > /dev/null

    helm upgrade --install specificai oci://ghcr.io/marketplace-specificai/specificai-platform \
      --version 4.13.0 --namespace "$NAMESPACE" \
      --values your-values.yaml --values secrets.values.yaml
    ```

    #### Verify

    ```bash
    kubectl get secret docker-ro-creds -n "$NAMESPACE" \
      -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'

    kubectl get pods -n "$NAMESPACE" \
      -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u

    kubectl get pods -n "$NAMESPACE" | grep -E 'ImagePullBackOff|ErrImagePull' || echo "no image pull errors"
    ```

    The Secret lists `https://index.docker.io/v1/`, which confirms the token
    is present without printing it. The platform's images start with
    `specificai/platform:`. No pod reports an image pull error, and every pod
    is `Running` or `Completed` within a few minutes.

## 4.11.1 — 2026-10-01

No infrastructure change.

## 4.11.0 — 2026-09-27

No infrastructure change.

## 4.10.2 — 2026-09-23

No infrastructure change.

## 4.10.0 — 2026-09-22

### What changes

The Playground's GPU inference pod (`specificai-vllm-inference`) now keeps
several base models loaded at once and parks idle ones in host memory, so it
needs much more RAM on its GPU node: the memory request rises from `16Gi` to
`40Gi` and the limit from `32Gi` to `52Gi`. The pod still asks for one GPU on
the `gpu-basic` tier. A `gpu-basic` node with less than about 40 GiB of
allocatable memory can no longer run it. When the node is too small, the pod
stays `Pending` with `Insufficient memory` and Playground sessions never
start. On AWS the chart's NodePool now provisions a `g5.4xlarge` instead of a
`g5.2xlarge` for this pod, which uses twice the vCPU quota.

The release also adds a `SUPERVISOR_ADMIN_TOKEN` key to the platform Secret.
It protects the inference pod's admin endpoints from other pods in the
cluster. The chart generates it when it manages the Secret
(`global.createSecret: true`, the default). If you manage the
`specificai-secrets` Secret yourself, add the key before upgrading. The
upgrade does not fail without it, but those endpoints are then left open.

Only installs with the Playground enabled
(`specificai-vllm-inference.enabled: true`) are affected. If you have not
enabled it, this version needs no infrastructure change.

=== "AWS"

    #### Console steps

    1. Open **Service Quotas > AWS services > Amazon Elastic Compute Cloud
       (Amazon EC2)** in the cluster's Region and find **Running On-Demand G
       and VT instances**. The value is counted in vCPUs. A `g5.4xlarge` uses
       16, so make sure the applied quota covers 16 more vCPUs than your
       training GPU nodes already use. If not, select **Request increase at
       account level**.
    2. If you replaced the chart's NodePools with your own, or an AWS
       Organizations service control policy or IAM condition limits
       `ec2:InstanceType`, allow `g5.4xlarge` (or a larger `g5` size) for the
       `gpu-basic` tier. The chart's own NodePool already allows the whole
       `g5` family, so you need no change there.

    #### CLI

    ```bash
    REGION=<AWS_REGION>

    aws service-quotas get-service-quota --service-code ec2 \
      --quota-code L-DB2E81BA --region "$REGION" \
      --query "Quota.Value"

    # Only if the value is too low:
    aws service-quotas request-service-quota-increase --service-code ec2 \
      --quota-code L-DB2E81BA --desired-value <NEW_VCPU_LIMIT> --region "$REGION"

    aws ec2 describe-instance-type-offerings --location-type availability-zone \
      --filters Name=instance-type,Values=g5.4xlarge --region "$REGION" \
      --query "InstanceTypeOfferings[].Location" --output text
    ```

    The last command lists the Availability Zones in the Region that offer
    `g5.4xlarge`. At least one of them must be a zone your cluster's subnets
    use.

    If you manage the `specificai-secrets` Secret yourself
    (`global.createSecret: false`), add the token before upgrading:

    ```bash
    NAMESPACE=specificai
    kubectl patch secret specificai-secrets -n "$NAMESPACE" --type merge \
      -p "{\"stringData\":{\"SUPERVISOR_ADMIN_TOKEN\":\"$(openssl rand -hex 20)\"}}"
    ```

    #### Verify

    After the upgrade, start a Playground session in the platform UI, then
    check where the inference pod landed:

    ```bash
    kubectl get pods -n "$NAMESPACE" -o wide | grep vllm-inference
    kubectl get nodes -L node.kubernetes.io/instance-type,workload | grep gpu-basic
    kubectl get events -n "$NAMESPACE" --field-selector reason=FailedScheduling
    ```

    The pod reaches `Running` on a node of type `g5.4xlarge` or larger, and
    there are no `FailedScheduling` events for it. If you manage the Secret
    yourself, check that the key exists without printing it:

    ```bash
    kubectl get secret specificai-secrets -n "$NAMESPACE" -o jsonpath='{.data}' | jq 'has("SUPERVISOR_ADMIN_TOKEN")'
    ```

=== "Azure"

    The Playground runs on the SKU you chose for the `gpu-basic` tier (or
    `gpu-low-performance`, if you set `specificai-vllm-inference.gpuTier` to
    it). That SKU needs more than 52 GiB of memory:
    `Standard_NV12ads_A10_v5` (110 GiB) or larger in the NVadsA10 v5 family
    works, and `Standard_NV36ads_A10_v5` is unaffected.
    `Standard_NV6ads_A10_v5` (55 GiB) and the smaller NCasT4_v3 sizes such as
    `Standard_NC4as_T4_v3` (28 GiB) and `Standard_NC8as_T4_v3` (56 GiB) leave
    no room for the pod's limit once AKS reserves node memory.

    #### Console steps

    1. Check the quota in **Subscriptions > your subscription > Settings >
       Usage + quotas**. Filter by the cluster's region and the VM family of
       the SKU you are moving to, and request an increase if the family's
       vCPU limit does not cover it.
    2. **AKS Automatic:** no portal step is needed. Set the new SKU in your
       values file, in both places the chart reads it:

        ```yaml
        global:
          gpuBasicNodeSelector:
            karpenter.azure.com/sku-name: "<GPU_BASIC_SKU>"   # e.g. Standard_NV12ads_A10_v5
        common:
          karpenter:
            azure:
              gpuBasicSkuNames:
                - "<GPU_BASIC_SKU>"
        ```

    3. **AKS Standard:** a node pool's VM size cannot be changed in place, so
       add a new pool. Open **Kubernetes services**, select your cluster, open
       **Settings > Node pools** and select **Add node pool**. Choose the new
       size, enable autoscaling with a minimum of 0, and under **Optional
       settings** add the Kubernetes label `workload=gpu-basic`, plus the same
       taints as your existing `gpu-basic` pool if it has any. After the
       upgrade, when no pod runs on the old pool, delete it.

    #### CLI

    ```bash
    LOCATION=<AZURE_REGION>
    az vm list-usage --location "$LOCATION" -o table | grep -i "NVADSA10v5"
    ```

    For AKS Standard, add the new pool. Linux pool names are lowercase
    alphanumeric, up to 12 characters:

    ```bash
    RG=<AKS_RESOURCE_GROUP>
    CLUSTER=<AKS_CLUSTER_NAME>

    az aks nodepool add -g "$RG" --cluster-name "$CLUSTER" \
      --name <NEW_POOL_NAME> \
      --node-vm-size <GPU_BASIC_SKU> \
      --labels workload=gpu-basic \
      --enable-cluster-autoscaler --min-count 0 --max-count <MAX_NODES> \
      --node-count 0

    # After the upgrade, once nothing runs on the old pool:
    az aks nodepool delete -g "$RG" --cluster-name "$CLUSTER" --name <OLD_POOL_NAME>
    ```

    If you manage the `specificai-secrets` Secret yourself
    (`global.createSecret: false`), add the token before upgrading:

    ```bash
    NAMESPACE=specificai
    kubectl patch secret specificai-secrets -n "$NAMESPACE" --type merge \
      -p "{\"stringData\":{\"SUPERVISOR_ADMIN_TOKEN\":\"$(openssl rand -hex 20)\"}}"
    ```

    #### Verify

    After the upgrade, start a Playground session in the platform UI, then
    check where the inference pod landed:

    ```bash
    kubectl get pods -n "$NAMESPACE" -o wide | grep vllm-inference
    kubectl get nodes -L node.kubernetes.io/instance-type,workload | grep gpu-basic
    kubectl get events -n "$NAMESPACE" --field-selector reason=FailedScheduling
    ```

    The pod reaches `Running` on a node of the new SKU, and there are no
    `FailedScheduling` events for it. If you manage the Secret yourself, check
    that the key exists without printing it:

    ```bash
    kubectl get secret specificai-secrets -n "$NAMESPACE" -o jsonpath='{.data}' | jq 'has("SUPERVISOR_ADMIN_TOKEN")'
    ```

=== "GCP"

    GKE Autopilot needs no change: it sizes the node to the pod's request.
    On GKE Standard, the default `gpu-basic` machine type `g2-standard-8`
    (32 GB) is too small for the new request. Move the tier to
    `g2-standard-16` (64 GB) or larger. The GPU stays one NVIDIA L4, so GPU
    quota is unchanged, but each node uses 8 more vCPUs.

    #### Console steps

    1. Check the regional CPU quota in **IAM & Admin > Quotas & System
       Limits**, filtered by the cluster's region and the G2 machine family.
       Request an increase if 16 vCPUs per `gpu-basic` node do not fit.
    2. **If the chart creates the ComputeClass for the tier** (the default on
       Standard), no console step is needed. Set the machine type in your
       values file:

        ```yaml
        global:
          computeClasses:
            tiers:
              gpuBasic:
                machineType: g2-standard-16
        ```

    3. **If you pre-created the `gpu-basic` node pool yourself:** open
       **Kubernetes Engine > Clusters**, select the cluster, open the **Nodes**
       tab and select the `gpu-basic` pool. Select **Edit**, change the
       machine type to `g2-standard-16`, and save. GKE recreates the pool's
       nodes, so do this when no training Job is running on them.

    #### CLI

    ```bash
    PROJECT=<GCP_PROJECT_ID>
    REGION=<GCP_REGION>
    CLUSTER=<GKE_CLUSTER_NAME>

    gcloud compute regions describe "$REGION" --project "$PROJECT" \
      --flatten="quotas[]" --format="table(quotas.metric,quotas.limit,quotas.usage)" | grep -i cpus

    # Only for a node pool you created yourself:
    gcloud container node-pools update gpu-basic \
      --cluster "$CLUSTER" --region "$REGION" --project "$PROJECT" \
      --machine-type g2-standard-16
    ```

    If you manage the `specificai-secrets` Secret yourself
    (`global.createSecret: false`), add the token before upgrading:

    ```bash
    NAMESPACE=specificai
    kubectl patch secret specificai-secrets -n "$NAMESPACE" --type merge \
      -p "{\"stringData\":{\"SUPERVISOR_ADMIN_TOKEN\":\"$(openssl rand -hex 20)\"}}"
    ```

    #### Verify

    After the upgrade, start a Playground session in the platform UI, then
    check where the inference pod landed:

    ```bash
    kubectl get pods -n "$NAMESPACE" -o wide | grep vllm-inference
    kubectl get nodes -L node.kubernetes.io/instance-type,cloud.google.com/gke-accelerator
    kubectl get events -n "$NAMESPACE" --field-selector reason=FailedScheduling
    ```

    The pod reaches `Running` on an L4 node with at least 64 GB of memory, and
    there are no `FailedScheduling` events for it. If you manage the Secret
    yourself, check that the key exists without printing it:

    ```bash
    kubectl get secret specificai-secrets -n "$NAMESPACE" -o jsonpath='{.data}' | jq 'has("SUPERVISOR_ADMIN_TOKEN")'
    ```

## 4.8.2 — 2026-09-16

No infrastructure change.

## 4.8.1 — 2026-09-13

No infrastructure change.

## 4.8.0 — 2026-09-13

No infrastructure change.

## 4.7.0 — 2026-09-12

### What changes

The Karpenter NodePools the chart creates now keep a busy node for as long as
the platform's longest Job may run: `expireAfter` and
`terminationGracePeriod` are `72h` on the GPU pools and `168h` on the CPU
pool, matching the training and evaluation Jobs' own deadlines. Before 4.7.0,
`expireAfter` was `5m` on GPU pools and `30m` on the CPU pool. Nodes therefore
expired almost immediately, and the Job pods' do-not-disrupt protection only
held them off until the node's termination grace period ran out. On EKS Auto
Mode that default is 24 hours, so any training Job still running about a day
after its node started was killed and failed with `Training job terminated
before cleanup`. Idle nodes are still removed 5 minutes after they empty, so
the change does not keep unused GPUs running.

This affects installs where the chart creates the NodePools
(`common.karpenter.createNodePools: true`): EKS Auto Mode, EKS with
Karpenter, and AKS Automatic. Helm applies the new NodePools during the
upgrade and nothing needs to change in your cloud account. Two things need
attention. NodeClaims are immutable, so nodes that exist at upgrade time keep
the old expiry, and a long Job already running on one can still be stopped by
the old deadline. And if you manage your own NodePools for the platform
instead of the chart's, they keep the short expiry until you change them.

=== "AWS"

    #### Console steps

    There is no console equivalent: the EKS console shows nodes but not their
    Karpenter expiry settings, and NodePools are edited with `kubectl`. Use the
    CLI steps below.

    #### CLI

    Before upgrading, list the platform Jobs that are still running. A Job
    started before the upgrade keeps its node's old expiry; if it is expected
    to run for more than about a day, let it finish before upgrading, or
    restart it afterwards.

    ```bash
    NAMESPACE=specificai
    kubectl get jobs -n "$NAMESPACE"   # Jobs whose COMPLETIONS is not yet 1/1 are still running
    kubectl get pods -n "$NAMESPACE" --field-selector status.phase=Running -o wide
    ```

    Run the upgrade from the [upgrade guide](upgrade.md#upgrade), then compare
    the NodePools with the nodes they have already launched:

    ```bash
    kubectl get nodepools -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.template.spec.expireAfter,GRACE:.spec.template.spec.terminationGracePeriod'

    kubectl get nodeclaims -o custom-columns='NAME:.metadata.name,NODEPOOL:.metadata.labels.karpenter\.sh/nodepool,EXPIRE_AFTER:.spec.expireAfter,CREATED:.metadata.creationTimestamp'
    ```

    NodeClaims that still show `5m` or `30m` were created before the upgrade.
    They need no action: they are replaced by nodes with the new settings once
    their pods finish and the node empties.

    If you run your own NodePools for the platform instead of the chart's,
    apply the same values yourself, `72h` for GPU pools and `168h` for CPU
    pools:

    ```bash
    kubectl patch nodepool <GPU_NODEPOOL_NAME> --type merge \
      -p '{"spec":{"template":{"spec":{"expireAfter":"72h","terminationGracePeriod":"72h"}}}}'
    kubectl patch nodepool <CPU_NODEPOOL_NAME> --type merge \
      -p '{"spec":{"template":{"spec":{"expireAfter":"168h","terminationGracePeriod":"168h"}}}}'
    ```

    #### Verify

    The first command lists every platform NodePool with `72h` or `168h` in
    both columns. After the next training Job starts, its node's NodeClaim
    shows the new `EXPIRE_AFTER`:

    ```bash
    kubectl get nodepools -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.template.spec.expireAfter,GRACE:.spec.template.spec.terminationGracePeriod'
    kubectl get nodeclaims -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.expireAfter,CREATED:.metadata.creationTimestamp'
    ```

=== "Azure"

    This applies to AKS Automatic and other AKS clusters that use node
    auto-provisioning with the chart's NodePools. AKS Standard clusters with
    pre-created node pools (`common.karpenter.createNodePools: false`) are not
    affected.

    #### Console steps

    There is no console equivalent: the Azure portal does not show or edit the
    Karpenter NodePools that node auto-provisioning uses. Use the CLI steps
    below.

    #### CLI

    Before upgrading, list the platform Jobs that are still running. A Job
    started before the upgrade keeps its node's old expiry; if it is expected
    to run for more than about a day, let it finish before upgrading, or
    restart it afterwards.

    ```bash
    NAMESPACE=specificai
    kubectl get jobs -n "$NAMESPACE"   # Jobs whose COMPLETIONS is not yet 1/1 are still running
    kubectl get pods -n "$NAMESPACE" --field-selector status.phase=Running -o wide
    ```

    Run the upgrade from the [upgrade guide](upgrade.md#upgrade), then compare
    the NodePools with the nodes they have already launched:

    ```bash
    kubectl get nodepools -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.template.spec.expireAfter,GRACE:.spec.template.spec.terminationGracePeriod'

    kubectl get nodeclaims -o custom-columns='NAME:.metadata.name,NODEPOOL:.metadata.labels.karpenter\.sh/nodepool,EXPIRE_AFTER:.spec.expireAfter,CREATED:.metadata.creationTimestamp'
    ```

    NodeClaims that still show `5m` or `30m` were created before the upgrade.
    They need no action: they are replaced by nodes with the new settings once
    their pods finish and the node empties.

    If you run your own NodePools for the platform instead of the chart's,
    apply the same values yourself, `72h` for GPU pools and `168h` for CPU
    pools:

    ```bash
    kubectl patch nodepool <GPU_NODEPOOL_NAME> --type merge \
      -p '{"spec":{"template":{"spec":{"expireAfter":"72h","terminationGracePeriod":"72h"}}}}'
    kubectl patch nodepool <CPU_NODEPOOL_NAME> --type merge \
      -p '{"spec":{"template":{"spec":{"expireAfter":"168h","terminationGracePeriod":"168h"}}}}'
    ```

    #### Verify

    The first command lists every platform NodePool with `72h` or `168h` in
    both columns. After the next training Job starts, its node's NodeClaim
    shows the new `EXPIRE_AFTER`:

    ```bash
    kubectl get nodepools -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.template.spec.expireAfter,GRACE:.spec.template.spec.terminationGracePeriod'
    kubectl get nodeclaims -o custom-columns='NAME:.metadata.name,EXPIRE_AFTER:.spec.expireAfter,CREATED:.metadata.creationTimestamp'
    ```

Not affected: GCP.

## 4.6.0 — 2026-09-09

No infrastructure change.

## 4.5.0 — 2026-09-08

### What changes

On Azure, the platform now reaches Blob storage with Workload Identity instead
of a storage account key. The backend's models volume and the Triton inference
server's model repository are both mounted with the Azure Blob CSI driver,
authenticated by the mounting pod's Kubernetes service account token, and the
chart no longer creates the `optune-storage-account-secret` key Secret. Once
the upgrade is healthy, the storage account can disable shared key access.
The behaviour is controlled by `global.azure.useWorkloadIdentityForBlob`,
which defaults to `true`.

Every existing Azure install is affected. Skipping the preparation breaks the
upgrade in one of three ways. The render fails before Helm reaches the cluster
when `backend.storage.blob.resourceGroup` or the
`specificai-inference.models.blob` values are missing. Kubernetes rejects the
upgrade with `spec.persistentvolumesource is immutable after creation`,
because the backend PersistentVolume's mount attributes cannot change on an
existing volume, so the backend PersistentVolumeClaim and PersistentVolume
must be deleted first. The volume's reclaim policy is `Retain`, so deleting
them leaves the blob data untouched. Finally, the blob mounts fail with
`AADSTS700213` when the managed identity has no federated credential for the
exact service account that mounts the volume, or cannot read the container
when it lacks a Storage Blob Data role.

If you cannot set up Workload Identity before this upgrade, you can keep the
storage account key for one more release with
`global.azure.useWorkloadIdentityForBlob: false`; see
[Azure: storage account key installs](upgrade.md#azure-storage-account-key-installs)
in the upgrade guide. That path needs no step below except keeping your
account-key values.

=== "Azure"

    The steps assume the release name `specificai` and namespace `specificai`
    from the [install guide](install.md), so the platform service account is
    `specificai` and the backend Deployment is `specificai-backend`. Replace
    them if your install differs.

    #### Console steps

    1. In the Azure portal, open **Kubernetes services**, select your
       cluster, and open **Settings > Security configuration**. Confirm that
       **OIDC issuer** and **Workload Identity** are both enabled; enable them
       and save if not. AKS Automatic clusters always have both.
    2. Open **Managed Identities** and select the identity whose client ID is
       your `global.roleId`. Under **Settings > Federated credentials**, check
       for a credential whose subject is
       `system:serviceaccount:specificai:specificai` and whose issuer is this
       cluster's OIDC issuer URL. If there is none, select **Add credential**,
       choose the **Kubernetes accessing Azure resources** scenario, and enter
       the cluster issuer URL (shown by the first CLI command below), namespace
       `specificai`, and service account `specificai`. Name it and select
       **Add**.
    3. Open the storage account that holds the platform's container and select
       **Access control (IAM) > Role assignments**. Confirm the same identity
       holds **Storage Blob Data Contributor** on the storage account or its
       resource group. If it does not, select **Add > Add role assignment**,
       pick **Storage Blob Data Contributor**, assign access to **Managed
       identity**, select the identity, and **Review + assign**.
    4. Only if you run Triton under its own service account
       (`specificai-inference.serviceAccount.create: true`): repeat steps 2 and
       3 for that dedicated identity, with service account
       `specificai-specificai-inference` and the **Storage Blob Data Reader**
       role, which is enough for its read-only mount. Set its client ID as
       `specificai-inference.serviceAccount.roleId`. By default Triton uses the
       platform service account and needs nothing extra.
    5. Add the new values to your values file. The resource group is the one
       that contains the storage account, not the AKS node resource group
       (`MC_*`):

        ```yaml
        backend:
          storage:
            blob:
              accountName: "<STORAGE_ACCOUNT_NAME>"
              containerName: "<CONTAINER_NAME>"
              resourceGroup: "<STORAGE_ACCOUNT_RESOURCE_GROUP>"
        specificai-inference:
          models:
            blob:
              enabled: true
              accountName: "<STORAGE_ACCOUNT_NAME>"
              containerName: "<CONTAINER_NAME>"
              resourceGroup: "<STORAGE_ACCOUNT_RESOURCE_GROUP>"
        ```

        Remove `backend.storage.blob.accountKey` from your values and secrets
        files.

    6. Scaling the backend down and deleting its volume objects has no portal
       equivalent; run the last CLI block right before `helm upgrade`. The
       platform is unavailable from the scale-down until the upgraded backend
       is running.

    #### CLI

    Check the cluster prerequisites. All three columns should read `True`:

    ```bash
    RG=<AKS_RESOURCE_GROUP>
    CLUSTER=<AKS_CLUSTER_NAME>

    az aks show -g "$RG" -n "$CLUSTER" \
      --query "{oidcIssuer:oidcIssuerProfile.enabled, workloadIdentity:securityProfile.workloadIdentity.enabled, blobCsiDriver:storageProfile.blobCsiDriver.enabled, issuerUrl:oidcIssuerProfile.issuerUrl}" \
      -o table

    # Only if one of them is not True:
    az aks update -g "$RG" -n "$CLUSTER" --enable-oidc-issuer --enable-workload-identity
    az aks update -g "$RG" -n "$CLUSTER" --enable-blob-driver
    ```

    Add the federated credential and the role assignment if they are missing:

    ```bash
    IDENTITY_RG=<MANAGED_IDENTITY_RESOURCE_GROUP>
    IDENTITY=<MANAGED_IDENTITY_NAME>
    STORAGE_RG=<STORAGE_ACCOUNT_RESOURCE_GROUP>
    STORAGE_ACCOUNT=<STORAGE_ACCOUNT_NAME>
    NAMESPACE=specificai

    ISSUER=$(az aks show -g "$RG" -n "$CLUSTER" --query oidcIssuerProfile.issuerUrl -o tsv)

    az identity federated-credential list -g "$IDENTITY_RG" --identity-name "$IDENTITY" \
      --query "[].{name:name, issuer:issuer, subject:subject}" -o table

    az identity federated-credential create -g "$IDENTITY_RG" --identity-name "$IDENTITY" \
      --name specificai-platform \
      --issuer "$ISSUER" \
      --subject "system:serviceaccount:${NAMESPACE}:specificai" \
      --audiences api://AzureADTokenExchange

    PRINCIPAL_ID=$(az identity show -g "$IDENTITY_RG" -n "$IDENTITY" --query principalId -o tsv)
    STORAGE_ID=$(az storage account show -g "$STORAGE_RG" -n "$STORAGE_ACCOUNT" --query id -o tsv)

    az role assignment list --assignee "$PRINCIPAL_ID" --scope "$STORAGE_ID" \
      --include-inherited --query "[].roleDefinitionName" -o tsv

    az role assignment create --assignee-object-id "$PRINCIPAL_ID" \
      --assignee-principal-type ServicePrincipal \
      --role "Storage Blob Data Contributor" --scope "$STORAGE_ID"
    ```

    Right before the upgrade, scale the backend down and delete its volume
    objects. Wait until the `get pods` command lists no backend pod before
    deleting; a PersistentVolumeClaim still in use stays `Terminating`.

    ```bash
    kubectl scale deployment specificai-backend -n "$NAMESPACE" --replicas=0
    kubectl get pods -n "$NAMESPACE" -l app=specificai-backend

    kubectl delete pvc "${NAMESPACE}-backend-pvc" -n "$NAMESPACE"
    kubectl delete pv "${NAMESPACE}-backend-pv"
    kubectl get pv "${NAMESPACE}-backend-pv"   # expect NotFound
    ```

    Then run the upgrade from the [upgrade guide](upgrade.md#upgrade). Helm
    recreates the PersistentVolume and PersistentVolumeClaim with the
    Workload Identity attributes and restores the backend's replica count.

    #### Verify

    `helm upgrade` can report success before the new pods attempt their
    mounts, so check the pods themselves:

    ```bash
    kubectl get secret optune-storage-account-secret -n "$NAMESPACE"   # expect NotFound

    kubectl get pv "${NAMESPACE}-backend-pv" -o jsonpath='{.spec.csi.volumeAttributes}'; echo
    # expect mountWithWorkloadIdentityToken, clientID and resourcegroup

    kubectl get deployment specificai-backend -n "$NAMESPACE"   # READY back at your replica count
    kubectl exec -n "$NAMESPACE" deploy/specificai-backend -- ls /app/models

    kubectl get pvc "${NAMESPACE}-specificai-inference-models-pvc" -n "$NAMESPACE"   # STATUS Bound
    kubectl get events -n "$NAMESPACE" --field-selector reason=FailedMount
    ```

    The `ls` lists the container's contents without an error, and no
    `FailedMount` event mentions `AADSTS700213` or an authorization error.
    If one does, recheck the federated credential subject and the role
    assignment for the identity named in the event.

    Once the platform is healthy, you can disable shared key access on the
    storage account (**Settings > Configuration > Allow storage account key
    access: Disabled**):

    ```bash
    az storage account update -g "$STORAGE_RG" -n "$STORAGE_ACCOUNT" --allow-shared-key-access false
    ```

    Rolling back to an earlier chart hits the same immutable volume: scale the
    backend down and delete the two volume objects again before
    `helm rollback`, and keep shared key access enabled until you no longer
    expect to roll back.

Not affected: AWS, GCP.

## 4.4.2 — 2026-09-10

No infrastructure change.

## 4.4.1 — 2026-09-09

No infrastructure change.

## 4.4.0 — 2026-09-08

No infrastructure change.

## 4.3.0 — 2026-09-08

No infrastructure change.

## 4.2.0 — 2026-09-08

No infrastructure change.

## 4.1.1 — 2026-09-06

No infrastructure change.

## 4.1.0 — 2026-08-31

No infrastructure change.

## 4.0.0 — 2026-08-19

No infrastructure change.
