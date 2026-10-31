# AWS S3 Bucket Contract for LocalStack

This contract describes the Crossplane resources planned for the study lab. It is not a manifest to apply directly; the version-controlled YAML is produced during implementation.

## Provider Packages

Install both exact-pinned packages after the Crossplane installation is ready:

| Role | Package |
|------|---------|
| Shared AWS provider configuration APIs | `xpkg.upbound.io/upbound/provider-family-aws:v2.4.0` |
| S3 managed-resource controller | `xpkg.upbound.io/upbound/provider-aws-s3:v2.4.0` |

The Upbound Marketplace declares both compatible with Crossplane `2.0+`. The live control plane is `v2.4.2`. The selected release's vulnerability findings and custom-endpoint limitation are documented in [research.md](../research.md).

## Provider Configuration

The provider configuration must:

- Use the provider's custom, static endpoint at `http://localstack.localstack.svc.cluster.local:4566`.
- Configure `spec.endpoint.url` with `type: Static` and the LocalStack URL, and set `source: Custom`.
- Map `s3`, `s3control`, and `sts` calls to that endpoint. The exact v2.4.0 provider schema uses a shared endpoint URL with an explicit service list.
- Keep the endpoint hostname immutable and configure `s3_use_path_style: true`.
- Enable S3 path-style addressing for LocalStack.
- Reference a credentials Secret in `crossplane-system` containing only dummy `test` credentials.
- Skip AWS-only credential, metadata, region, or account checks when required by the provider version and LocalStack.
- Fail visibly when the LocalStack endpoint is unavailable; it must not fall back to a real AWS endpoint.

The Secret itself is created outside Git. Only its reference is part of the desired-state contract.

## Bucket Resource

The S3 Bucket is a namespaced Crossplane managed resource in `crossplane-system`. It specifies region `us-east-1` and references the LocalStack `ClusterProviderConfig`. The sample resource name is `localstack-s3-smoke`.

The pinned `provider-aws-s3:v2.4.0` package defines the namespaced API `s3.aws.m.upbound.io/v1beta1`; confirm the CRD is served in the cluster before applying the bucket manifest. Its `managementPolicies` explicitly allow Observe, Create, Update, Delete, and LateInitialize actions.

## Reconciliation Contract

1. Argo CD reconciles the provider packages from Git.
2. The provider packages install and report healthy; their APIs become available.
3. Argo CD reconciles the ProviderConfig and Bucket declarations from Git.
4. Crossplane reports the Bucket `Ready=True` and `Synced=True`.
5. An AWS CLI request using the explicit LocalStack endpoint lists the same bucket.

The initial deployment is staged to respect the CRD dependency. Later changes are made through Git. Since the root Argo CD application has pruning disabled, deletion of a Git declaration alone does not remove the corresponding Kubernetes object.
