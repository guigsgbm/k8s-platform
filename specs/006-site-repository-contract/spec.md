# Feature Specification: Static-Site Repository Contract

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-11-08

**Status**: Draft

**Input**: User description: "Define exactly what one site repository contains so that a tagged repository is discovered, its infrastructure is provisioned, and GitHub Actions builds and publishes the content to its LocalStack website bucket."

## Clarifications

- The existing `XStaticSite` contract generates `status.bucketName` from the resource UID, which a workflow without Kubernetes access cannot discover. The user chose to require a deterministic bucket name derived from the repository name, overriding the initial no-change-to-`k8s-platform` constraint.
- The existing Composition manages a placeholder object at the configured index-document key, which can overwrite content uploaded by the workflow. The user chose to require the platform to stop managing that placeholder object so published content remains authoritative.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a discoverable site repository (Priority: P1)

As a site owner, I want one repository to contain the site files, its `XStaticSite`, catalog metadata, and deployment workflow so that adding the discovery topic provisions an isolated website after the required platform contract dependencies are in place.

**Why this priority**: A complete, conforming repository is the input consumed by existing discovery and hosting infrastructure.

**Independent Test**: In the single manual end-to-end run, create one hand-made repository in the lab organization with the required layout and topic. Verify discovery creates its site namespace and Crossplane reports the `XStaticSite` ready with the deterministic bucket name.

**Acceptance Scenarios**:

1. **Given** a private, non-archived repository in `labs-guigsgbm` whose name matches the discovery rule and whose default branch contains the required files, **When** it receives the `backstage-static-site` topic, **Then** Argo CD discovers its `deploy/` directory and Crossplane provisions its `XStaticSite`.
2. **Given** the repository's `XStaticSite` is ready, **When** its status is inspected, **Then** `status.bucketName` is `site-<repository-name>` and `status.websiteURL` identifies that bucket's LocalStack website endpoint.
3. **Given** the repository contains the catalog descriptor, **When** Backstage reads it, **Then** it identifies a Component of type `website` whose GitHub project-slug points to the repository.

### User Story 2 - Publish the website from the repository (Priority: P1)

As a site owner, I want a push to the default branch or a manual workflow dispatch to publish the repository's static files so that the website reflects its committed content without Kubernetes access or repository secrets.

**Why this priority**: Provisioned storage does not provide a usable website until the repository content is uploaded.

**Independent Test**: In the same single manual end-to-end run, run the workflow, verify it waits for the matching bucket and completes its sync, then open the printed website URL and confirm the configured index document is served.

**Acceptance Scenarios**:

1. **Given** a push to the default branch or a manual dispatch, **When** the workflow runs on `lab-localstack`, **Then** it installs the pinned tools, waits for the repository's bucket using AWS CLI, synchronizes the site files, and prints the website URL.
2. **Given** the repository's bucket has not yet been provisioned, **When** the workflow checks for it, **Then** it retries using AWS CLI only for a bounded period and reports a clear failure if the bucket does not become available.
3. **Given** a push to a branch other than the default branch, **When** the workflow is triggered, **Then** no deployment job is allocated to a runner.
4. **Given** a deployment run, **When** its actions execute, **Then** it uses the runner-provided LocalStack endpoint and dummy AWS credentials, makes no Kubernetes API calls, and reads no repository secret.

### Edge Cases

- A repository name does not match the discovery expression; it is not a valid input to this contract and must not be generated as an undiscoverable site.
- A required file or directory is missing, or the XStaticSite name differs from the repository name; the repository is nonconforming and discovery or deployment must not silently target another site.
- `status.bucketName` does not match `site-<repository-name>`; the platform dependency is not satisfied and the workflow must fail rather than guess a different bucket.
- The bucket is not yet available or does not become available within the workflow's wait limit; the deployment reports failure and does not fall back to real AWS.
- The website has a custom index or error document; the configured document names must match the files uploaded from `src/`.
- The website content has not yet been uploaded; Crossplane readiness indicates infrastructure readiness only, not content availability.
- A workflow is dispatched manually from another branch; it follows the explicit `workflow_dispatch` trigger policy and publishes that dispatch's checked-out revision.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A conforming site repository MUST be private, non-archived, owned by `labs-guigsgbm`, carry the GitHub topic `backstage-static-site`, and have a name matching `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`.
- **FR-002**: The repository MUST contain `src/`, `deploy/xstaticsite.yaml`, `catalog-info.yaml`, `.github/workflows/deploy.yml`, and a short `README.md`. With the initial document values, `src/` MUST contain `index.html` and `error.html`; if either document placeholder is changed, its corresponding file in `src/` MUST use that value as its filename. `src/` MUST contain the publish-ready static site files; plain static files need no build step.
- **FR-003**: `deploy/xstaticsite.yaml` MUST declare exactly one `platform.k8s-platform.io/v1alpha1` `XStaticSite`, with `metadata.name` equal to the repository name and no `metadata.namespace`. It MUST set the configured index and error document names.
- **FR-004**: The discovery-name rule MUST bound the repository name so `site-<repository-name>` is a valid S3 bucket name no longer than 63 characters.
- **FR-005**: As a required platform dependency, the `XStaticSite` contract MUST set `status.bucketName` deterministically to `site-<metadata.name>` and MUST report a `status.websiteURL` for that bucket. This narrowly overrides the current UID-derived bucket name; no other discovery or runner changes are included.
- **FR-006**: As a required platform dependency, Crossplane MUST stop managing a placeholder object at the configured index-document key. GitHub Actions MUST be the owner of the uploaded website objects, and `XStaticSite` readiness MUST represent infrastructure readiness rather than content availability.
- **FR-007**: `catalog-info.yaml` MUST declare a Backstage `Component` with `spec.type: website` and the `github.com/project-slug` annotation set to `labs-guigsgbm/<repository-name>`.
- **FR-008**: `.github/workflows/deploy.yml` MUST run only for pushes to the repository's default branch and for `workflow_dispatch`. Non-default-branch pushes MUST be filtered before a job can allocate a runner. The deploy job MUST use `runs-on: lab-localstack`.
- **FR-009**: The workflow MUST install AWS CLI version `2.38.1` and Node.js version `24.11.1`. It MUST install Node.js using `actions/setup-node` pinned to a full, immutable commit SHA, not a version tag. Plain static HTML does not require a build; if a build step is added, it MUST publish its output to `dist/`, and the workflow MUST synchronize that output instead of `src/`.
- **FR-010**: Before upload, the workflow MUST derive the bucket name as `site-<repository-name>`, matching `XStaticSite.status.bucketName`, and wait for that bucket using AWS CLI commands only. The wait MUST be bounded to 10 minutes and MUST fail explicitly on timeout.
- **FR-011**: The workflow MUST use `aws s3 sync` to publish the site files to the derived bucket through the runner-provided `AWS_ENDPOINT_URL`, and MUST print the matching `XStaticSite.status.websiteURL` format: `http://site-<repository-name>.s3-website.localhost.localstack.cloud:4566/`.
- **FR-012**: The workflow MUST use the dummy `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` supplied by the runner environment. It MUST NOT use `kubectl`, a kubeconfig, Kubernetes credentials, or any cluster permission, and MUST NOT fall back to real AWS.
- **FR-013**: The repository MUST contain no secrets or credentials. The workflow MUST NOT reference GitHub repository secrets for AWS or Kubernetes access.
- **FR-014**: All repository-specific values MUST use exactly these contract placeholders wherever a concrete value is required: `__REPOSITORY_NAME__`, `__INDEX_DOCUMENT__`, and `__ERROR_DOCUMENT__`. The initial document-name values MUST be `index.html` and `error.html`. The workflow MUST derive the current repository name from GitHub Actions context rather than storing a duplicate name.
- **FR-015**: `README.md` MUST briefly describe the repository layout, the topic required for discovery, the index/error document settings, how publishing is triggered, and the locally reported website URL. It MUST state that credentials are provided by the runner and are not stored in the repository.
- **FR-016**: Validation for this feature MUST be limited to one manual end-to-end run using a hand-created repository. No additional automated, load, failure-injection, cleanup, or repeated tests are in scope.

### Key Entities *(include if feature involves data)*

- **Site repository**: A private, topic-tagged GitHub repository identified by its discovery-compliant name and containing site files, deployment declaration, catalog metadata, workflow, and README.
- **XStaticSite**: The namespaced platform resource named after the repository; its status reports the infrastructure readiness, deterministic bucket name, and website URL.
- **Published website**: The objects synchronized from the repository's static content into its isolated LocalStack bucket and served through the reported website URL.
- **Repository placeholders**: The three template-time values for repository name, index document, and error document.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In the single manual end-to-end run, one conforming hand-created repository is discovered, its XStaticSite becomes ready, and its status bucket name exactly matches `site-<repository-name>`.
- **SC-002**: In that same run, a default-branch push or manual dispatch completes publication after the bucket becomes available, and the printed website URL serves the committed index document and static assets.
- **SC-003**: The workflow's bucket wait has a maximum duration of 10 minutes and reports an explicit timeout if provisioning takes longer.
- **SC-004**: The delivered repository contains every required path, including the index and error files for the configured document values, and exactly the three listed repository-specific placeholders; no credentials or secrets are committed.
- **SC-005**: The workflow uses no Kubernetes access and sends S3 requests only to the runner-provided LocalStack endpoint.

## Assumptions

- Specs 003-005 are in effect: discovery watches non-archived repositories in `labs-guigsgbm` with the `backstage-static-site` topic and syncs `deploy/`; the runner is ephemeral, labeled `lab-localstack`, and supplies the LocalStack endpoint and dummy credentials.
- This specification explicitly overrides the initial no-change-to-`k8s-platform` constraint only for the two platform dependencies in FR-005 and FR-006. The deterministic bucket identity and removal of the managed placeholder are necessary for a workflow without cluster access to publish content reliably.
- The repository name used for the `XStaticSite`, bucket, and GitHub project-slug annotation is the GitHub repository name without the organization prefix.
- The initial site is plain static HTML and needs no build. If a later site needs a build, its output directory is `dist/`.
- The website URL is the existing LocalStack website URL format and remains internal to the lab.
- No new credentials, secrets, or runner/discovery configuration are introduced by this repository contract.

## Out of Scope

- The Backstage template and Backstage configuration.
- Changes to GitHub Actions runner configuration or repository discovery.
- Any `k8s-platform` changes other than the two required dependencies in FR-005 and FR-006.
- A build framework or dependency manifest for the initial plain-static site.
- Testing beyond the one manual end-to-end run of a hand-created repository.
