# Tasks: Manage LocalStack S3 Buckets with Crossplane

**Input**: Design documents from `/specs/002-crossplane-s3-localstack/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: No automated test suite is requested for this declarative GitOps feature. Validate the manifests with server-side dry-run after their CRDs are served, then follow the staged rollout and LocalStack smoke test in `specs/002-crossplane-s3-localstack/quickstart.md`.

**Organization**: Tasks are grouped by user story so the end-to-end capability can be implemented and validated as an increment.

## Format: `[ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel with other marked tasks in the same phase; different files and no incomplete-task dependency.
- **[Story]**: User story from `spec.md`.
- Operational tasks identify the runbook they follow.

## Phase 1: Setup

**Purpose**: Repository scaffolding and project prerequisites.

The `applications/` directory, root Argo CD Application, Crossplane, LocalStack, and Spec Kit feature structure already exist. No new project setup or dependencies are needed.

## Phase 2: Foundational

**Purpose**: Reconfirm the lab and compatibility gate before publishing provider desired state.

- [x] T001 Confirm the target lab still runs Crossplane `v2.4.2` and LocalStack is reachable at `http://localstack.localstack.svc.cluster.local:4566`; compare with the compatibility evidence in `specs/002-crossplane-s3-localstack/research.md` and update `plan.md`/`research.md` before proceeding if either has changed or compatibility is no longer established.

**Checkpoint**: The target cluster and the planned provider compatibility still match the researched baseline.

## Phase 3: User Story 1 - Declare an S3 bucket for LocalStack (Priority: P1)

**Goal**: Install the pinned AWS S3 provider, configure it to use only LocalStack, and manage a bucket through the existing GitOps workflow.

**Independent Test**: Reconcile the provider packages first, then the provider configuration and bucket; confirm provider health, Crossplane `Ready=True` and `Synced=True`, and visibility of the bucket through the LocalStack endpoint using dummy credentials only.

### Implementation

- [x] T002 [P] [US1] Add `applications/crossplane-aws-s3.yaml` with Crossplane `Provider` declarations for `provider-family-aws` and `provider-aws-s3`, both pinned to `v2.4.0`; include no credentials or provider configuration resources in this first-stage manifest.
- [x] T003 [P] [US1] Update `README.md` with the AWS S3 provider's pinned, LocalStack-only purpose, its accepted lab-only security caveat, and links to `specs/002-crossplane-s3-localstack/quickstart.md` and `research.md`.
- [x] T004 [P] [US1] Update `specs/002-crossplane-s3-localstack/quickstart.md` to document the two-stage rollout, provider/API readiness checks, out-of-Git dummy Secret prerequisite, provider and bucket status/error inspection, explicit LocalStack smoke test, ephemeral state, and cleanup when root pruning is disabled.
- [x] T005 [US1] Before publishing the provider manifest, confirm both package references remain exact-pinned to `v2.4.0`, re-check their compatibility and Marketplace security findings against the authoritative sources in `specs/002-crossplane-s3-localstack/research.md`, record any changed findings, and server-side dry-run `applications/crossplane-aws-s3.yaml`; stop if the version is incompatible or its accepted risk has changed.
- [x] T006 [US1] Publish only the provider package stage and follow `specs/002-crossplane-s3-localstack/quickstart.md`: wait for Argo CD reconciliation, confirm both Crossplane Providers are installed and healthy, and verify the provider APIs are served before proceeding; inspect provider conditions/events and stop if installation fails or exceeds the 10-minute target.
- [x] T007 [US1] After T006, create the `localstack-aws-credentials` Secret in `crossplane-system` outside Git by following `specs/002-crossplane-s3-localstack/quickstart.md`; use only the LocalStack dummy credentials and verify the Secret key without displaying its value.
- [x] T008 [US1] Prepare `applications/localstack-s3-bucket.yaml` with the LocalStack `ClusterProviderConfig` and namespaced `localstack-s3-smoke` Bucket in `crossplane-system`; use the v2.4.0 CRD API versions, route S3, S3 Control, and STS to the exact LocalStack endpoint, enable path-style S3, reference the out-of-Git Secret, and declare the intended deletion behavior. Publish only after T006 and T007.
- [x] T009 [US1] After the provider APIs and Secret are ready, server-side dry-run `applications/localstack-s3-bucket.yaml`, then publish the second stage; verify Argo CD reconciliation and inspect ProviderConfig/Bucket conditions and events, confirming failures remain visible and reconciliation does not fall back to real AWS.
- [x] T010 [US1] Follow the LocalStack smoke test in `specs/002-crossplane-s3-localstack/quickstart.md` with the AWS CLI, dummy credentials, and the explicit internal endpoint; verify `localstack-s3-smoke` is listed and its Crossplane resource remains `Ready=True` and `Synced=True` on subsequent observations, within the 10-minute target and without contacting real AWS.

**Checkpoint**: The pinned provider is healthy, and the Git-declared bucket is ready and visible in LocalStack without real AWS credentials or resources.

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Ensure desired state, operator guidance, and safety constraints agree.

- [x] T011 Review `applications/crossplane-aws-s3.yaml`, `applications/localstack-s3-bucket.yaml`, `README.md`, and all files under `specs/002-crossplane-s3-localstack/`; run `git diff --check` and confirm exact provider pins, LocalStack-only endpoints, no committed credentials or plaintext secrets, no public exposure, and accurate cleanup guidance for the root application's `prune: false`.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No tasks; repository and platform prerequisites already exist.
- **Foundational (Phase 2)**: T001 gates all rollout work.
- **User Story 1 (Phase 3)**: Depends on T001. T002, T003, and T004 can proceed in parallel. T005 depends on T002; T006 depends on T005; T007 depends on T006; T008 depends on T007; T009 depends on T008; T010 depends on T009.
- **Polish (Phase 4)**: T011 follows the implementation, rollout, validation, and documentation tasks.

### User Story Dependencies

- **US1 (P1)**: Depends on the existing Crossplane and LocalStack installations and the compatibility gate in T001. The provider package stage must become healthy before its APIs, Secret, provider configuration, and Bucket are used.

### Parallel Opportunities

- T002, T003, and T004 touch separate files and can be prepared in parallel after T001.
- Keep rollout stages sequential: the second-stage manifest may be prepared in the worktree, but do not publish it until T006 confirms the provider APIs are served and T007 creates the Secret.

## Implementation Strategy

### MVP First

1. Complete T001 and publish the pinned provider packages.
2. Wait for healthy providers and served APIs before creating the out-of-Git dummy Secret.
3. Publish the LocalStack provider configuration and Bucket, then verify readiness and visibility through the LocalStack endpoint.

### Incremental Delivery

1. Reconcile only the provider packages and confirm package health.
2. Reconcile the LocalStack provider configuration and sample Bucket as a separate GitOps stage.
3. Complete the documented smoke test and review the endpoint, secret-handling, and non-pruning cleanup guidance.

## Notes

- The selected `v2.4.0` provider release has documented Marketplace vulnerability findings and a custom-endpoint trade-off; re-check and record them before publishing as required by T005.
- The AWS credentials Secret contains only dummy LocalStack credentials and must remain outside Git.
- Do not add a real AWS account, credentials, or public LocalStack endpoint as part of this feature.
- Removing the Bucket manifest alone does not delete its Kubernetes resource because the root Argo CD Application has `prune: false`; follow the explicit cleanup procedure in `quickstart.md`.
- T006-T010 completed through the staged GitOps rollout. The provider packages are healthy, the out-of-Git Secret contains the expected key, and the LocalStack Bucket is `Ready=True` and `Synced=True`; the AWS CLI smoke test lists it at the explicit in-cluster endpoint.
