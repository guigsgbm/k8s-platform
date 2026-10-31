# Feature Specification: Manage LocalStack S3 Buckets with Crossplane

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-10-30

**Status**: Implemented

**Input**: User description: "O laboratório já tem Crossplane instalado e o LocalStack disponível no cluster Kubernetes. A aplicação LocalStack usa o serviço `ClusterIP` no endereço interno `http://localstack.localstack.svc.cluster.local:4566`, conforme `applications/localstack.yaml`. Esta feature adiciona o provider AWS para S3 ao Crossplane e permite criar buckets no LocalStack por meio de recursos declarativos versionados no Git e reconciliados pelo Argo CD. A instalação do Crossplane, por si só, ainda não fornece o provider AWS para criar buckets. A versão do provider e sua compatibilidade com a versão atual do Crossplane deverão ser verificadas e fixadas durante o planejamento."

## User Scenarios & Testing

### User Story 1 - Declare an S3 bucket for LocalStack (Priority: P1)

As the lab owner, I want Crossplane to manage S3 buckets through the existing GitOps workflow so that I can create and review local AWS-compatible resources without using a real AWS account.

**Why this priority**: Installing and configuring the AWS provider and declaring a bucket together deliver the requested end-to-end capability.

**Independent Test**: Add a bucket declaration to the repository, let Argo CD reconcile it, and verify that Crossplane reports it ready and that the bucket is visible through the LocalStack S3 endpoint. The test uses no real AWS credentials and creates no real AWS resources.

**Acceptance Scenarios**:

1. **Given** Crossplane is installed and LocalStack is ready at the lab's internal endpoint, **When** the feature is planned, **Then** the exact installed Crossplane version is checked and an AWS provider release compatible with it is selected using authoritative compatibility or release documentation.
2. **Given** a compatible AWS provider release has been selected, **When** its desired state is committed, **Then** its exact version is pinned in Git and Argo CD reconciles the provider into Crossplane.
3. **Given** the provider is installed and configured for LocalStack, **When** the lab owner commits a declarative S3 bucket resource, **Then** Argo CD and Crossplane reconcile it and the bucket becomes available in LocalStack.
4. **Given** a bucket resource is being reconciled, **When** the lab owner checks its status and LocalStack, **Then** they can determine whether it is ready and verify its existence without contacting real AWS.
5. **Given** LocalStack is unreachable or its provider configuration is invalid, **When** Crossplane attempts reconciliation, **Then** the failure is visible and requests are not silently redirected to real AWS.
6. **Given** the lab owner reviews the Git changes, **When** they inspect provider and bucket configuration, **Then** no real AWS credentials or plaintext secret values are present.

### Edge Cases

- No AWS provider release is compatible with the Crossplane version installed in the lab; planning must resolve the compatibility before implementation proceeds rather than selecting an unverified or floating version.
- The provider package is installed but not healthy; bucket resources must not be presented as successfully provisioned, and the failure must be inspectable.
- LocalStack is unavailable or the configured endpoint cannot be reached; reconciliation reports the problem and does not fall back to AWS.
- A bucket name is invalid or already in use in LocalStack; the resource reports a reconciliation failure that the lab owner can inspect.
- The bucket declaration is removed from Git; because the existing root Argo CD application has pruning disabled, documentation must explain that removing the file alone does not necessarily remove the managed resource or the bucket.
- LocalStack loses its ephemeral state after a restart; the behavior of Crossplane's declared bucket resource is made clear to the lab owner.

## Requirements

### Functional Requirements

- **FR-001**: The AWS provider required to manage S3 buckets MUST be installed and configured through version-controlled desired state reconciled by the lab's existing Argo CD and Crossplane setup.
- **FR-002**: Before implementation, planning MUST verify the exact Crossplane version installed in the lab and use authoritative upstream documentation or release information to establish compatibility with the selected AWS provider release.
- **FR-003**: The AWS provider package version MUST be pinned to an exact release in Git; floating versions MUST NOT be used.
- **FR-004**: The provider configuration MUST direct S3 operations to `http://localstack.localstack.svc.cluster.local:4566` and MUST NOT silently send operations to real AWS if LocalStack is unavailable or the local endpoint is misconfigured.
- **FR-005**: The setup and verification MUST work without real AWS credentials or a real AWS account. Any authentication configuration MUST be limited to non-production LocalStack use, and plaintext secret values MUST NOT be committed to Git.
- **FR-006**: The lab owner MUST be able to declare one or more S3 buckets as version-controlled resources and have Argo CD reconcile those declarations for Crossplane to manage.
- **FR-007**: The lab owner MUST be able to inspect the provider and bucket reconciliation outcomes, including failures that prevent a bucket from becoming ready.
- **FR-008**: The repository MUST document a smoke test that verifies a declared bucket through LocalStack and clearly distinguishes the local emulator from real AWS.
- **FR-009**: The repository MUST document the effects of removing a bucket declaration, taking the existing root application's disabled pruning into account, and MUST NOT imply that deleting the Git file alone guarantees resource deletion.
- **FR-010**: This feature MUST remain scoped to the existing lab and MUST NOT expose LocalStack publicly or manage S3 resources in a real AWS account.

### Key Entities

- **AWS provider**: The Crossplane capability, at a pinned compatible release, that reconciles AWS-compatible S3 resources.
- **Provider configuration**: The connection and authentication settings that direct provider operations to the local emulator.
- **S3 bucket resource**: A declarative, Git-versioned desired bucket state reconciled by Crossplane and represented in LocalStack.

### Out of Scope

- Installing or upgrading Crossplane, LocalStack, Minikube, or Argo CD.
- Provisioning resources in a real AWS account or configuring a production AWS provider.
- Managing AWS resource types other than S3 buckets.
- Publicly exposing LocalStack or making its ephemeral state durable.

## Success Criteria

### Measurable Outcomes

- **SC-001**: The planning artifacts identify the installed Crossplane version, an exact compatible AWS provider release, and authoritative evidence for their compatibility before implementation begins.
- **SC-002**: After desired state is published, Argo CD reports the provider as reconciled and Crossplane reports it healthy within 10 minutes.
- **SC-003**: A bucket declared in Git is reported ready by Crossplane and can be listed through the LocalStack endpoint within 10 minutes of reconciliation, without creating a real AWS resource.
- **SC-004**: A lab owner can verify provider and bucket status and complete the documented LocalStack smoke test in under 10 minutes without real AWS credentials.
- **SC-005**: Review of the repository confirms the provider release is exact-pinned, operations target only the specified LocalStack endpoint, and no real credentials or plaintext secrets are committed.
- **SC-006**: The bucket removal guidance accurately reflects the root Argo CD application's `prune: false` behavior.

## Assumptions

- Crossplane and LocalStack are already installed and operating in the lab; this feature does not install either service.
- The Crossplane chart is currently pinned to `2.4.2` in `applications/crossplane.yaml`; planning will verify the actual installed version rather than assume the chart declaration alone proves it.
- LocalStack is available at `http://localstack.localstack.svc.cluster.local:4566` through its in-cluster `ClusterIP` service.
- The existing root Argo CD application watches `applications/`, synchronizes automatically, and has pruning disabled.
- This is a single-owner study lab. LocalStack is an emulator, not a security boundary or a production AWS service.
- LocalStack's state is ephemeral under its current configuration; durable bucket data is not part of this feature.
