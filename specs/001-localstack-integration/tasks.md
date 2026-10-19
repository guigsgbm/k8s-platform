# Tasks: LocalStack for the Kubernetes Lab

**Input**: Design documents from `/specs/001-localstack-integration/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: No automated test suite is requested for this declarative GitOps feature. Validate the Helm render and run the documented acceptance scenarios against the personal Minikube lab.

**Organization**: Tasks are grouped by user story so each story can be implemented and validated as an increment.

## Format: `[ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel with other marked tasks in the same phase; different files and no incomplete-task dependency.
- **[Story]**: User story from `spec.md`.
- Every task includes its relevant repository path. Operational tasks identify the runbook they follow.

## Phase 1: Setup

**Purpose**: Repository scaffolding and platform prerequisites.

The repository already has `applications/`, a root Argo CD Application watching that directory, and the Spec Kit feature structure. No new project setup or dependencies are needed.

## Phase 2: Foundational

**Purpose**: Prepare the out-of-Git prerequisite needed before Argo CD starts LocalStack.

- [ ] T001 Replace the `--from-literal` token example in `specs/001-localstack-integration/quickstart.md` with a non-echoing prompt and stdin-based Secret creation procedure; ensure no token value is placed in shell history, process arguments, logs, or Git.
- [ ] T002 Before publishing the LocalStack Application, create the `localstack` namespace and `localstack-auth-token` Secret with key `auth-token` by following `specs/001-localstack-integration/quickstart.md`; verify the key exists without displaying its value.

**Checkpoint**: The token Secret exists in the target namespace and no secret value has entered the repository.

## Phase 3: User Story 1 - Run LocalStack through the lab's GitOps workflow (Priority: P1)

**Goal**: Install a pinned LocalStack release through the existing Argo CD reconciliation workflow, with readiness and secret handling visible to the lab owner.

**Independent Test**: Publish the Application after the token Secret is present; confirm Argo CD reports it `Synced` and `Healthy`, the pod is ready, and the pinned chart renders without a public-facing Service.

### Implementation

- [ ] T003 [P] [US1] Add `applications/localstack.yaml` as an Argo CD Application for the official `localstack` chart `0.7.1` and image `localstack/localstack:2026.10.0`; set namespace `localstack`, `ClusterIP`, `persistence.enabled: false`, the token `secretKeyRef`, planned resource requests/limits, automated sync, self-healing, deliberate pruning, and `CreateNamespace=true`.
- [ ] T004 [P] [US1] Add a LocalStack section to `README.md` describing the GitOps-managed application, pinned releases, out-of-Git token prerequisite, Argo CD status checks, and where to find the usage and cleanup runbook.
- [ ] T005 [US1] Render the chart from the values in `applications/localstack.yaml` with Helm and correct any errors; confirm the rendered image is pinned, `LOCALSTACK_AUTH_TOKEN` is sourced from the Secret, the Service is `ClusterIP` on port 4566, and persistence is disabled.
- [ ] T006 [US1] Publish `applications/localstack.yaml` to the configured Git branch and verify the root Application discovers it and LocalStack becomes ready in Minikube; record the actual status-check or troubleshooting details in `README.md` if they differ from `specs/001-localstack-integration/quickstart.md`.

## Phase 4: User Story 2 - Use the local AWS-compatible endpoint for study (Priority: P1)

**Goal**: Enable an in-cluster study client and the workstation to use LocalStack without contacting real AWS.

**Independent Test**: Follow the documented smoke tests with dummy AWS credentials; verify an in-cluster AWS CLI operation succeeds at the cluster endpoint and the workstation reaches the service through a temporary port-forward only.

### Implementation

- [ ] T007 [US2] Complete `specs/001-localstack-integration/quickstart.md` and `specs/001-localstack-integration/contracts/localstack-endpoint.md`: replace the AWS CLI image placeholder with a published fixed tag; document the cluster DNS endpoint, workstation port-forward, health check, dummy credentials, and an AWS-compatible smoke test.
- [ ] T008 [US2] Run the in-cluster smoke scenario in `specs/001-localstack-integration/quickstart.md`; verify the AWS CLI reaches `http://localstack.localstack.svc.cluster.local:4566`, returns the emulator account, and creates/lists a disposable test resource without real AWS credentials.
- [ ] T009 [US2] Run the workstation access scenario in `specs/001-localstack-integration/quickstart.md`; verify port-forward access to `localhost:4566` and confirm `kubectl get ingress,svc -n localstack` shows no ingress or `NodePort`.

## Phase 5: User Story 3 - Understand and safely reset the lab (Priority: P2)

**Goal**: Make ephemeral state and intentional cleanup consequences clear, including the root Application's non-pruning behavior.

**Independent Test**: Review the lifecycle and removal guidance in `README.md` and `specs/001-localstack-integration/quickstart.md`; using only disposable simulated data, confirm a LocalStack restart loses that data as documented.

### Implementation

- [ ] T010 [P] [US3] Document in `README.md` and `specs/001-localstack-integration/quickstart.md` that simulated resources are ephemeral and lost on pod restart, and explain that removing the manifest alone does not remove the Application because the root Application has `prune: false`; provide the intentional cascade-and-namespace cleanup sequence.
- [ ] T011 [US3] Using only a disposable simulated resource, follow the restart/reset scenario in `specs/001-localstack-integration/quickstart.md` and confirm the resource is gone after LocalStack restarts; do not cascade-delete the Application or namespace as part of this validation unless the lab owner explicitly intends to remove the installation.

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Ensure the final desired state and operator documentation agree.

- [ ] T012 Review `applications/localstack.yaml`, `README.md`, and all files under `specs/001-localstack-integration/`; run `git diff --check` and confirm there are no plaintext secrets, floating `latest` tags, or instructions that expose the endpoint beyond the local lab.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No tasks; the repository and platform prerequisites already exist.
- **Foundational (Phase 2)**: T001 must precede T002; the namespace and token Secret must exist before the LocalStack pod starts.
- **User Story 1 (Phase 3)**: Depends on Phase 2. T003 and T004 may run in parallel; T005 depends on T003; T006 depends on T002, T003, and T005.
- **User Story 2 (Phase 4)**: Requires the ready LocalStack service from T006. T007 documents the workflows before T008/T009 validate them; the workstation check follows the in-cluster check so it can inspect the disposable resource created there.
- **User Story 3 (Phase 5)**: Requires the service from T006. T010 precedes T011 because the reset check follows the updated lifecycle guidance; T011 uses disposable data and must not remove the actual installation.
- **Polish (Phase 6)**: T012 follows the story tasks and the resulting documentation/configuration changes.

### User Story Dependencies

- **US1 (P1)**: Depends on the foundational Secret preparation; delivers the deployed LocalStack service.
- **US2 (P1)**: Depends on US1 being ready so its endpoint and smoke tests can be exercised.
- **US3 (P2)**: Depends on US1 being ready; otherwise independent of US2 if it creates its own disposable resource.

### Parallel Opportunities

- T003 and T004 can be implemented in parallel because they modify different files and neither depends on the other's output.
- T011 is an operational validation against disposable data and does not edit repository files.
- Do not run T008 and T009 concurrently: T009 may inspect the disposable resource created by T008.

## Implementation Strategy

### MVP First

1. Complete Phase 2: prepare the namespace and token Secret outside Git.
2. Complete Phase 3: add the Argo CD Application, render it, and verify LocalStack is ready.
3. Stop and validate the deployed service before proceeding to client exercises.

### Incremental Delivery

1. Deliver US1 to manage and observe LocalStack through GitOps.
2. Deliver US2 to use the local endpoint from a study workload and the workstation.
3. Deliver US3 to make ephemeral state and intentional cleanup safe and understandable.
4. Complete Phase 6 to ensure documentation, desired state, and secret-handling guidance agree.

## Notes

- No LocalStack token belongs in Git. The only Git-managed token configuration is the reference to the out-of-band Secret.
- `persistence.enabled: false` is intentional: restarting LocalStack resets simulated resources.
- T011's restart loses simulated state by design; only use disposable resources for that check.
- `git diff --check` and Helm rendering are configuration validation; no automated application test suite is introduced.
