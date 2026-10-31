# Implementation Plan: Host Static Sites with Crossplane

**Branch**: `main` (specification directory is pinned by `.specify/feature.json`) | **Date**: 2026-10-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/003-static-site-hosting/spec.md`

## Summary

Expose a namespaced Crossplane `XStaticSite` API backed by a Composition. A declaration sets an index document (default `index.html`) and optional error document; the Composition creates an S3 bucket, website configuration, a public-access-block configuration, and a narrowly scoped public-read bucket policy in LocalStack. It publishes the LocalStack website URL and Crossplane readiness through the composite resource status.

The live Minikube cluster already has Crossplane `v2.4.2`, the AWS family and S3 providers `v2.4.0`, their required S3 CRDs, the LocalStack `ClusterProviderConfig`, and a ready smoke-test bucket. The provider has no Composition function installed, so this feature adds the pinned `function-patch-and-transform:v0.11.2` package. The website endpoint is LocalStack's `s3-website.localhost.localstack.cloud` endpoint; users reach it from their workstation only while the documented LocalStack port-forward is active.

## Technical Context

**Language/Version**: Kubernetes YAML and Markdown; Crossplane `v2.4.2`

**Primary Dependencies**: Crossplane `v2.4.2`; Upbound `provider-family-aws:v2.4.0` and `provider-aws-s3:v2.4.0` (existing); `function-patch-and-transform:v0.11.2`; LocalStack image `2026.10.0` (existing)

**Storage**: One ephemeral LocalStack S3 bucket per site; site content is uploaded separately and is lost when LocalStack restarts.

**Testing**: Validate manifests against the served APIs; verify Function health, XRD establishment, Composition readiness, and every managed resource's `Ready=True`/`Synced=True`; upload a small test page through the explicit LocalStack endpoint; request the website URL through a temporary port-forward; test error documents and two-site isolation.

**Target Platform**: Existing Minikube cluster; Argo CD in `argocd`; Crossplane in `crossplane-system`; LocalStack in `localstack`

**Project Type**: Kubernetes platform GitOps repository

**Performance Goals**: All required resources ready within 10 minutes; website content retrievable within 5 minutes after upload; readiness, failure reason, and URL discoverable within 5 minutes.

**Constraints**: Pin the new function package; reuse the existing AWS provider and LocalStack endpoint; use no real AWS account or credentials; do not commit secrets; do not create an Ingress or other public route; allow anonymous object reads only inside the LocalStack lab; document explicit cleanup because the root Argo CD application has `prune: false`.

**Scale/Scope**: One lab; multiple independently named sites; one bucket and one set of hosting controls per site; no content build or deployment pipeline.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. The Function, XRD, Composition, and site declarations are Git-managed. Content remains an explicitly separate upload. |
| II. Focus on platform infrastructure | Pass. The feature provides a reusable platform API and its infrastructure resources, not a site application or build pipeline. |
| III. Reconcile automatically, prune deliberately | Pass with an operational note. The root application has `prune: false`; deleting a declaration from Git does not delete its XR or bucket. The quickstart requires explicit deletion and confirms completion. |
| IV. Keep secrets out of Git | Pass. The feature reuses the existing out-of-Git dummy-credentials Secret through the existing LocalStack-only `ClusterProviderConfig`. |
| V. Make deployments reproducible | Pass with the existing lab-only AWS provider exception. The S3 provider remains pinned to `v2.4.0`; its accepted vulnerability findings are documented in [feature 002 research](../002-crossplane-s3-localstack/research.md). The new Composition function is pinned to `v0.11.2`. |
| VI. Review cluster impact | Pass. The plan adds one Composition function and four namespaced S3 resources per declared site. The bucket policy grants only object reads, the endpoint remains internal, and no LocalStack or Crossplane upgrade is planned. |

**Post-design re-check**: Pass. The public-read policy is confined to the emulator's internal endpoint and the workstation's loopback-bound port-forward. No public Kubernetes route is introduced. The release-risk exception remains the already accepted, lab-only AWS provider pin.

## Project Structure

### Documentation (this feature)

```text
specs/003-static-site-hosting/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── static-site.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Created by /speckit-tasks, not by this plan
```

### Source Code (repository root)

```text
applications/
├── crossplane-patch-and-transform.yaml  # New: pinned Function package
├── crossplane-static-site-api.yaml      # New: XRD and Composition
└── <site-declaration>.yaml              # Added by GitOps users, one per site
README.md                                # Updated: static-site operations and cleanup
```

**Structure Decision**: Keep platform APIs and package declarations as manifests directly under the existing `applications/` path watched by the root Argo CD application. Users add each `XStaticSite` declaration as a separate manifest there. Roll out in stages: install and verify the function first; then publish the XRD and Composition; finally publish site declarations after the API and Composition are established. This avoids applying resources before their CRDs or function are ready.

## Complexity Tracking

| Violation / exception | Why needed | Simpler alternative rejected because |
|-----------------------|------------|-------------------------------------|
| Reuse AWS S3 provider `v2.4.0` with its previously disclosed Marketplace findings | The working lab already runs this pin and it has passed LocalStack reconciliation; changing the provider is explicitly out of scope and risks the existing custom-endpoint integration. The accepted lab-only risk and review requirement are in [feature 002 research](../002-crossplane-s3-localstack/research.md). | A new provider release has not been validated against this LocalStack setup. Revisit the pin before any real AWS use or when a secure, tested release is available. |
| Permit anonymous `s3:GetObject` on each emulated site's objects | S3 website hosting must serve content without AWS credentials. The policy is scoped to one generated bucket and the LocalStack service remains ClusterIP-only. | Private objects cannot satisfy the static website acceptance scenarios; a public route or real AWS bucket is out of scope. |
