# Tasks: Automatic Static-Site Discovery

**Input**: Design documents from `/specs/004-static-site-discovery/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/discovery.md](./contracts/discovery.md), [quickstart.md](./quickstart.md)

**Tests**: No automated test suite is requested for this declarative GitOps feature. Validate manifests with server-side dry-run, then run the staged checks in [quickstart.md](./quickstart.md) against the live lab.

**Organization**: Tasks are grouped by user story. Manifests are copied verbatim from [contracts/discovery.md](./contracts/discovery.md); `<ORG>` is replaced by the GitHub organization name from T002.

## Format: `[ID] [P?] [Story] Description with file path`

- **[P]**: Can run in parallel with other marked tasks in the same phase when files and prerequisites do not overlap.
- **[Story]**: User story from `spec.md` (US1 tag and deploy, US2 convergence, US3 guardrails, US4 visibility, US5 decommission).
- Tasks that commit or push to `main`, create GitHub topics or repositories, or create cluster objects are operational and need the user's go-ahead.

## Phase 1: Setup

No project setup or new dependencies are needed: `applications/`, the root Argo CD application, Crossplane, LocalStack and the `XStaticSite` API already exist, and this feature adds no package or image to pin.

## Phase 2: Foundational

**Purpose**: Confirm the live lab and the inputs the manifests need before writing them.

- [X] T001 Follow "Prerequisites" 1–3 in `specs/004-static-site-discovery/quickstart.md`: confirm the Minikube lab context; Argo CD `v3.5.5` with `argocd-applicationset-controller` running; `docs-demo` carries `argocd.argoproj.io/tracking-id`; `xstaticsites` Ready; `validatingadmissionpolicies.admissionregistration.k8s.io` is served.
- [ ] T002 Obtain from the user the GitHub **organization** name `<ORG>` (a personal account cannot be scanned) and confirm `curl -s -o /dev/null -w '%{http_code}\n' https://api.github.com/orgs/<ORG>/repos` prints `200`. Prepare, with the user, a public canary repository in `<ORG>` with a lowercase name matching `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$` and `deploy/xstaticsite.yaml` (kind `XStaticSite`, no `metadata.namespace`), without the topic yet.

**Checkpoint**: Environment matches the plan, `<ORG>` is known and a canary repository exists.

## Phase 3: User Story 1 - Tag a repository and get a site (Priority: P1) 🎯 MVP

**Goal**: Tagging a repository of `<ORG>` produces an Application `site-<repo>` and a Ready `XStaticSite` in namespace `site-<repo>`, with no commit to this repository.

**Independent Test**: Quickstart Stage 3 steps 1–3: tag the canary, then within the latency target see the Application Synced/Healthy and `XStaticSite` `READY=True`.

- [X] T003 [P] [US1] Create `applications/static-sites-project.yaml` from `contracts/discovery.md`: AppProject `static-sites` in `argocd`; `sourceRepos` `https://github.com/<ORG>/*`; destination `https://kubernetes.default.svc` namespace `site-*`; `clusterResourceWhitelist` only `Namespace` named `site-*`; `namespaceResourceWhitelist` only `platform.k8s-platform.io/XStaticSite`.
- [X] T004 [P] [US1] Create `applications/static-site-discovery.yaml` from `contracts/discovery.md`: ApplicationSet `static-site-discovery` with the `argocd.argoproj.io/sync-options: Prune=false` annotation and finalizer `resources-finalizer.argocd.argoproj.io`; `goTemplate` with `missingkey=error`; `syncPolicy` `preserveResourcesOnDeletion: true` and `applicationsSync: create-update`; `scmProvider.github` with `organization: <ORG>`, `excludeArchivedRepos: true`, `cloneProtocol: https`, `requeueAfterSeconds: 180`, filter `repositoryMatch: '^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$'` and `labelMatch: '^backstage-static-site$'`; the commented optional `tokenRef` line; template `site-{{.repository}}` to project `static-sites`, `source.path: deploy`, `targetRevision: HEAD`, destination namespace `site-{{.repository}}`, `automated` `selfHeal: true` and `prune: true`, `CreateNamespace=true`, and no deletion finalizer. Do not add a `pathsExist` filter and do not store any token.
- [X] T005 [US1] Run `kubectl apply --dry-run=server -f` on both new files (after T003 and T004) and review YAML, source, destination, pruning and deletion effects per constitution principle VI.
- [ ] T006 [US1] After T005 and user approval, commit and push `applications/static-sites-project.yaml` first (quickstart Stage 1 step 2 checks), then `applications/static-site-discovery.yaml` (Stage 2 checks). Verify the root application `k8s-platform` stays Synced/Healthy and the ApplicationSet reports no `ErrorOccurred=True`.
- [ ] T007 [US1] Run quickstart Stage 3 steps 1–3: add the topic `backstage-static-site` to the canary repository; verify `site-<repo>` Synced/Healthy and `kubectl -n site-<repo> get xstaticsites` shows `SYNCED=True` and `READY=True` within 5 minutes, with zero commits to this repository. Also verify a same-topic repository outside `<ORG>` is never discovered.

**Checkpoint**: MVP complete: a tagged repository becomes a running site automatically.

## Phase 4: User Story 2 - Converge on manifest changes (Priority: P1)

**Goal**: Changes in the site repository reach the cluster automatically, drift is reverted, and removed manifests are pruned.

**Independent Test**: Quickstart Stage 3 steps 4–5 on the canary.

- [ ] T008 [US2] Run quickstart Stage 3 step 4: change `errorDocument` in the canary's `deploy/xstaticsite.yaml` and confirm the live `XStaticSite` converges without manual refresh; edit the live resource by hand and confirm it is reverted (self-heal).
- [ ] T009 [US2] Run quickstart Stage 3 step 5: remove the manifest from `deploy/`, confirm the resource is pruned (bucket deleted), and restore the manifest; record the observed behavior in `specs/004-static-site-discovery/quickstart.md` if it differs.

**Checkpoint**: Convergence, self-heal and pruning behave as in FR-006.

## Phase 5: User Story 3 - Guardrails limit what a repository can deploy (Priority: P1)

**Goal**: A repository can deploy only an `XStaticSite` in its own namespace; every violation is rejected and visible.

**Independent Test**: Quickstart Stage 4 guardrail rows against a throwaway tagged repository.

- [X] T010 [US3] Create `applications/static-site-namespace-policy.yaml` from `contracts/discovery.md`: `ValidatingAdmissionPolicy` and `ValidatingAdmissionPolicyBinding` both named `static-site-own-namespace`; `failurePolicy: Fail`; match `platform.k8s-platform.io` `xstaticsites` on CREATE and UPDATE; `matchConditions` `object.metadata.namespace.startsWith('site-')`; validation denying a present `argocd.argoproj.io/tracking-id` that does not start with `<namespace>:` (guarded with `has(object.metadata.annotations)`); binding `validationActions: [Deny]`.
- [ ] T011 [US3] Run `kubectl apply --dry-run=server -f applications/static-site-namespace-policy.yaml`, then after user approval commit and push it; verify `kubectl get validatingadmissionpolicy static-site-own-namespace` and that `docs-demo` in `static-sites` still reconciles (the policy must not match it).
- [ ] T012 [US3] Run the Stage 4 guardrail tests from `specs/004-static-site-discovery/quickstart.md` on a throwaway tagged repository: a `Secret`/`ConfigMap`, a `ClusterRole`, an `XStaticSite` with `metadata.namespace: default`, and one with `metadata.namespace: site-<other>`. Confirm each is rejected, nothing is applied, and the message is visible on the Application within 2 minutes. If the auto-created namespace is blocked by the project, fix `applications/static-sites-project.yaml` and record it in `specs/004-static-site-discovery/research.md`.

**Checkpoint**: SC-003 holds.

## Phase 6: User Story 4 - See what is discovered and why not (Priority: P2)

**Goal**: An operator lists discovered sites and identifies the cause of each non-discovery case.

**Independent Test**: Quickstart Stage 4 failure rows, each diagnosed with the documented commands in under 5 minutes.

- [ ] T013 [US4] Run the Stage 4 failure rows from `specs/004-static-site-discovery/quickstart.md`: tagged repository with no `deploy/`, an invalid manifest (`indexDocument: ""`), an invalid repository name (for example `My_Site`), wrong topic, and the rate-limit check; confirm each cause is identifiable with the Troubleshooting commands and correct the quickstart wording where it is not.
- [X] T014 [US4] Update `README.md` with a "Static-site discovery" section: the five non-discovery reasons and where each is visible (from `data-model.md`), the list/diagnose commands from `contracts/discovery.md`, the namespace/name rule `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`, the organization-only requirement, the 1 + N request cost with the rate-limit table (60/hour unauthenticated, 5000/hour authenticated) and its mitigations (token created outside Git, longer `requeueAfterSeconds`), and a link to the quickstart.

**Checkpoint**: FR-010 and FR-015 are documented and verified.

## Phase 7: User Story 5 - Decommission a site deliberately (Priority: P2)

**Goal**: Untagging, archiving or deleting never deletes anything; a documented procedure removes a site.

**Independent Test**: Quickstart Stage 4 retention and survival rows, then the removal procedure on the canary.

- [ ] T015 [US5] Run the Stage 4 "Retention" and "Survival" rows: untag, archive, then delete a throwaway repository and confirm Application, namespace, `XStaticSite` and bucket remain; restart `argocd-applicationset-controller`; run `kubectl -n argocd delete applicationset static-site-discovery` and confirm Applications and sites survive; re-sync the root application and confirm the ApplicationSet returns. If Applications do not survive, fix finalizer, annotation or policy in `applications/static-site-discovery.yaml`.
- [ ] T016 [US5] Run "Deliberate removal of a site" from `specs/004-static-site-discovery/quickstart.md` on the canary or a throwaway site and confirm the Application, `XStaticSite`, bucket and namespace are gone; fix the quickstart steps where reality differs.
- [X] T017 [US5] Update `README.md` with a "Removing a static site" subsection: the four removal steps, that repository lifecycle events delete nothing, that deleting a manifest from `deploy/` or from `applications/` deletes the resource under `prune: true`, and that bucket contents are irrecoverable.

**Checkpoint**: SC-004 and SC-006 hold.

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T018 [P] Run quickstart Stage 5 once more than one repository is tagged (or a private repository is needed): create the Secret outside Git with the documented `read -rs` command, uncomment `tokenRef` (name and key only) in `applications/static-site-discovery.yaml`, commit, and confirm no `ErrorOccurred` condition.
- [X] T019 [P] Search this repository (`applications/`, `bootstrap/`, `README.md`, `specs/004-static-site-discovery/`) for tokens or credentials (for example `ghp_`, `github_pat_`, `token:` values) and confirm none exist (SC-007).
- [X] T020 Review all three new manifests against the constitution: pruning enabled, no resources finalizer on generated Applications, project limits, no new unpinned dependency, no public route, no real AWS; confirm the root application `k8s-platform` is Synced/Healthy.
- [ ] T021 Delete throwaway repositories, namespaces, Applications and Secrets created during testing; keep only intended sites; confirm `git status` shows only the intended files.

## Dependencies & Execution Order

- Phase 2 (T001, T002) blocks everything; T002 needs user input.
- US1 (T003–T007) is the MVP. T003 and T004 are parallel; T005 follows both; T006 follows T005; T007 follows T006.
- US2 (T008, T009) needs US1 complete.
- US3 (T010–T012) needs US1's project and ApplicationSet live; T010 can be drafted in parallel with T007 but is pushed after it.
- US4 (T013, T014) needs US1; T013 benefits from US3's rejection messages. T014 and T017 edit the same file, so they run in sequence.
- US5 (T015–T017) needs US1 and US3; T016 runs after T015.
- Polish after the desired stories; T018 and T019 are parallel.

## Parallel Example: User Story 1

```text
T003 applications/static-sites-project.yaml
T004 applications/static-site-discovery.yaml
```

## Implementation Strategy

- **MVP**: Phase 2 plus US1, stopping at the T007 checkpoint with one canary site.
- **Then**: US3 immediately after the MVP, because automatic deployment without guardrails should not stay live for long; then US2, US4, US5.
- Roll out in the staged order of the quickstart so each manifest is verified before the next is published.
