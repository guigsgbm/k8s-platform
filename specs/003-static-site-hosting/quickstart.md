# Quickstart: Validating Static Sites in LocalStack

Validation guide for [spec.md](./spec.md). The resource contract and research are in [contracts/static-site.md](./contracts/static-site.md) and [research.md](./research.md).

## Prerequisites

- Minikube running, Argo CD healthy in `argocd`, Crossplane healthy in `crossplane-system`, and LocalStack healthy in `localstack`.
- The existing Crossplane AWS S3 provider and family packages are `v2.4.0`; the LocalStack `ClusterProviderConfig` and dummy-credential Secret already exist.
- The LocalStack API endpoint is `http://localstack.localstack.svc.cluster.local:4566`. No real AWS account or credentials are used.
- AWS CLI and `curl` are available on the workstation. Website access requires a temporary port-forward.

## Roll out the deterministic bucket name

The Composition now names each bucket `site-<XStaticSite name>` instead of deriving it from the XR UID. The existing `docs-demo` bucket is intentionally disposable: remove `applications/docs-demo.yaml` and wait for Crossplane to delete its XR and old bucket, then publish the Composition change and restore the declaration. Its old contents are not migrated.

Before applying the updated platform manifest, run the required server-side dry-run:

```sh
kubectl apply --dry-run=server -f applications/crossplane-static-site-api.yaml
```

## 1. Verify prerequisites before publishing feature manifests

Confirm the current provider APIs and local-only service:

```sh
kubectl get providers.pkg.crossplane.io -o wide
kubectl get crd buckets.s3.aws.m.upbound.io \
  bucketwebsiteconfigurations.s3.aws.m.upbound.io \
  bucketpolicies.s3.aws.m.upbound.io \
  bucketpublicaccessblocks.s3.aws.m.upbound.io
kubectl get svc localstack -n localstack
```

The LocalStack Service must remain `ClusterIP`, and the website URL must work only while the local port-forward is active.

## 2. Install the Composition function

Publish `applications/crossplane-patch-and-transform.yaml` first. Wait for its Function to report `INSTALLED=True` and `HEALTHY=True` before publishing a Composition that references it:

```sh
kubectl get functions.pkg.crossplane.io
```

The package is pinned to `v0.11.2`, which declares compatibility with Crossplane `2.0+`.

## 3. Publish and verify the API

After the function is healthy, publish `applications/crossplane-static-site-api.yaml`. Wait for Crossplane to establish the `XStaticSite` API and report the Composition ready before adding any site declarations:

```sh
kubectl get crd xstaticsites.platform.k8s-platform.io
kubectl get compositions.apiextensions.crossplane.io
kubectl get events -n crossplane-system --sort-by=.lastTimestamp
```

Validate the new manifests with server-side dry-run against the installed APIs. If the function, XRD, Composition, or managed-resource API is unhealthy, do not publish a site declaration; inspect the corresponding package conditions and Kubernetes events.

## 4. Declare and verify a site

Ensure the target namespace exists before adding a site (for example, manage a `static-sites` `Namespace` manifest through GitOps). Add one `XStaticSite` manifest under `applications/` in Git. For example:

```yaml
apiVersion: platform.k8s-platform.io/v1alpha1
kind: XStaticSite
metadata:
  name: docs-demo
  namespace: static-sites
spec:
  indexDocument: index.html
  errorDocument: error.html
```

Omitting `indexDocument` uses `index.html`; `errorDocument` is optional. Commit and push the declaration so the root Argo CD application creates it. Wait for the XR and its four composed infrastructure resources to report ready. The bucket name is `site-docs-demo`; readiness does not indicate that content has been published:

```sh
kubectl describe xstaticsite docs-demo -n static-sites
kubectl get buckets,bucketwebsiteconfigurations,bucketpublicaccessblocks,bucketpolicies \
  -n static-sites
```

Read `status.bucketName` and `status.websiteURL` from the XR. The bucket name is deterministically derived from the `XStaticSite` name.

## 5. Publish content and test the website

The static-site repository workflow owns website content. It publishes the configured content directory on default-branch pushes or manual dispatch through the runner's LocalStack endpoint; it does not use Kubernetes access or repository secrets. Follow [the repository contract](../006-site-repository-contract/contracts/site-repository.md) to create and publish a conforming site. Confirm the workflow completes before requesting the website.

Start the local-only port-forward in a terminal:

```sh
kubectl port-forward svc/localstack 4566:4566 -n localstack
```

Request the reported website URL and a published static asset while the port-forward is active:

```sh
curl --fail "$(kubectl get xstaticsite docs-demo -n static-sites \
  -o jsonpath='{.status.websiteURL}')"
```

Confirm the response contains the committed index page and the linked asset is served. `Ready=True` and `Synced=True` describe infrastructure, not publication state.

## 6. Failure, cleanup, and LocalStack restart

- Inspect XR conditions, composed managed resources, and events for failures. The provider configuration must continue targeting the exact LocalStack endpoint with dummy credentials only.
- The root Argo CD application has `prune: false`. Removing an XR file from Git alone does not delete the live XR. Remove the declaration from Git, wait for Argo CD to sync, then explicitly delete the XR:

  ```sh
  kubectl delete xstaticsite docs-demo -n static-sites
  ```

- Deleting the XR deletes the generated bucket and its contents. Back up any files that must be retained before cleanup.
- Wait for Crossplane to delete that XR's managed resources and LocalStack bucket before removing the shared Composition or function. Do not remove the shared provider configuration, provider packages, or dummy Secret while another site or S3 resource uses them.
- LocalStack persistence is disabled. A restart loses bucket contents; the repository workflow must publish the site files again.
