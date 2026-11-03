# Feature Specification: Automatic Static-Site Discovery

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-11-02

**Status**: Draft

**Input**: User description: "Automatic discovery and deployment of static-site repositories by Argo CD. When a GitHub repository is marked as a static site (via the topic `backstage-static-site`), Argo CD must automatically create an Application that syncs that repository's `deploy/` directory into a dedicated namespace, so the XStaticSite declared there is reconciled by Crossplane. Removing the marker must not silently destroy infrastructure."

## Clarifications

### Session 2026-11-02

- Q: When a site's repository is untagged, archived or deleted, should its Application stay and keep syncing, or be removed? → A: It stays and keeps auto-syncing; it is never removed automatically.
- Q: How should a site's namespace be named so two repository names can never collide? → A: Reject repositories whose names are not already valid; no normalization (prefix + unmodified name).
- Q: Which version of each site repository should the cluster follow? → A: The repository's default branch.
- Q (planning): The built-in discovery lists only organizations; the lab owner is a personal account. Use the built-in discovery with an organization, or add a lookup service for the user account? → A: Built-in discovery; the owner must be an organization.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Tag a repository and get a site (Priority: P1)

As a platform engineer, I tag a repository with the topic `backstage-static-site` and, without any change to this repository, see an Application appear, sync and become Healthy, with the `XStaticSite` declared under the repository's `deploy/` directory reported Ready in its own namespace.

**Why this priority**: This is the core capability; it removes the need for a platform commit per new site.

**Independent Test**: Tag one repository of the configured owner that contains a valid `XStaticSite` manifest under `deploy/`, make no commit to this repository, and verify within the latency target that the Application is Healthy and the `XStaticSite` is Ready.

**Acceptance Scenarios**:

1. **Given** the discovery is enabled and a repository of the configured owner has the topic and a valid `deploy/` directory, **When** a few minutes pass, **Then** a dedicated namespace and a dedicated Application exist, the Application is Synced and Healthy, and the `XStaticSite` is Ready.
2. **Given** two tagged repositories with different names, **When** both are discovered, **Then** each has its own namespace and Application and neither affects the other.
3. **Given** a repository of a different owner carries the same topic, **When** discovery runs, **Then** it is never discovered.

---

### User Story 2 - Converge on manifest changes (Priority: P1)

As a platform engineer, I change the manifest in a site repository and the cluster converges to it automatically, including reverting manual drift.

**Why this priority**: Without continuous convergence, discovery would only be a one-time bootstrap.

**Independent Test**: Commit a change to the site's `XStaticSite` (for example the error document) and verify the live resource matches within a few minutes; then edit the live resource manually and verify it is reverted.

**Acceptance Scenarios**:

1. **Given** a discovered, Healthy site, **When** its manifest is changed in the repository, **Then** the live `XStaticSite` matches the new manifest without manual refresh or sync.
2. **Given** a discovered site, **When** the live `XStaticSite` is edited by hand, **Then** it is reverted to the repository state.
3. **Given** a resource is removed from the repository's `deploy/` directory, **When** the Application syncs, **Then** the live resource is deleted (pruned), and the documentation warns that removing an `XStaticSite` from `deploy/` also deletes its bucket contents.

---

### User Story 3 - Guardrails limit what a repository can deploy (Priority: P1)

As a platform engineer, I need a discovered repository to be unable to deploy anything beyond a static site, so that tagging a repository cannot compromise the cluster.

**Why this priority**: Repositories are outside this repository's review process; the guardrails are what make automatic deployment safe.

**Independent Test**: Push a manifest to a discovered repository containing a disallowed kind (for example a Secret), a cluster-scoped resource, and an `XStaticSite` targeting another namespace; verify each is rejected and the rejection is visible on the Application.

**Acceptance Scenarios**:

1. **Given** a discovered repository, **When** its `deploy/` directory contains a kind other than `XStaticSite`, **Then** that resource is not applied and the Application shows a clear rejection.
2. **Given** a discovered repository, **When** a manifest targets a namespace other than the site's own, **Then** it is not applied and the rejection is visible.
3. **Given** a discovered repository, **When** a manifest declares a cluster-scoped resource, **Then** it is not applied and the rejection is visible.

---

### User Story 4 - See what is discovered and why not (Priority: P2)

As a platform engineer, I can list which repositories are discovered and determine why a given repository was not (wrong topic, invalid repository name, missing `deploy/` directory, invalid manifest, unreadable repository).

**Why this priority**: Without it, a missing site is a silent failure and troubleshooting depends on guesswork.

**Independent Test**: Prepare one repository for each non-discovery reason and verify the documented procedure identifies the reason for each one.

**Acceptance Scenarios**:

1. **Given** several discovered sites, **When** the operator follows the documented listing procedure, **Then** each discovered repository is shown with its Application status.
2. **Given** a tagged repository without `deploy/`, **When** the operator investigates, **Then** its Application is visibly failed with an error that the `deploy/` path does not exist.
3. **Given** a tagged repository with an invalid manifest, **When** the operator investigates, **Then** its Application is visibly failed with the validation error.
4. **Given** a repository that cannot be read, **When** the operator investigates, **Then** the documented procedure shows the access failure rather than a silent absence.

---

### User Story 5 - Decommission a site deliberately (Priority: P2)

As a platform engineer, I decommission a site on purpose: archiving, untagging or deleting the repository never deletes the namespace, the `XStaticSite` or the bucket and its contents; a documented explicit procedure performs the removal.

**Why this priority**: Site content lives in ephemeral lab storage; accidental deletion by an upstream metadata change is unacceptable.

**Independent Test**: Untag a discovered repository and verify namespace, Application, `XStaticSite` and bucket remain; then follow the documented procedure and verify everything is removed.

**Acceptance Scenarios**:

1. **Given** a discovered site, **When** its repository is untagged, archived or deleted, **Then** the namespace, `XStaticSite` and bucket contents remain intact.
2. **Given** the operator follows the documented removal procedure, **When** it completes, **Then** the Application, namespace, `XStaticSite` and bucket are removed, and the procedure states what is irrecoverable.

### Edge Cases

- Repository tagged but with no `deploy/` directory: its Application exists and shows the missing-path failure (US4); no resources are created.
- Repository tagged with an invalid manifest: the Application exists and shows the failure; no other site is affected.
- Two repositories whose names would map to the same namespace: impossible by construction, because the namespace is the fixed prefix plus the unmodified repository name and repository names are unique per owner.
- Repository renamed: the new name is treated as a new site; the previous namespace and resources are kept until explicit removal, and the documentation warns that the old and new names then run as two independent sites with separate buckets.
- Repository archived, deleted or untagged: its Application, namespace and resources are retained (US5); the Application keeps auto-syncing from the repository, so a deleted or unreadable repository shows as a visible sync error until explicit removal.
- GitHub API unavailable or rate-limited during discovery: existing Applications and sites are unaffected, no site is removed as a result of an incomplete listing, and the failure is visible to the operator.
- Manifest with a disallowed kind or a different target namespace: rejected and visible (US3).
- Argo CD restarted or the discovery definition deleted: existing Applications, namespaces and sites survive.
- Repository whose name is not already valid as part of a namespace name (uppercase letters, underscores, dots, or too long once prefixed): it is not deployed, no normalization is attempted, and the operator can tell why.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The platform MUST discover repositories solely through the GitHub topic `backstage-static-site`, restricted to one configurable GitHub owner, which MUST be a GitHub organization (the built-in discovery cannot list a personal account's repositories); repositories of other owners MUST never be discovered.
- **FR-002**: Each discovered repository MUST get its own namespace, named as a fixed prefix plus the unmodified repository name, and its own Application, so sites are isolated from each other. A repository whose name does not yield a valid namespace name MUST NOT be deployed, and the reason MUST be discoverable.
- **FR-003**: Each Application MUST sync only the repository's `deploy/` directory, taken from the repository's default branch, into that repository's own namespace.
- **FR-004**: All discovered Applications MUST run in a dedicated Argo CD project that permits only: the discovered repositories' source, the sites' destination namespaces, and the `XStaticSite` kind. Cluster-scoped resources, other than the site's own namespace that the platform creates for it, and any other kind (including Secrets, RBAC and Crossplane Providers) MUST be denied.
- **FR-005**: A manifest that violates the limits of FR-004, or that targets any namespace other than its own site's (including another site's namespace), MUST be rejected, and the rejection MUST be visible on the Application to the operator.
- **FR-006**: Applications MUST use automated sync with self-heal. Pruning of resources removed from the repository's `deploy/` directory MUST be enabled; cascade deletion of an Application's resources when its repository leaves the discovered set MUST remain disabled (FR-007).
- **FR-007**: Removal of a repository from the discovered set (untag, archive, delete) MUST NOT delete the Application's resources, its namespace or the bucket. The Application MUST never be removed automatically: it stays in place and keeps auto-syncing until the documented removal procedure is run, and no automatic cascade deletion MAY occur.
- **FR-008**: Resource tracking MUST remain annotation-based, as already configured, so Argo CD coexists with Crossplane-managed resources.
- **FR-009**: No credential or token MUST be stored in this repository, manifests, logs or shell history. Public repositories MUST work without any credential. If private repositories are ever needed, credentials MUST be created outside Git.
- **FR-010**: The documentation MUST state the GitHub API rate-limit implication of discovery (unauthenticated access is limited to 60 requests per hour per source address; authenticated access to a much higher limit) and the mitigations: an optional out-of-Git credential, and a polling interval chosen so that expected requests stay within the available limit.
- **FR-011**: A newly tagged repository MUST be reflected in Argo CD within a few minutes with no manual refresh (SC-001).
- **FR-012**: If the GitHub listing is unavailable, rate-limited or incomplete, the platform MUST NOT remove or alter existing Applications or sites, and the failure MUST be visible to the operator.
- **FR-013**: Two sites MUST NEVER share a namespace; the namespace name MUST NOT be derived by lossy normalization of the repository name.
- **FR-014**: Existing Applications, namespaces and sites MUST survive an Argo CD restart and deletion of the discovery definition.
- **FR-015**: The operator MUST be able to list discovered repositories and determine the reason for each non-discovery case: wrong topic, invalid repository name, missing `deploy/`, invalid manifest, unreadable repository.
- **FR-016**: The feature MUST be delivered as a manifest under `applications/` managed by the existing root Argo CD application. Any new upstream dependency MUST be version-pinned, and the lab-only conventions of specs 001–003 MUST be followed.
- **FR-017**: The repository MUST contain a quickstart covering staged rollout, verification, troubleshooting and deliberate cleanup, like the previous specs.
- **FR-018**: The feature MUST NOT create or require any real AWS resource or credential, and MUST NOT expose LocalStack or the sites publicly.

### Key Entities *(include if feature involves data)*

- **Site repository**: A GitHub repository of the configured owner carrying the topic and holding its `XStaticSite` manifest under `deploy/`.
- **Site Application**: The per-repository Argo CD Application that syncs `deploy/` into the site namespace.
- **Site namespace**: The namespace derived deterministically from the repository name, containing the site's resources.
- **Site project**: The Argo CD project that bounds source, destinations and allowed kinds for all site Applications.
- **Discovery definition**: The single declaration that finds tagged repositories and generates one Application per repository.

### Out of Scope

- Creating repositories, templates or Backstage configuration.
- The GitHub Actions self-hosted runner and uploading site content.
- Any real AWS resources.
- Discovery of repositories of other owners or by criteria other than the topic.
- Automatic decommissioning of sites.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A newly tagged repository with a valid manifest has a Healthy Application and a Ready `XStaticSite` in its own namespace within 5 minutes, with zero commits to this repository and no manual refresh.
- **SC-002**: A manifest change in a site repository is reflected in the live site within 5 minutes without manual action.
- **SC-003**: In review and testing, 100% of attempts to deploy a disallowed kind, a cluster-scoped resource or a foreign namespace are rejected, and the rejection is visible to the operator in under 2 minutes.
- **SC-004**: After untagging, archiving or deleting a repository, 100% of its namespace, `XStaticSite` and bucket contents remain until the documented removal procedure is run.
- **SC-005**: For each of the five non-discovery reasons, an operator identifies the cause in under 5 minutes using the documentation.
- **SC-006**: After an Argo CD restart or deletion of the discovery definition, 100% of existing sites remain present and Ready.
- **SC-007**: A search of this repository, the cluster manifests and the documented commands finds no stored token or credential.

## Assumptions

- Spec 003 is in place: Argo CD, Crossplane and LocalStack run on Minikube and the namespaced `XStaticSite` API is available.
- Resource tracking is already annotation-based in the lab's Argo CD; planning verifies it rather than treating it as proof.
- The default discovery runs over public repositories without a credential at a 3-minute polling interval. Each cycle costs 1 + N GitHub requests (N = tagged repositories), so the unauthenticated limit of 60 per hour supports at most one tagged repository at that interval with headroom; two or more sites require the out-of-Git credential or a longer interval (see research.md).
- The namespace name is a fixed prefix plus the unmodified repository name, so it is unique per repository; repositories with names that are invalid for this are not deployed (FR-002), and the site template (separate spec) is responsible for generating valid names.
- Archived repositories are treated like untagged ones: no new Application is created, and the existing Application and resources are kept.
- Each `XStaticSite` gets its own bucket named from the resource's unique identity (spec 003), so same-named manifests in different site repositories never collide.
- The lab's GitHub owner `guigsgbm` is a personal account, so site repositories must be created in a GitHub organization (name supplied when the feature is rolled out).
- Pruning follows the constitution's default (enabled), including on the root and Crossplane applications. Only the repository-level lifecycle (untag, archive, delete, rename) is decoupled from deletion, and the quickstart documents that deleting a manifest or removing an `XStaticSite` from `deploy/` deletes the bucket and its contents.
