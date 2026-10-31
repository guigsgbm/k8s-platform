# Static Site Contract for LocalStack

This contract describes the Crossplane resources planned for the lab. It is not a manifest to apply directly; implementation produces the version-controlled YAML.

## User-Facing API

The API is a namespaced `XStaticSite` composite resource:

| Field | Contract |
|-------|----------|
| API version | `platform.k8s-platform.io/v1alpha1` |
| Kind | `XStaticSite` |
| `spec.indexDocument` | Optional; defaults to `index.html` and maps to the website configuration's `indexDocument.suffix` |
| `spec.errorDocument` | Optional; maps to `errorDocument.key` |
| `status.conditions` | Crossplane's readiness and reconciliation status |
| `status.bucketName` | Generated LocalStack bucket name, `site-<XR UID>` |
| `status.websiteURL` | `http://<bucket-name>.s3-website.localhost.localstack.cloud:4566/` |

The XR reports infrastructure readiness and includes a minimal placeholder index page. Users upload their own site content separately; the Composition does not manage it.

## Composition

The Composition uses the pinned `function-patch-and-transform:v0.11.2` package and creates the following namespaced managed resources for each XR:

1. `s3.aws.m.upbound.io/v1beta1` `Bucket`, with an external name based on the XR UID and region `us-east-1`.
2. `BucketWebsiteConfiguration`, selected from the bucket controlled by the same XR, with the configured index and optional error document.
3. `BucketPublicAccessBlock`, selected from that same bucket, with all four public-access blocking/restriction fields set to `false`.
4. `BucketPolicy`, selected from that same bucket, allowing only `s3:GetObject` on `arn:aws:s3:::<bucket-name>/*` for website clients.
5. `Object`, selected from that same bucket, with key taken from `spec.indexDocument`, content type `text/html`, and a fixed inline placeholder page. It lets a new site respond immediately; it does not deliver real site content.

Each managed resource references the existing cluster-scoped `ClusterProviderConfig/localstack`. S3, S3 Control, and any provider operations required by reconciliation must remain directed to `http://localstack.localstack.svc.cluster.local:4566`. The provider's existing Secret contains only dummy LocalStack credentials and remains outside Git.

The policy grants anonymous reads only for objects in the emulated site bucket. The LocalStack Service remains `ClusterIP`; no ingress, NodePort, public DNS record, real AWS resource, or real credential is introduced.

## Website Access

The website URL follows LocalStack's documented endpoint format:

```text
http://<bucket-name>.s3-website.localhost.localstack.cloud:4566/
```

For workstation access, run:

```sh
kubectl port-forward svc/localstack 4566:4566 -n localstack
```

Then open the URL reported in `status.websiteURL`. Verify that the hostname resolves to the workstation's loopback and that the URL is not reachable without the port-forward. The in-cluster S3 management endpoint is separate and remains `http://localstack.localstack.svc.cluster.local:4566`.

## Reconciliation and Deletion

1. Install the pinned Composition Function and wait for it to be healthy.
2. Install the XRD and Composition, then wait for the API and Composition to become ready.
3. Add `XStaticSite` declarations through GitOps.
4. Verify the XR and all composed resources are ready and verify content through LocalStack's website endpoint.
5. To remove a site, remove its declaration from Git and explicitly delete the XR. Wait for its composed resources and LocalStack bucket to be deleted.

The root Argo CD application has `prune: false`; deleting a Git file alone does not delete the Kubernetes object. Do not remove the Composition or Function while any XR depends on them. LocalStack is ephemeral, so uploaded content is lost on restart; Crossplane recreates the placeholder index page, but any other files must be uploaded again.
