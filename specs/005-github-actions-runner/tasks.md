---
description: "Implementation tasks for in-cluster GitHub Actions runners"
---

# Tasks: In-Cluster GitHub Actions Runners

**Input**: Design documents from `specs/005-github-actions-runner/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/runner-pool.md](./contracts/runner-pool.md), [quickstart.md](./quickstart.md)

**Tests**: No automated test suite is requested. The plan and success criteria require chart rendering, server-side dry-runs, staged rollout checks, and manual runner/cleanup verification from [quickstart.md](./quickstart.md).

**Organization**: Tasks are grouped by user story so each increment can be implemented and verified independently. The delivered feature includes no workflow file; job checks use a temporary workflow in a non-public repository in the organization.

## Format: `[ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel with other marked tasks in the same phase when files and prerequisites do not overlap.
- **[Story]**: User story from `spec.md` (US1–US4).
- Tasks that publish manifests, change GitHub settings, run jobs, or remove cluster resources require operator approval before those actions.

## Path Conventions

This is a Kubernetes platform GitOps repository. Application manifests live in `applications/`; feature operations and verification live in `specs/005-github-actions-runner/`.

## Phase 1: Setup

No project initialization or dependencies are needed. Reuse the existing root Argo CD application, `applications/`, and chart delivery through Argo CD.

## Phase 2: Foundational

**Purpose**: Confirm shared cluster and GitHub prerequisites before implementing or syncing the runner applications.

- [X] T001 [P] Confirm the Minikube context, healthy Argo CD root application, healthy LocalStack ClusterIP service, and node outbound access to GitHub and GHCR using the prerequisites in `specs/005-github-actions-runner/quickstart.md`.
- [X] T002 [P] Confirm the existing `Default` runner group in organization `labs-guigsgbm` allows all repositories except public repositories, and require approval for fork and outside-collaborator workflows as described in `specs/005-github-actions-runner/quickstart.md`. This intentionally allows all non-public organization repositories to use the pool.
- [X] T003 Create namespace `arc-runners` and the `arc-github-token` and `lab-aws-credentials` Secrets outside Git, following `specs/005-github-actions-runner/quickstart.md`; use a fine-grained token owned by `labs-guigsgbm` with organization permissions **Self-hosted runners: Read and write** and **Administration: Read**, and only the lab's dummy AWS values. Every credential field exists only in a Secret created outside Git (FR-003, FR-010, FR-015); do not put values in repository files, command history, logs, or workflow files.

**Checkpoint**: Cluster prerequisites, runner-group restrictions, namespace, and out-of-Git Secrets are ready.

## Phase 3: User Story 1 - Install runners through GitOps and see them register (Priority: P1) 🎯 MVP

**Goal**: The root Argo CD application installs the controller and pool in order, and the organization shows a registered, idle pool without manual runner registration.

**Independent Test**: With the prerequisites in place, sync the two Applications and verify both become Healthy in order, the pool registers and is idle within 10 minutes, and no runner pod exists while idle.

### Implementation for User Story 1

- [X] T004 [P] [US1] Create `applications/arc-controller.yaml` as Application `arc-controller` in `argocd`, using controller chart `gha-runner-scale-set-controller` version `0.15.1` from `ghcr.io/actions/actions-runner-controller-charts`, destination namespace `arc-systems`, `flags.watchSingleNamespace: arc-runners`, `flags.logLevel: info`, requests `50m` CPU / `64Mi` memory and memory limit `256Mi`; enable automated sync and self-heal, set `prune: false`, `CreateNamespace=true` and `ServerSideApply=true`, add the `Prune=false` annotation, omit the resources finalizer, and comment the pruning exception and removal order.
- [X] T005 [P] [US1] Create `applications/arc-runner-set.yaml` as Application `arc-runner-set` in `argocd`, using pool chart `gha-runner-scale-set` version `0.15.1`, wave `"1"`, destination namespace `arc-runners`, and the values in `specs/005-github-actions-runner/contracts/runner-pool.md`: organization URL `https://github.com/labs-guigsgbm`, group `Default`, label/name `lab-localstack`, Secret `arc-github-token`, `minRunners: 0`, `maxRunners: 2`, controller ServiceAccount `arc-systems/arc-controller-gha-rs-controller`, pinned runner image `ghcr.io/actions/actions-runner:2.338.2`, non-root uid `1001`, RuntimeDefault seccomp, no privilege escalation or capabilities, no mounted ServiceAccount token, requests `250m` CPU / `512Mi` memory and memory limit `2Gi`, and LocalStack endpoint/region plus dummy AWS Secret references. Set automated sync and self-heal, `prune: false`, the `Prune=false` annotation, `CreateNamespace=true`, `ServerSideApply=true`, bounded retry backoff, and no resources finalizer; do not store credentials in the manifest. The pool must be in the namespace the controller watches (`arc-runners`). `maxRunners` is at least `minRunners`; `minRunners` is 0 or more (FR-005).
- [X] T006 [US1] Render both pinned charts and server-side dry-run the CRDs and `applications/arc-controller.yaml` plus `applications/arc-runner-set.yaml`; verify the controller renders no ClusterRole or ClusterRoleBinding, the runner image and security/resource values match the contract, and each Application has the required no-prune settings before any live sync.
- [ ] T007 [US1] After operator approval, publish and sync `applications/arc-controller.yaml` and `applications/arc-runner-set.yaml`; verify the controller and pool become Healthy in order, four `actions.github.com` CRDs exist, the organization lists `lab-localstack` as registered and idle within 10 minutes, and `arc-runners` has no runner pod while idle.

**Checkpoint**: MVP complete: the root application installs a registered, idle runner pool through the ordered GitOps applications.

## Phase 4: User Story 2 - Run a clean ephemeral job against LocalStack (Priority: P1)

**Goal**: A workflow in a non-public organization repository can request the documented label, use a fresh runner for one job, reach LocalStack with dummy credentials, and leave no runner or job state behind.

**Independent Test**: Dispatch a temporary guarded job from a non-public repository in the organization, verify its S3 request succeeds through the in-cluster endpoint, then verify the runner pod is removed within 2 minutes. Repeat 10 jobs and run two concurrently to verify fresh, isolated runners.

### Implementation for User Story 2

- [ ] T008 [US2] Update `specs/005-github-actions-runner/quickstart.md` with the temporary `workflow_dispatch` job procedure using `runs-on: lab-localstack`, the trigger/default-branch guard from `specs/005-github-actions-runner/contracts/runner-pool.md`, the LocalStack endpoint and dummy credentials, AWS CLI `2.38.1` installation under the runner user's home, and a commit-SHA-pinned `actions/setup-node` reference with a fixed Node.js version; explicitly state that no workflow file is delivered or committed by this feature.
- [ ] T009 [US2] Run the temporary job from a non-public repository in the organization using `specs/005-github-actions-runner/quickstart.md`; verify it reaches LocalStack without port-forward or Kubernetes access, each of 10 consecutive jobs uses a fresh runner, concurrent jobs use separate pods, no credential appears in logs, and each runner pod disappears within 2 minutes after its job.
- [ ] T010 [US2] Using the access-control checks in `specs/005-github-actions-runner/quickstart.md`, verify a public repository, a repository outside the organization, a fork, and a pull-request trigger are never executed on this pool; confirm private organization repositories can use it and only default-branch pushes and manual dispatch are allowed by the documented guard.

**Checkpoint**: An allowed workflow can use the runner safely, and untrusted or unauthorized work cannot use it.

## Phase 5: User Story 3 - See runner state and diagnose failures (Priority: P2)

**Goal**: The operator can identify pool registration, runner counts, recent failure causes, and why jobs remain queued without exposing credentials.

**Independent Test**: Use only `specs/005-github-actions-runner/quickstart.md` to diagnose each documented queue/failure cause in under 10 minutes without reading a Secret value.

### Implementation for User Story 3

- [ ] T011 [US3] Complete the troubleshooting section in `specs/005-github-actions-runner/quickstart.md` with commands and expected signals for Argo CD Application health, the AutoscalingRunnerSet and ephemeral runner resources, listener/controller pods and logs, and GitHub runner-group status; distinguish wrong label, public repository or repository outside the organization, maximum capacity, missing/expired/revoked credential, GitHub unavailability, missing CRDs, and node resource pressure, and document recovery without printing Secret values.
- [ ] T012 [US3] Follow `specs/005-github-actions-runner/quickstart.md` to exercise the five queue/failure causes (wrong label, public or outside-organization repository, exhausted capacity, invalid credential, and GitHub unreachable); verify each cause is diagnosable in under 10 minutes, no credential value appears in logs, and restoring connectivity or the out-of-Git Secret recovers registration without a Git edit.

**Checkpoint**: The documented diagnostics distinguish the required causes and support safe recovery.

## Phase 6: User Story 4 - Scale to zero and remove the feature safely (Priority: P2)

**Goal**: The operator can pause or scale the pool to zero and deliberately remove the runner feature without Argo CD pruning it or affecting LocalStack and sites.

**Independent Test**: Verify zero idle runner pods while the pool remains registered, then follow the approved removal procedure and verify the feature's resources and GitHub registration are gone while LocalStack and sites remain healthy.

### Implementation for User Story 4

- [ ] T013 [US4] Complete the pause and deliberate-removal sections in `specs/005-github-actions-runner/quickstart.md`: stop new jobs and drain running jobs, remove the pool before the controller, then remove CRDs, out-of-Git Secrets, namespaces, and GitHub registrations; include the stuck-cleanup-finalizer recovery and verify both Applications retain `prune: false`. The removal order is: pool, then controller, then CRDs, Secrets and namespaces (FR-016).
- [ ] T014 [US4] After explicit operator approval in a suitable lab window, follow the pause/removal procedure in `specs/005-github-actions-runner/quickstart.md`; verify no runner pod, pool registration, Secret, CRD, or `arc-runners`/`arc-systems` namespace remains and that LocalStack and existing sites remain healthy.

**Checkpoint**: The feature can be paused and removed only through the documented, ordered procedure.

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Make the feature discoverable from the repository documentation and verify the final change remains within scope.

- [X] T015 Update `README.md` with a concise runner-platform overview, the two Application manifest paths, the out-of-Git credential prerequisite, the deliberate no-prune lifecycle warning, and a link to `specs/005-github-actions-runner/quickstart.md`; state that this feature delivers no site workflow.
- [ ] T016 Review `applications/arc-controller.yaml`, `applications/arc-runner-set.yaml`, `README.md`, and `specs/005-github-actions-runner/quickstart.md`; run YAML and whitespace validation, search tracked changes for credential values or real AWS credentials, and confirm no LocalStack, Crossplane, discovery, `XStaticSite`, or workflow files were changed.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No project initialization or dependency changes are needed.
- **Foundational (Phase 2)**: Complete T001–T003 before implementing or syncing the runner applications.
- **User Stories (Phases 3–6)**: US1 depends on the foundation. US2, US3, and US4 each depend on the working US1 pool; US3 and US4 do not depend on US2's job workflow checks.
- **Polish (Phase 7)**: Complete after the selected user stories and their quickstart changes.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2. T004 and T005 can be authored in parallel; T006 requires both manifests and T007 follows successful preflight.
- **US2 (P1)**: Starts after US1. It validates runner behavior through a temporary workflow only; it does not add a repository workflow file.
- **US3 (P2)**: Starts after US1. It relies on the deployed controller and pool, but not on completion of US2.
- **US4 (P2)**: Starts after US1. It relies on the deployed controller and pool, but not on completion of US2 or US3.

### Parallel Opportunities

- T001 and T002 can proceed in parallel: cluster readiness and GitHub organization setup are independent.
- T004 and T005 can proceed in parallel because they create distinct manifests; the manifests must still encode the controller-before-pool rollout order.
- US2, US3, and US4 are independently testable after US1, but their quickstart edits and live-cluster operations should be performed sequentially to avoid overlapping documentation changes or disrupting shared runner resources.

### Parallel Example: User Story 1

```text
In parallel: T004 creates applications/arc-controller.yaml
             T005 creates applications/arc-runner-set.yaml
Then:        T006 renders and validates both; T007 performs the approved live rollout
```

### Parallel Example: User Story 2

No safe same-story parallel work is identified: the quickstart procedure must be finalized before running its temporary workflow checks.

### Parallel Example: User Story 3

No safe same-story parallel work is identified: diagnostic documentation and the failure checks use the same live pool and quickstart.

### Parallel Example: User Story 4

No safe same-story parallel work is identified: pause and cleanup are sequential operations against shared cluster resources.

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 2 prerequisites.
2. Create and preflight the two Applications in Phase 3.
3. Perform the approved staged sync and verify registration and idle state.
4. Stop at the US1 checkpoint if only the registered pool MVP is needed; do not use it for jobs until US2's workflow-use and isolation checks are complete.

### Incremental Delivery

1. Complete prerequisites and US1 to install and register the ordered controller/pool.
2. Complete US2 to verify the hardened ephemeral runner and LocalStack access.
3. Complete US3 to validate operational visibility and recovery.
4. Complete US4 to validate pausing and deliberate cleanup.
5. Finish repository documentation and scope/security checks.
