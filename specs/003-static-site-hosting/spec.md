# Feature Specification: Host Static Sites with Crossplane

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-10-30

**Status**: Draft

**Input**: User description: "Disponibilizar sites estáticos via Crossplane e LocalStack. Objetivo: Permitir que a equipe declare um site estático usando um recurso de alto nível do Crossplane. Uma Composition transforma essa declaração em recursos S3 configurados para hospedar o site no LocalStack."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Declare a static site (Priority: P1)

As a platform user, I want to declare a static site through a high-level Crossplane resource so that its S3 hosting infrastructure is provisioned in the lab's LocalStack without writing individual S3 resource declarations.

**Why this priority**: A reusable site declaration and its resulting hosting resources are the core capability requested.

**Independent Test**: Add one site declaration to the repository, allow the existing GitOps workflow to reconcile it, and verify that the site and its backing hosting resources report ready in LocalStack.

**Acceptance Scenarios**:

1. **Given** Crossplane and the LocalStack S3 provider are ready, **When** a user declares a site with a unique name and an index document, **Then** the platform creates the required S3 hosting resources in LocalStack and reports the site's readiness.
2. **Given** two sites have distinct declarations, **When** both are reconciled, **Then** each has isolated hosting resources and one site's configuration does not overwrite the other's.
3. **Given** LocalStack is unreachable or the declaration is invalid, **When** reconciliation is attempted, **Then** the site is not reported ready, the failure is inspectable, and no request is redirected to real AWS.

### User Story 2 - Verify and operate a hosted site (Priority: P2)

As a platform user, I want to find the local website address and verify its content so that I can use a provisioned site without exposing LocalStack to the internet.

**Why this priority**: A successfully provisioned bucket is useful only when users can identify and verify the site endpoint and understand its lifecycle.

**Independent Test**: After a site reports ready, upload a small HTML file using the documented LocalStack workflow, request the site's local website address, and verify the expected page content. Confirm that the address is reachable only through the lab's internal access path.

**Acceptance Scenarios**:

1. **Given** a site is ready and its content has been uploaded, **When** a user opens the reported local website address, **Then** the configured index document is served with the expected content.
2. **Given** a site has a custom error document and a requested object is absent, **When** a user requests the absent path, **Then** the configured error document is served according to the site's settings.
3. **Given** a user removes a site declaration from Git, **When** they follow the documented cleanup procedure, **Then** they can distinguish removing the declaration from deliberately deleting the Crossplane and LocalStack resources.

### Edge Cases

- The site declaration omits or specifies an invalid index document; the request is rejected or reports a clear reconciliation failure rather than producing a ready site.
- Two declarations request the same site identifier or backing bucket name; the conflict is visible and does not silently replace another site's resources.
- LocalStack becomes unavailable during provisioning or later reconciliation; the failure is visible and no resources are created in a real AWS account.
- The hosting resources are ready but no site files have been uploaded; the platform distinguishes infrastructure readiness from content being available to serve.
- The website endpoint is requested from outside the lab's internal access path; no public route to LocalStack or the hosted site is created.
- A site declaration is removed from Git; because the root Argo CD application has pruning disabled, removal of the file alone is not represented as guaranteed cleanup.
- LocalStack restarts and loses its ephemeral state; the site and content recovery implications are documented.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A platform user MUST be able to declare a static site using a high-level Crossplane resource managed through the existing GitOps repository.
- **FR-002**: The platform MUST translate each site declaration into the S3 hosting resources and settings required to serve a static website in LocalStack, without requiring the user to declare those lower-level resources individually.
- **FR-003**: A site declaration MUST identify the site uniquely and MUST support an index document, defaulting to `index.html`; it MAY specify an error document.
- **FR-004**: The platform MUST support multiple sites at once, with separate backing storage and settings for each declaration.
- **FR-005**: The site status MUST make provisioning readiness, failures, and the local website address discoverable.
- **FR-006**: Site content MUST be retrievable through the local website address after it is uploaded to the site's backing storage; the repository MUST document the content-upload and smoke-test workflow.
- **FR-007**: All S3 operations for this feature MUST target the existing in-cluster LocalStack endpoint, `http://localstack.localstack.svc.cluster.local:4566`; an unavailable or invalid LocalStack endpoint MUST NOT cause fallback to real AWS.
- **FR-008**: The feature MUST require no real AWS account or credentials, and MUST NOT commit plaintext secret values.
- **FR-009**: The LocalStack service and website access MUST remain internal to the lab; this feature MUST NOT create a public ingress or route to LocalStack.
- **FR-010**: The repository MUST document how site declarations and their resources are intentionally removed, accounting for the root Argo CD application's `prune: false` behavior and LocalStack's ephemeral state.
- **FR-011**: The planning phase MUST verify that the existing Crossplane and AWS S3 provider support the required website-hosting resources and work with the current LocalStack setup before implementation begins.

### Key Entities *(include if feature involves data)*

- **Static site declaration**: A user's high-level desired state for one site, including its unique identifier and document settings.
- **Hosted site**: The reconciled site and its local website address, readiness, and any provisioning failure.
- **Site content**: HTML and related static assets uploaded separately to the site's backing storage; content deployment is not managed by the site declaration in this feature.
- **Local website address**: The lab-internal address through which the configured site documents are served by LocalStack.

### Out of Scope

- Building site source code or automatically uploading and versioning site content.
- Public DNS, TLS certificates, CDN services, or internet-facing ingress.
- Provisioning or managing any resources in a real AWS account.
- Installing or upgrading Crossplane, LocalStack, Argo CD, or the existing AWS S3 provider unless planning establishes that a compatible update is required.
- Durable storage for LocalStack data.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A valid site declaration results in all required hosting resources reporting ready in LocalStack within 10 minutes of reconciliation.
- **SC-002**: After a test page is uploaded, a user can retrieve the configured index document and verify its expected response content through the local website address within 5 minutes.
- **SC-003**: Two distinct site declarations can be reconciled simultaneously without naming collisions or cross-site configuration changes.
- **SC-004**: A user can identify a site's readiness, failure reason, and local website address from its reported status in under 5 minutes.
- **SC-005**: Review and smoke testing confirm that all site operations use LocalStack, no real AWS resources or credentials are involved, and no public route to LocalStack is created.
- **SC-006**: The documented cleanup steps accurately describe the effects of removing a declaration and the loss of ephemeral LocalStack state.

## Assumptions

- Crossplane, LocalStack, and the Upbound AWS S3 provider are already installed in the lab. Existing repository configuration declares Crossplane `2.4.2`, pins the AWS provider packages to `v2.4.0`, and configures the LocalStack endpoint; planning will verify the live environment and compatibility rather than treating those declarations as proof.
- LocalStack is available only through the existing cluster-internal `ClusterIP` service. Local workstation access, when needed, uses the lab's existing local-only access procedure.
- The initial site experience is infrastructure provisioning and hosting configuration. Users upload site files separately; a content build or deployment pipeline is not part of this feature.
- A site is intended for unauthenticated reading through the local emulator's website endpoint, not for internet exposure or production use.
- LocalStack data is ephemeral under the existing configuration, and the root Argo CD application does not automatically prune resources.
