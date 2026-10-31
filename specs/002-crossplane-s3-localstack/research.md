# Research: Manage LocalStack S3 Buckets with Crossplane

Researched 2026-10-30 against the live Minikube cluster, Crossplane v2.4 documentation, Upbound Marketplace, LocalStack documentation, and the upstream `provider-upjet-aws` issue tracker.

## R1. Crossplane version and provider compatibility

- **Observed**: The live `crossplane` and `crossplane-rbac-manager` deployments use `xpkg.crossplane.io/crossplane/crossplane:v2.4.2`. The Crossplane chart declaration in [applications/crossplane.yaml](../../applications/crossplane.yaml) also pins chart `2.4.2`. Read-only implementation checks reconfirmed that version and found no AWS Provider resources or S3/provider-configuration CRDs installed. The LocalStack service is a ready ClusterIP with ready endpoints; an AWS CLI `s3 ls` request using dummy credentials reached the exact in-cluster endpoint.
- **Decision**: Use the Upbound service-scoped `provider-aws-s3` package and its `provider-family-aws` package, both pinned to `v2.4.0`.
- **Compatibility evidence**: The [provider-aws-s3 v2.4.0 Marketplace page](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v2.4.0) and [provider-family-aws v2.4.0 Marketplace page](https://marketplace.upbound.io/providers/upbound/provider-family-aws/v2.4.0) both declare compatibility with Crossplane `2.0+`; the live `v2.4.2` installation is within that stated range. Crossplane's [v2.4 provider documentation](https://docs.crossplane.io/v2.4/packages/providers/) explains that packages may constrain their Crossplane version and that Crossplane checks those constraints by default.
- **Package source**: `xpkg.upbound.io/upbound/provider-family-aws:v2.4.0` and `xpkg.upbound.io/upbound/provider-aws-s3:v2.4.0`. Pin the family package explicitly to the same release as the service provider.
- **Implementation schema check**: The v2.4.0 source defines `spec.endpoint.url` with static URL configuration, an explicit `services` list, `source: Custom`, `hostnameImmutable`, and the `s3_use_path_style` / AWS-validation skip fields used by the manifest. See the [provider v2.4.0 API types](https://github.com/crossplane-contrib/provider-upjet-aws/blob/v2.4.0/apis/cluster/v1beta1/types.go).
- **Alternatives considered**:
  - `provider-aws-s3:v2.9.5`: Marketplace declares Crossplane `2.0+` compatibility and reports no known vulnerabilities, but the open S3 custom-endpoint regression described in R2 has not been shown fixed or verified for this release.
  - `provider-aws-s3:v2.4.1`: Marketplace declares Crossplane `2.0+` compatibility but currently reports one Critical vulnerability.
  - The broader legacy AWS provider: not selected; the service-scoped Upbound package has direct LocalStack integration guidance and version-specific evidence relevant to this feature.

## R2. LocalStack endpoint and custom S3 behavior

- **Decision**: Configure the provider's custom endpoint as `http://localstack.localstack.svc.cluster.local:4566`, use path-style S3 requests, and map every service needed by bucket reconciliation—including `s3` and `s3control`—to the custom endpoint. Map STS to the same endpoint if used by the provider.
- **Rationale**: The [LocalStack Crossplane guide](https://docs.localstack.cloud/aws/connecting/infrastructure-as-code/crossplane/) demonstrates the provider-family pattern, a static custom endpoint, dummy credentials, path-style S3, and verification of the created bucket in LocalStack. Its package example is older and is used only as evidence for the integration pattern, not for the version selection.
- **Important failure mode**: The open upstream [issue #2213](https://github.com/crossplane-contrib/provider-upjet-aws/issues/2213), updated 2026-09-16, reports that `provider-aws-s3` versions `v2.5.0` and later fail to observe buckets at custom S3 endpoints because the S3 Control tagging call does not correctly fall back. The report's version table says `v2.4.0` worked and versions `v2.5.0`, `v2.5.5`, `v2.6.3`, and `v2.7.1` failed. The issue remained open when checked; it does not explicitly test `v2.9.5`.
- **Endpoint safety**: A service omitted from the provider's custom endpoint mapping could use its normal AWS endpoint. Include the S3 Control endpoint and any other API actually used by reconciliation. Use dummy `test` credentials only, and make the smoke test verify Crossplane's `Ready` and `Synced` conditions across repeated observations, not just initial bucket creation.

## R3. Security status and accepted release trade-off

- **Decision**: Proceed with `v2.4.0` for this study lab, as selected by the user after disclosure. Do not reuse this choice for a production AWS integration.
- **Evidence**: Rechecking both version-specific Upbound Marketplace pages on 2026-10-30 confirmed 0 Critical, 2 High, 0 Medium, and 2 Low findings for each of `provider-aws-s3:v2.4.0` and `provider-family-aws:v2.4.0`; this does not change the accepted lab-only risk. The Marketplace does not identify the individual findings in the page content retrieved for this plan. The newer `provider-aws-s3:v2.9.5` page reported no known vulnerabilities.
- **Mitigations and gates**:
  - Re-check the Marketplace vulnerability status and exact package versions before merge.
  - Store only dummy LocalStack credentials (`test` / `test`) in the out-of-Git Kubernetes Secret. Never configure a real AWS account or real AWS credentials.
  - Keep all provider operations directed to the LocalStack service and fail the acceptance test if reconciliation succeeds without the bucket appearing in LocalStack.
  - Revisit and upgrade the pin when a later release both addresses the open custom-endpoint issue and passes the LocalStack reconciliation test.

## R4. Argo CD and resource ordering

- **Observed**: The root Argo CD application sources the `applications/` directory, enables automated sync and self-healing, and has `prune: false` in [bootstrap/root-application.yaml](../../bootstrap/root-application.yaml). The Crossplane Provider CRD exists, but AWS ProviderConfig and S3 Bucket CRDs are not yet installed.
- **Decision**: Stage the first installation through Git:
  1. Commit the two pinned `Provider` package declarations and wait for the AWS family and S3 provider to report installed and healthy.
  2. Confirm the provider APIs are served; create the dummy-credentials Secret outside Git.
  3. Commit the `ClusterProviderConfig` and bucket resource for Argo CD to reconcile.
- **Rationale**: The first-stage provider package installs the custom APIs that the second-stage manifests require. Keeping those stages distinct avoids an initial Argo CD sync racing ahead of CRD installation.
- **Removal consequence**: Removing a resource manifest from Git does not automatically delete the live object because root pruning is disabled. The quickstart must describe explicit deletion and the effect on the LocalStack bucket.

## R5. Credential and bucket contract

- **Decision**: Store a credentials file containing only the LocalStack dummy credentials in a Kubernetes Secret in `crossplane-system`, outside Git. Reference it from a `ClusterProviderConfig`. Create the S3 bucket as a namespaced Crossplane managed resource.
- **Bucket verification**: Crossplane must report the bucket `Ready=True` and `Synced=True`; an AWS CLI request with the explicit LocalStack endpoint must list the same bucket.
- **Deletion**: Declare the managed-resource deletion behavior explicitly where supported by the selected API. The cleanup guide must account for the root application's `prune: false`; do not imply deleting the Git file alone removes the bucket.

## R6. Validation

- Render/parse the desired-state YAML and validate the package manifests against the installed Crossplane APIs.
- After provider installation, use server-side dry-run for the ProviderConfig and bucket resources.
- Confirm both provider package resources are healthy, the provider configuration is valid, and the bucket reaches `Ready=True` and `Synced=True`.
- Use the AWS CLI with dummy credentials and the exact internal endpoint to list the bucket in LocalStack.
- Do not apply any desired-state changes to the live cluster during planning. The cluster checks in R1 and R4 were read-only.
