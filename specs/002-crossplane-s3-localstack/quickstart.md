# Quickstart: Validating Crossplane S3 Buckets in LocalStack

Validation guide for [spec.md](./spec.md). The resource contract and version rationale are in [contracts/aws-s3-bucket.md](./contracts/aws-s3-bucket.md) and [research.md](./research.md).

## Prerequisites

- Minikube running, Argo CD healthy in `argocd`, Crossplane healthy in `crossplane-system`, and LocalStack healthy in `localstack`.
- The existing endpoint is `http://localstack.localstack.svc.cluster.local:4566`.
- AWS CLI available in the temporary smoke-test pod.
- No real AWS account or credentials are used. The AWS credential Secret contains only dummy LocalStack credentials and is created outside Git.

## 1. Install the provider packages

Publish only `applications/crossplane-aws-s3.yaml` in the first rollout stage. If preparing both stages in one worktree, do not include `applications/localstack-s3-bucket.yaml` in this first commit. Argo CD's root application watches `applications/` and automatically reconciles these package declarations.

Wait for both providers to report installed and healthy, and confirm the provider APIs have been served before proceeding:

```sh
kubectl get applications.argoproj.io -n argocd
kubectl get providers.pkg.crossplane.io -o wide
kubectl get crd clusterproviderconfigs.aws.m.upbound.io \
  buckets.s3.aws.m.upbound.io
```

Both `Provider` resources must show `INSTALLED=True` and `HEALTHY=True`. The CRD check must succeed for both APIs. The package pins are `v2.4.0`; Upbound Marketplace declares compatibility with Crossplane `2.0+`. The live lab runs Crossplane `v2.4.2`. The selected release has two High and two Low Marketplace findings for each package; see the accepted lab-only exception in [plan.md](./plan.md) and [research.md](./research.md).

If installation is unhealthy or exceeds 10 minutes, do not publish the second stage. Inspect the package conditions and events:

```sh
kubectl describe provider.pkg.crossplane.io/upbound-provider-family-aws
kubectl describe provider.pkg.crossplane.io/upbound-provider-aws-s3
kubectl get events -n crossplane-system --sort-by=.lastTimestamp
```

## 2. Create the LocalStack-only credentials Secret

After both providers are healthy and their APIs are served, create the credentials Secret outside Git. These values are dummy LocalStack credentials, not valid AWS keys:

```sh
printf '[default]\naws_access_key_id = test\naws_secret_access_key = test\n' |
  kubectl create secret generic localstack-aws-credentials \
    -n crossplane-system \
    --from-file=creds=/dev/stdin
```

Do not replace the dummy values with real AWS credentials or commit this Secret. Verify the key exists without displaying its value:

```sh
kubectl describe secret localstack-aws-credentials -n crossplane-system
```

The output should list the `creds` data key only.

## 3. Reconcile the LocalStack provider configuration and bucket

Only after steps 1 and 2, publish `applications/localstack-s3-bucket.yaml`. Its version-pinned API schema routes S3, S3 Control, and STS to the exact LocalStack URL, uses path-style S3 requests, skips AWS-only validation, and references only the out-of-Git dummy Secret. If LocalStack is unavailable, those configured requests fail against that endpoint; they are not retried against real AWS.

Validate the manifest against the installed APIs before publishing, then wait for Argo CD to sync:

```sh
kubectl apply --dry-run=server -f applications/localstack-s3-bucket.yaml
kubectl get applications.argoproj.io -n argocd
```

Inspect provider configuration and bucket status. The bucket must become both `Ready=True` and `Synced=True` within 10 minutes:

```sh
kubectl describe clusterproviderconfig.aws.m.upbound.io/localstack
kubectl wait --for=condition=Ready \
  bucket.s3.aws.m.upbound.io/localstack-s3-smoke \
  -n crossplane-system --timeout=10m
kubectl wait --for=condition=Synced \
  bucket.s3.aws.m.upbound.io/localstack-s3-smoke \
  -n crossplane-system --timeout=10m
kubectl describe bucket.s3.aws.m.upbound.io/localstack-s3-smoke \
  -n crossplane-system
kubectl get events -n crossplane-system --sort-by=.lastTimestamp
```

If a resource is not ready, inspect its conditions and events before continuing. A missing Secret, unhealthy package, invalid bucket, or unreachable LocalStack endpoint must remain visible as a reconciliation failure; do not substitute real credentials or an AWS endpoint.

## 4. Verify the bucket in LocalStack

Run the AWS CLI with dummy credentials and the explicit in-cluster endpoint:

```sh
kubectl run crossplane-s3-smoke -n localstack --rm -i --restart=Never \
  --image=amazon/aws-cli:2.38.1 \
  --env=AWS_ACCESS_KEY_ID=test \
  --env=AWS_SECRET_ACCESS_KEY=test \
  --env=AWS_DEFAULT_REGION=us-east-1 \
  --command -- aws --endpoint-url \
    http://localstack.localstack.svc.cluster.local:4566 s3 ls
```

**Expected**: the output includes `localstack-s3-smoke`. This command explicitly targets the in-cluster LocalStack endpoint, not AWS. Confirm the Crossplane bucket remains `Ready=True` and `Synced=True` on subsequent observations.

LocalStack persistence is disabled. A LocalStack restart loses simulated bucket contents; while the Bucket resource remains declared, Crossplane should reconcile the bucket back into the empty emulator.

## 5. Removal and cleanup

The root Argo CD application has `prune: false`. Removing the bucket declaration from Git alone therefore does not delete the live managed resource.

1. Remove the Bucket from the manifest in Git and wait for the root application to sync. The Kubernetes resource remains because pruning is disabled.
2. Ensure the LocalStack bucket contains no objects, then explicitly delete the Crossplane Bucket resource. Its management policies allow deletion of the external bucket:

   ```sh
   kubectl delete bucket.s3.aws.m.upbound.io/localstack-s3-smoke \
     -n crossplane-system
   ```

3. Wait for Crossplane to finish deleting the LocalStack bucket. Then remove the provider configuration and dummy Secret if no other resource uses them:

   ```sh
   kubectl delete clusterproviderconfig.aws.m.upbound.io/localstack
   kubectl delete secret localstack-aws-credentials -n crossplane-system
   ```

Do not remove the provider packages while managed resources still depend on them.
