---
description: "Task list for the static-site repository contract"
---

# Tasks: Static-Site Repository Contract

**Input**: Design documents from `/specs/006-site-repository-contract/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/site-repository.md`, `quickstart.md`

**Tests**: Follow the spec's single manual end-to-end run and required server-side dry-run. No additional automated or repeated tests are in scope.

## Phase 1: Setup

**Purpose**: No project initialization is needed; use the existing `k8s-platform` GitOps repository and the hand-created site repository specified by the contract.

## Phase 2: Foundational

**Purpose**: Update the shared platform contract required by both user stories.

- [X] T001 Update `applications/crossplane-static-site-api.yaml` so the bucket external name, `status.bucketName`, and `status.websiteURL` derive from the `XStaticSite` name as `site-<name>`; remove the Composition-managed placeholder `Object` so readiness represents infrastructure only.

## Phase 3: User Story 1 - Create a discoverable site repository (Priority: P1)

**Goal**: Provide a conforming private repository whose discovery creates an `XStaticSite` with the deterministic bucket and whose catalog descriptor identifies the site.

**Independent Test**: In the one manual end-to-end run, verify the hand-created repository has the required topic and layout, discovery creates its site namespace and `XStaticSite`, and the resource becomes ready with `status.bucketName: site-<repository-name>` and the matching LocalStack website URL.

### Implementation for User Story 1

- [X] T002 [US1] Create a private, non-archived repository in `labs-guigsgbm` with a name matching `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`, add the `backstage-static-site` topic, and use its default branch for the files listed in `specs/006-site-repository-contract/contracts/site-repository.md`.
- [X] T003 [P] [US1] Add exactly one `XStaticSite` declaration at `<site-repository>/deploy/xstaticsite.yaml`, using the repository name for `metadata.name`, omitting `metadata.namespace`, and setting the index/error fields with `__INDEX_DOCUMENT__` and `__ERROR_DOCUMENT__` (`index.html` and `error.html` initially).
- [X] T004 [P] [US1] Add `<site-repository>/catalog-info.yaml` declaring a Backstage `Component` of type `website` with name `__REPOSITORY_NAME__` and project-slug `labs-guigsgbm/__REPOSITORY_NAME__`.
- [X] T005 [P] [US1] Add publish-ready static content at `<site-repository>/src/index.html` and `<site-repository>/src/error.html`, matching the configured document names; link `src/site.css` from the index as specified in `specs/006-site-repository-contract/quickstart.md`.
- [X] T006 [P] [US1] Add `<site-repository>/README.md` documenting the required layout, discovery topic, index/error settings, publication triggers, LocalStack URL, and runner-provided credentials as specified in `specs/006-site-repository-contract/contracts/site-repository.md`.

## Phase 4: User Story 2 - Publish the website from the repository (Priority: P1)

**Goal**: Publish the repository's static files to its matching LocalStack bucket on default-branch pushes or manual dispatch, without Kubernetes access or repository secrets.

**Independent Test**: In the same single end-to-end run, verify the workflow uses `lab-localstack`, waits at most 10 minutes for `site-<repository-name>`, synchronizes the configured content through the runner's LocalStack endpoint, prints the matching website URL, and serves the committed index and CSS asset.

### Implementation for User Story 2

- [X] T007 [US2] Add `<site-repository>/.github/workflows/deploy.yml` with push and `workflow_dispatch` triggers, a job-level default-branch guard, `runs-on: lab-localstack`, AWS CLI `2.38.1` user-local installation, and Node.js `24.11.1` via `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1`; derive `site-<repository-name>` from Actions context, wait for the bucket using AWS CLI only for at most 10 minutes with an explicit timeout failure, run `aws s3 sync` through `AWS_ENDPOINT_URL`, and print the contract URL without using Kubernetes, repository secrets, or real AWS.

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Align existing operating documentation with the new bucket identity and content-ownership behavior, then perform only the validation specified by the feature.

- [X] T008 [P] Update `README.md` to describe deterministic repository-derived bucket names and workflow-owned website content.
- [X] T009 [P] Update `specs/003-static-site-hosting/data-model.md` to describe deterministic bucket names and infrastructure-only readiness.
- [X] T010 [P] Update `specs/003-static-site-hosting/contracts/static-site.md` to describe the deterministic bucket and removal of the Composition-managed placeholder object.
- [X] T011 [P] Update `specs/003-static-site-hosting/quickstart.md` to explain the bucket-identity change, workflow-owned content, and destructive recreation of the old `docs-demo` bucket without migration.
- [X] T012 Run `kubectl apply --dry-run=server -f applications/crossplane-static-site-api.yaml` before applying the platform manifest, as required by `specs/006-site-repository-contract/quickstart.md`.
- [X] T013 Complete the single manual end-to-end run in `specs/006-site-repository-contract/quickstart.md`: verify discovery and deterministic `XStaticSite` status, run publication, then request the printed website URL and its `site.css` asset.

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No setup tasks; the existing repositories and platform are used.
- **Foundational (Phase 2)**: T001 must complete before either user story.
- **User Stories (Phases 3-4)**: Both depend on T001. User Story 2 uses the repository created for User Story 1 and follows its setup.
- **Polish (Phase 5)**: Documentation tasks can proceed independently; validation tasks require the implementation and documentation updates to be complete.

### User Story Dependencies

- **User Story 1 (P1)**: Starts after T001; no dependency on another user story.
- **User Story 2 (P1)**: Uses the site repository established in User Story 1; complete User Story 1 before validating publication.

### Parallel Opportunities

- After T002 creates the repository, T003-T006 edit different paths and can proceed in parallel.
- T008-T011 edit separate documentation files and can proceed in parallel.
- Do not parallelize T012/T013 with implementation changes; the dry-run precedes applying the manifest, and the single end-to-end run validates the integrated result.

## Parallel Example: User Story 1

```text
After T002:
T003: Add <site-repository>/deploy/xstaticsite.yaml
T004: Add <site-repository>/catalog-info.yaml
T005: Add <site-repository>/src/index.html, src/error.html, and src/site.css
T006: Add <site-repository>/README.md
```

## Implementation Strategy

### MVP First

1. Complete T001 to establish the required platform contract.
2. Complete User Story 1 to make a repository discoverable and provision its bucket.
3. Complete User Story 2 so the provisioned site serves its committed content; both P1 stories are needed for the useful end-to-end MVP.
4. Update the operating references and perform the one required dry-run and manual end-to-end validation.

### Incremental Delivery

1. Deliver the platform dependency and conforming site repository (User Story 1).
2. Add repository publication (User Story 2) and verify the site in the same manual run.
3. Complete documentation alignment and the specified validation; do not add further test runs.
