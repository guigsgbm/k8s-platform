# Research: Host Static Sites with Crossplane

Researched 2026-10-30 against the live Minikube cluster, Crossplane v2.4 documentation, the pinned Upbound provider source, and LocalStack's static-website tutorial.

## R1. Current cluster and provider readiness

- **Observed read-only**: The current context is `minikube`. Crossplane and its RBAC manager run `v2.4.2`. `provider-family-aws` and `provider-aws-s3` `v2.4.0` report `INSTALLED=True` and `HEALTHY=True`.
- **Observed read-only**: The namespaced CRDs `Bucket`, `BucketWebsiteConfiguration`, `BucketPolicy`, and `BucketPublicAccessBlock` are served. The existing `ClusterProviderConfig/localstack` and `localstack-s3-smoke` bucket exist; the smoke bucket reports `Ready=True` and `Synced=True`.
- **Observed read-only**: LocalStack is running behind a `ClusterIP` service on port `4566`. No `Function` package is installed.
- **Decision**: Reuse the current provider configuration and packages. Add only the Composition function required by the high-level API; do not upgrade Crossplane, the AWS provider, or LocalStack.
- **Boundary**: No feature resources or website test bucket were created during planning. Before writing or publishing the feature manifests, implementation must perform a disposable, LocalStack-only website smoke test and remove its test bucket. Do not test against real AWS.

## R2. S3 website-hosting APIs in the pinned provider

- **Verified**: The `provider-upjet-aws` `v2.4.0` generated namespaced example defines `s3.aws.m.upbound.io/v1beta1` `BucketWebsiteConfiguration` with `indexDocument.suffix`, optional `errorDocument.key`, `region`, and a bucket selector. This confirms the API version and document field shapes for the installed package.
- **Verified**: The same pinned source has namespaced `BucketPolicy` and `BucketPublicAccessBlock` examples. The latter exposes the four public-access controls; the former accepts a policy document and bucket selector.
- **Decision**: Compose one S3 `Bucket`, `BucketWebsiteConfiguration`, `BucketPublicAccessBlock`, and `BucketPolicy` for each XR. Use `matchControllerRef` bucket selectors where available so each child configuration targets only its owning site's bucket. Allow anonymous `s3:GetObject` only on that bucket's object ARN, and disable its public-access-block settings so the website can serve those objects.
- **Evidence**: [Website configuration example](https://raw.githubusercontent.com/crossplane-contrib/provider-upjet-aws/v2.4.0/examples-generated/namespaced/s3/v1beta1/bucketwebsiteconfiguration.yaml), [bucket policy example](https://raw.githubusercontent.com/crossplane-contrib/provider-upjet-aws/v2.4.0/examples-generated/namespaced/s3/v1beta1/bucketpolicy.yaml), and [public-access-block example](https://raw.githubusercontent.com/crossplane-contrib/provider-upjet-aws/v2.4.0/examples-generated/namespaced/s3/v1beta1/bucketpublicaccessblock.yaml).

## R3. LocalStack static website endpoint and access

- **Verified from vendor guidance**: LocalStack supports S3 static website hosting, including index and error documents, and documents the website endpoint format `http://<bucket>.s3-website.localhost.localstack.cloud:4566/`. The tutorial uses a public-read bucket policy and verifies the index and error responses.
- **Decision**: Publish that endpoint in the XR's status. For workstation access, use the existing `kubectl port-forward svc/localstack 4566:4566 -n localstack` workflow. The `localhost.localstack.cloud` hostname is intended to resolve to the workstation's loopback address, where the port-forward listens; it is not the in-cluster Service DNS name. Test this exact hostname and port-forward route before merge.
- **Network boundary**: Continue using the existing `ClusterIP` Service, with no ingress, NodePort, or public DNS/route. Website content is unauthenticated to clients that can reach the LocalStack endpoint, so document the scope of the bucket policy and confirm no exposure changes in the smoke test.
- **S3 management endpoint**: Crossplane and upload operations continue targeting the existing `http://localstack.localstack.svc.cluster.local:4566` endpoint with dummy credentials. Keep all required provider service mappings pointed to LocalStack; do not rely on AWS endpoint defaults.
- **Evidence**: [LocalStack static website tutorial](https://docs.localstack.cloud/aws/tutorials/s3-static-website-terraform/) and [LocalStack endpoint-access guidance](https://docs.localstack.cloud/aws/customization/networking/accessing-endpoint-url/).

## R4. Composition API and function package

- **Verified**: Crossplane v2.4 uses namespaced composite resource definitions with `apiextensions.crossplane.io/v2`; its composition documentation supports a pipeline that invokes a Composition function. The patch-and-transform function supports patches between the XR and composed resources, including publishing computed values into XR status.
- **Observed**: No Function package is installed in the current cluster.
- **Decision**: Install `xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.11.2` as a pinned `Function`. The Upbound Marketplace package page declares compatibility with Crossplane `2.0+` and lists the release as published on 2026-10-26, which includes Crossplane `2.4.2`.
- **Evidence**: [Crossplane v2.4 XRD documentation](https://docs.crossplane.io/v2.4/composition/composite-resource-definitions/), [Composition documentation](https://docs.crossplane.io/v2.4/composition/compositions/), [function package documentation](https://docs.crossplane.io/v2.4/packages/functions/), and [function-patch-and-transform v0.11.2](https://marketplace.upbound.io/functions/crossplane-contrib/function-patch-and-transform/v0.11.2).

## R5. Naming, isolation, and reported status

- **Decision**: Generate a bucket name as `site-<XStaticSite metadata.uid>`. The UID is unique across namespaces, avoids S3 bucket-name collisions between sites with the same Kubernetes name, and keeps the generated name within S3's length limit. Use that exact value for the bucket external name, policy resource ARN, `status.bucketName`, and website URL.
- **Decision**: Define a namespaced `XStaticSite` API with `spec.indexDocument` defaulting to `index.html` and optional `spec.errorDocument`. Expose the standard Crossplane readiness conditions plus `status.bucketName` and `status.websiteURL`. Reject empty or invalid document names through the XRD schema where practical; otherwise leave a visible reconciliation failure rather than reporting the site ready.
- **Readiness semantics**: Site readiness means the bucket, website configuration, policy, and public-access-block resources are reconciled. It does not mean content has been uploaded. Document this distinction in the API contract and quickstart.
- **Evidence**: Crossplane v2.4 documents XRD scope and status fields; the provider examples confirm the exact website configuration fields and bucket selectors.

## R6. Rollout, GitOps, and lifecycle

- **Decision**: Use three GitOps stages: (1) install the function and wait for `INSTALLED=True` and `HEALTHY=True`; (2) add the XRD and Composition and wait for the XR API and Composition to become established; (3) add site declarations.
- **Removal consequence**: The root application has `prune: false`, so removing an XR manifest does not delete its Kubernetes XR or external LocalStack bucket. The cleanup guide must explicitly delete the XR, wait for its composed resources to finish deleting, then remove the Composition, XRD, and function only when no site depends on them.
- **LocalStack lifecycle**: Persistence remains disabled. A restart loses bucket data and uploaded content; while the XR remains, Crossplane should recreate the bucket and hosting configuration, but users must re-upload content.
- **Security note**: Reuse the lab-only AWS provider `v2.4.0` release and its previously disclosed Marketplace findings; see [feature 002 research](../002-crossplane-s3-localstack/research.md). Re-check that exception before publishing and do not use this configuration for real AWS.
