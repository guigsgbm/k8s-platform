# Tasks: Host Static Sites with Crossplane

**Input**: Design documents from `/specs/003-static-site-hosting/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: No automated test suite is requested for this declarative GitOps feature. Validate manifests with server-side dry-run against served APIs, then perform the staged rollout and LocalStack-only smoke tests in [quickstart.md](./quickstart.md).

**Organization**: Tasks are grouped by user story. The Function and API must be rolled out in order before testing a site declaration.

## Format: `[ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel with other marked tasks in the same phase when files and prerequisites do not overlap.
- **[Story]**: User story from `spec.md`.
- Operational tasks identify the runbook they follow.

## Phase 1: Setup

**Purpose**: Repository scaffolding and project prerequisites.

The `applications/` directory, root Argo CD Application, Crossplane, LocalStack, and feature structure already exist. No project setup or new dependencies are needed.

## Phase 2: Foundational

**Purpose**: Reconfirm the lab-only compatibility and website smoke-test gates before creating feature manifests.

- [x] T001 Confirm the target context is the intended Minikube lab; verify Crossplane `v2.4.2`, the AWS family and S3 providers `v2.4.0`, the four required namespaced S3 APIs, `ClusterProviderConfig/localstack`, the out-of-Git dummy-credentials Secret, and the LocalStack `ClusterIP` service. Confirm the provider endpoints still target only `http://localstack.localstack.svc.cluster.local:4566`, re-check the accepted provider findings and function compatibility in `research.md`, and update `plan.md`/`research.md` if the observed baseline has changed; stop if LocalStack-only compatibility is not established.
- [x] T002 After T001 and before writing or publishing feature manifests, follow `specs/003-static-site-hosting/quickstart.md` to create a uniquely named disposable bucket directly in LocalStack, configure index/error documents and a bucket-scoped public-read policy, upload a small page, and verify the exact website hostname through the workstation port-forward. Confirm the LocalStack service remains `ClusterIP`, no public route is introduced, and no real AWS endpoint is contacted; delete the test objects and bucket and verify cleanup before proceeding.

**Checkpoint**: The live lab and package compatibility match the plan, and the website endpoint works against LocalStack with the disposable test resources removed.

## Phase 3: User Story 1 - Declare a static site (Priority: P1)

**Goal**: Provide a namespaced `XStaticSite` API that provisions isolated S3 website hosting resources in LocalStack and reports readiness and the website URL.

**Independent Test**: After the Function and API are healthy, reconcile a temporary site declaration and verify its bucket, website configuration, public-access block, policy, `Ready=True`/`Synced=True`, and status fields without using real AWS.

### Implementation

- [x] T003 [P] [US1] Add `applications/crossplane-patch-and-transform.yaml` declaring only the pinned `xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.11.2` Crossplane Function package; include no credentials or site resources.
- [x] T004 [US1] Re-check the function package pin and Crossplane compatibility against `specs/003-static-site-hosting/research.md`, then server-side dry-run `applications/crossplane-patch-and-transform.yaml`; do not proceed if the version or compatibility evidence has changed.
- [X] T005 [US1] Publish only `applications/crossplane-patch-and-transform.yaml` through GitOps and follow `specs/003-static-site-hosting/quickstart.md`; wait for the Function to report `INSTALLED=True` and `HEALTHY=True` within the 10-minute target, and inspect package conditions/events if it does not.
- [x] T006 [US1] After T005, add `applications/crossplane-static-site-api.yaml` with the namespaced `XStaticSite` XRD and Composition. Define the `indexDocument` default and optional `errorDocument`; compose one UID-derived S3 `Bucket`, `BucketWebsiteConfiguration`, `BucketPublicAccessBlock`, and bucket-scoped `BucketPolicy` per XR; use the existing `ClusterProviderConfig/localstack` and same-controller bucket selectors; publish `status.bucketName` and the LocalStack `status.websiteURL`; preserve visible failure conditions and do not introduce a real AWS endpoint or credentials.
- [X] T007 [US1] Server-side dry-run `applications/crossplane-static-site-api.yaml` against the served Crossplane and provider APIs, correct validation errors, then publish it through GitOps only after the Function is healthy.
- [X] T008 [US1] Follow `specs/003-static-site-hosting/quickstart.md` to verify the XRD is established and the Composition is ready before creating site declarations; inspect package, XRD, Composition, and Kubernetes event conditions and stop if any required API or function is unhealthy.

**Checkpoint**: The Function is healthy and the published `XStaticSite` API and Composition are ready for site declarations.

## Phase 4: User Story 2 - Verify and operate a hosted site (Priority: P2)

**Goal**: Let users locate and test a site's LocalStack website endpoint, understand that content upload is separate from infrastructure readiness, and remove sites deliberately.

**Independent Test**: Follow the quickstart with temporary declarations: retrieve the configured index page, verify a configured error document, confirm a second site's resources are isolated, and explicitly clean up both sites.

### Implementation

- [x] T009 [P] [US2] Update `README.md` with the static-site API's purpose, its LocalStack-only/public-read-in-lab boundary, the separate content-upload model, and links to `specs/003-static-site-hosting/quickstart.md` and `research.md`.
- [x] T010 [P] [US2] Complete `specs/003-static-site-hosting/quickstart.md` with the staged Function/API/site rollout, readiness and failure inspection, example declaration, explicit LocalStack upload endpoint, website URL/port-forward checks, two-site and error-document verification, and explicit cleanup for `prune: false` and ephemeral LocalStack data.
- [X] (dropped, Principle VIII: test-only; `docs-demo` is Ready) T011 [US2] After T008, perform the end-to-end checks in `specs/003-static-site-hosting/quickstart.md` using temporary site declarations: verify all four composed resources report `Ready=True` and `Synced=True` within 10 minutes; confirm `status.bucketName`, `status.websiteURL`, and failure visibility; upload and retrieve the index document through the website URL within 5 minutes; verify the configured error document; and confirm two sites have distinct buckets and policies without cross-site changes.
- [X] (dropped, Principle VIII: test-only) T012 [US2] Verify that an invalid document setting is rejected by the XRD schema or leaves the XR visibly not ready, that provider reconciliation continues to target only LocalStack when it is unavailable, and that website access is unavailable without the documented local port-forward; then remove the temporary declarations, explicitly delete the XRs, and wait for all composed resources and buckets to be deleted.

**Checkpoint**: Site content and error handling work through the local endpoint, sites remain isolated, failures are visible, and temporary resources are fully removed.

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Ensure manifests, documentation, rollout behavior, and safety boundaries agree.

- [x] T013 Review `applications/crossplane-patch-and-transform.yaml`, `applications/crossplane-static-site-api.yaml`, `README.md`, and all files under `specs/003-static-site-hosting/`; run `git diff --check` and confirm exact package pins, LocalStack-only endpoints, no committed credentials or plaintext secrets, no public route, accurate status/readiness semantics, and explicit cleanup guidance for the root application's `prune: false`.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No tasks; the repository and platform prerequisites already exist.
- **Foundational (Phase 2)**: T001 gates T002; both must pass before feature manifests are written or published.
- **User Story 1 (Phase 3)**: Depends on Phase 2. T003 precedes T004 and T005; T005 must confirm a healthy Function before T006/T007 publish the API; T008 confirms the API is ready before any site declaration is tested.
- **User Story 2 (Phase 4)**: Depends on T008. T009 and T010 can proceed in parallel with API rollout because they touch separate documentation files; T011/T012 require the API and runbook, and T012 explicitly cleans up the temporary sites.
- **Polish (Phase 5)**: T013 follows implementation, rollout, validation, and documentation tasks.

### User Story Dependencies

- **US1 (P1)**: Depends on the live compatibility and disposable website checks in Phase 2; delivers the reusable API and Composition.
- **US2 (P2)**: Depends on US1's established API and Composition; validates the user-facing website access and lifecycle.

### Parallel Opportunities

- T003 can run alongside T009 and T010 after Phase 2 because they modify separate files and do not require the API to be published.
- T009 and T010 can run in parallel because they update different documentation files.
- Keep rollout sequential: Function package first, then XRD/Composition, then temporary site declarations. Do not publish a site declaration until T008 confirms the API and Composition are ready.

## Implementation Strategy

### MVP First

1. Complete T001 and T002 to confirm the exact lab and endpoint behavior before feature manifests are created.
2. Install and verify the pinned Function package.
3. Publish and verify the XRD and Composition.
4. Reconcile one temporary site and verify its four managed resources and reported status.

### Incremental Delivery

1. Deliver the static-site API and verify infrastructure readiness in LocalStack.
2. Add the user workflow documentation and test index/error serving, two-site isolation, and visible failure behavior.
3. Explicitly delete temporary site resources, then complete the cross-cutting safety and documentation review.

## Notes

- The root Argo CD application has `prune: false`; deleting a declaration from Git alone does not delete the live XR or its bucket.
- Site readiness covers infrastructure reconciliation, not whether site content has been uploaded.
- LocalStack is ephemeral; after restart, declared infrastructure may be reconciled again, but content must be uploaded again.
- Anonymous `s3:GetObject` is limited to each site's objects in the LocalStack lab; do not use this configuration for real AWS or expose LocalStack through an ingress, NodePort, or public route.
