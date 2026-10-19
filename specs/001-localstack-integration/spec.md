# Feature Specification: LocalStack for the Kubernetes Lab

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-10-19

**Status**: Draft

**Input**: User description: "I need to fit LocalStack into my current setup. This is a personal local lab for study. I currently have Minikube running Kubernetes and a GitOps repository for Kubernetes infrastructure applications connected to Argo CD. I want to use LocalStack for study."

## User Scenarios & Testing

### User Story 1 - Run LocalStack through the lab's GitOps workflow (Priority: P1)

As the lab owner, I want LocalStack to be part of my existing Kubernetes infrastructure setup so that I can use it without maintaining a separate deployment process.

**Why this priority**: The GitOps-managed service is the foundation for all subsequent study and testing scenarios.

**Independent Test**: Publish the LocalStack desired state to the infrastructure repository and confirm that the existing reconciliation workflow makes the service ready in Minikube without manually installing it into the cluster.

**Acceptance Scenarios**:

1. **Given** Minikube and Argo CD are already operating, **When** the LocalStack configuration is added to the infrastructure repository, **Then** Argo CD detects and reconciles it into the Minikube cluster.
2. **Given** the configuration is reconciled, **When** the lab owner checks the service status, **Then** they can determine whether LocalStack is ready or why it is not ready.
3. **Given** a LocalStack release has been selected, **When** the lab owner reviews the desired state, **Then** the exact release is visible and will not change without a deliberate repository change.
4. **Given** the LocalStack configuration is reviewed, **When** the lab owner checks the repository contents, **Then** no plaintext real credentials or access tokens are present.

### User Story 2 - Use the local AWS-compatible endpoint for study (Priority: P1)

As the lab owner, I want to direct a study client to LocalStack so that I can try AWS-compatible operations without using real AWS resources.

**Why this priority**: Using the emulator for hands-on exercises is the primary value of adding LocalStack to the lab.

**Independent Test**: Follow the documented smoke test from a workload in Minikube and verify a successful operation against LocalStack, with no request sent to a real AWS account.

**Acceptance Scenarios**:

1. **Given** LocalStack is ready, **When** a workload in Minikube is configured with the lab endpoint, **Then** it can reach LocalStack and complete a documented smoke test.
2. **Given** LocalStack is ready, **When** the lab owner follows the documented local access procedure from their workstation, **Then** they can reach the same endpoint without exposing it publicly.
3. **Given** an exercise uses AWS-compatible credentials, **When** it is configured for LocalStack, **Then** it does not require real AWS credentials or create real AWS resources.
4. **Given** the lab owner selects supported services for an exercise, **When** the LocalStack setup is configured for that exercise, **Then** the selected services are available through the same study endpoint without changing the study client.

### User Story 3 - Understand and safely reset the lab (Priority: P2)

As the lab owner, I want the lifecycle and state-retention behavior to be clear so that I can reset or remove LocalStack without surprising effects on the cluster.

**Why this priority**: Clear lifecycle expectations prevent confusion when study data is lost and help avoid unintended GitOps cleanup.

**Independent Test**: Follow the documentation to identify what survives a restart and how to intentionally remove the service, then verify the resulting state in Minikube.

**Acceptance Scenarios**:

1. **Given** LocalStack is restarted, **When** the lab owner checks simulated resources, **Then** the documented default state-retention behavior matches the observed behavior.
2. **Given** the lab owner decides to remove LocalStack, **When** they follow the documented cleanup steps, **Then** they can distinguish removing its desired configuration from deliberately deleting its cluster resources.

### Edge Cases

- Argo CD cannot read the Git repository or reconcile the LocalStack configuration; the lab owner can see a failure status and troubleshooting guidance rather than mistaking it for a ready service.
- The LocalStack process is running but not ready to accept requests; readiness is not reported until the endpoint can serve requests.
- The workstation's chosen local access port is already in use; the documented procedure explains how to choose another local port without changing cluster-wide exposure.
- LocalStack restarts or is removed while simulated AWS resources exist; the default ephemeral-state behavior and cleanup implications are documented.
- A workload is accidentally configured with real AWS credentials; the local exercise instructions make clear that LocalStack is not a production boundary and do not ask the user to commit real credentials.

## Requirements

### Functional Requirements

- **FR-001**: The lab owner MUST be able to add, review, and update LocalStack's desired state through the existing Kubernetes infrastructure GitOps repository.
- **FR-002**: The existing reconciliation workflow MUST deploy LocalStack to the Minikube lab without a routine manual installation step.
- **FR-003**: Study workloads running in Minikube MUST be able to reach a stable LocalStack endpoint.
- **FR-004**: The lab owner MUST have a documented way to reach the LocalStack endpoint from their workstation that does not make it publicly accessible.
- **FR-005**: The lab owner MUST be able to configure which supported AWS-compatible services are available for study without changing the study client.
- **FR-006**: The configuration MUST use a fixed LocalStack release so that an update to the lab repository does not silently change LocalStack behavior.
- **FR-007**: The setup and smoke-test instructions MUST work without real AWS credentials and MUST identify that test operations target the local emulator.
- **FR-008**: No plaintext real credentials, access tokens, or other secret values MUST be required in or committed to the GitOps repository.
- **FR-009**: The documentation MUST state whether simulated resources survive a restart and provide a safe, intentional removal procedure consistent with the repository's GitOps cleanup behavior.
- **FR-010**: The setup MUST remain scoped to the personal Minikube lab and MUST NOT expose the LocalStack endpoint to the public internet.

### Key Entities

- **LocalStack lab service**: The locally managed AWS-compatible emulator, its readiness, release, and enabled service capabilities.
- **Study client**: A workstation tool or in-cluster workload configured to send AWS-compatible requests to the local endpoint.
- **Simulated resource state**: Resources created by study exercises and the documented retention or reset behavior for that state.

## Success Criteria

### Measurable Outcomes

- **SC-001**: After the desired state is published to the GitOps repository, LocalStack becomes ready in Minikube within 10 minutes without a manual installation step.
- **SC-002**: A lab owner can complete the documented smoke test from an in-cluster study client and observe a successful response in under 10 minutes, without creating real AWS resources.
- **SC-003**: A lab owner can reach the endpoint from their workstation by following the documented local-only access procedure in under 5 minutes, with no public route to the endpoint.
- **SC-004**: All release, access, service-selection, state-retention, and cleanup expectations needed to repeat or reset the lab are documented in the repository.

## Assumptions

- Minikube and Argo CD are already installed and operating; this feature does not install or reconfigure either one.
- The infrastructure repository remains the source of truth for platform applications, and its existing Argo CD root application watches `applications/`.
- This is a single-owner study lab, not a production, shared, or internet-facing service.
- The initial setup provides a local endpoint for in-cluster workloads and a documented workstation access method; it does not add a public ingress.
- Simulated resources are ephemeral by default. Durable persistence is outside this initial feature unless the owner requests it later.
- No particular AWS service or study workload is prescribed. The lab owner chooses LocalStack-supported services as needed for individual exercises.
- The LocalStack release and distribution will be selected during planning, subject to current product access requirements and the repository's reproducibility and secret-handling principles.
