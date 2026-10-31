# Data Model: Host Static Sites with Crossplane

## XStaticSite

Represents one user's high-level desired state for a static site. It is a namespaced Crossplane composite resource, declared in Git and reconciled by Argo CD.

| Attribute | Requirement |
|-----------|-------------|
| API | `platform.k8s-platform.io/v1alpha1`, kind `XStaticSite` |
| Identity | Kubernetes resource UID; used to generate a unique LocalStack bucket name |
| Index document | `spec.indexDocument`, defaults to `index.html` |
| Error document | Optional `spec.errorDocument` |
| Readiness | Standard Crossplane `Ready` and `Synced` conditions reflect the composed infrastructure |
| Website URL | `status.websiteURL`, using the LocalStack website endpoint |
| Bucket name | `status.bucketName`, generated for this XR and exposed for content upload |
| Content state | Not managed by the XR; readiness does not claim that files have been uploaded |

## Composed S3 Resources

Each `XStaticSite` owns one instance of each resource below. The resources are namespaced in the XR's namespace, use the existing `ClusterProviderConfig/localstack`, and target the existing in-cluster LocalStack endpoint.

| Resource | Purpose | Important fields |
|-----------|---------|------------------|
| S3 `Bucket` | Isolated storage for the site's objects | External name `site-<XR UID>`; region `us-east-1` |
| `BucketWebsiteConfiguration` | Enables website serving | Index suffix from the XR; optional error-document key |
| `BucketPublicAccessBlock` | Allows the website policy to serve unauthenticated reads | All four bucket-level public-access blocking/restriction flags disabled |
| `BucketPolicy` | Allows website clients to retrieve objects | `s3:GetObject` for principal `*`, scoped to this bucket's object ARN only |

The bucket policy is intentionally public-read within the lab. Its reachability is constrained by LocalStack's `ClusterIP` Service and workstation port-forward; this does not create a real AWS bucket, public Kubernetes route, or internet-facing endpoint.

## Local Website Address

The Composition reports:

```text
http://<bucket-name>.s3-website.localhost.localstack.cloud:4566/
```

From a workstation, this URL is usable while `kubectl port-forward svc/localstack 4566:4566 -n localstack` is active. The website URL serves content only after the user separately uploads files to the bucket.

## Relationships and Lifecycle

- One `XStaticSite` owns one bucket and its website, public-access, and policy resources.
- The bucket selector on each dependent managed resource matches resources controlled by the same XR; configuration for one site cannot select another site's bucket.
- The XRD defines and validates the user-facing API. The Composition translates the XR into the four S3 resources and computes its status fields.
- Removing the XR YAML from Git alone has no deletion effect because the root Argo CD application has `prune: false`. Explicitly delete the XR to trigger managed-resource cleanup.
- LocalStack is ephemeral. A restart removes bucket state and uploaded content; Crossplane may recreate declared infrastructure, but content must be uploaded again.
