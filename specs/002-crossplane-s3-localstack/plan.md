# Implementation Plan: Manage LocalStack S3 Buckets with Crossplane

**Branch**: `main` (specification directory is pinned by `.specify/feature.json`) | **Date**: 2026-10-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/002-crossplane-s3-localstack/spec.md`

## Summary

Install the Upbound AWS provider family and S3 provider as Crossplane packages pinned to `v2.4.0`, configure their S3-compatible endpoint to the existing LocalStack service, and manage a sample bucket through a Crossplane resource committed to Git. The live cluster was checked read-only: the Crossplane deployment runs `v2.4.2`, and no providers are installed. Both provider package pages declare compatibility with Crossplane `2.0+`.

The selected provider release is an explicit trade-off. An open upstream report documents S3 bucket observe failures against custom endpoints starting in provider `v2.5.0`; its author reports `v2.4.0` worked. Upbound's Marketplace currently reports two High vulnerabilities for `v2.4.0`. The user selected this version with those findings disclosed. Keep the provider limited to this study lab, use dummy credentials only, and re-check the package's security status before merge.

Roll out in two GitOps stages: first reconcile the provider packages and wait for them to become healthy and install their APIs; then create the dummy-credential Secret outside Git and commit the `ClusterProviderConfig` and bucket resource. This avoids applying custom resources before their CRDs are served.

## Technical Context

**Language/Version**: Kubernetes YAML and Markdown; Crossplane `v2.4.2` (verified from the live deployment image)

**Primary Dependencies**: Argo CD (existing root application), Crossplane `v2.4.2`, Upbound `provider-family-aws:v2.4.0`, Upbound `provider-aws-s3:v2.4.0`, LocalStack (existing)

**Storage**: S3 bucket state remains in LocalStack's existing ephemeral storage; no durable persistence is added.

**Testing**: Server-side dry-run after provider CRDs are installed; Argo CD sync/health; Crossplane provider and bucket readiness; AWS CLI `s3 ls` against the internal LocalStack endpoint with dummy credentials.

**Target Platform**: Existing Minikube cluster (Kubernetes API reports `v1.37.0`), Argo CD in `argocd`, Crossplane in `crossplane-system`, and LocalStack in `localstack`

**Project Type**: Kubernetes platform GitOps repository

**Performance Goals**: Provider installation and the first declared bucket reconcile within 10 minutes each (per SC-002 and SC-003).

**Constraints**: Pin both provider packages; point S3, S3 Control, and any required STS operations at LocalStack; use no real AWS credentials; do not expose LocalStack; keep dummy credentials out of Git; account for the root application's `prune: false`; explicitly document the two High Marketplace vulnerability findings on the selected provider release.

**Scale/Scope**: One lab owner, one LocalStack endpoint, one example S3 bucket; support additional buckets by adding declarations, not other AWS services.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. Provider packages, provider configuration, and bucket desired state are committed and reconciled by the existing root application. The local-only test credential Secret remains out of Git. |
| II. Focus on platform infrastructure | Pass. The provider extends the existing Crossplane platform capability and the bucket is a declarative infrastructure resource. |
| III. Reconcile automatically, prune deliberately | Pass with an operational note. The root application self-heals but has `prune: false`; the quickstart documents deliberate cleanup rather than promising automatic deletion. |
| IV. Keep secrets out of Git | Pass. The AWS credential Secret contains only dummy LocalStack credentials and is created outside Git. No real AWS credential or LocalStack auth token is added. |
| V. Make deployments reproducible | Pass with an explicit security exception. Both AWS provider packages are pinned to `v2.4.0`; see Complexity Tracking for the Marketplace's two High findings and the user's acceptance of this study-lab trade-off. |
| VI. Review cluster impact | Pass. Validation covers provider compatibility, CRD availability, exact API targets, sync behavior, bucket lifecycle, and security findings before merge. |

**Post-design re-check**: Pass with the same documented, user-accepted provider vulnerability exception. No Crossplane or Argo CD installation changes are required.

## Project Structure

### Documentation (this feature)

```text
specs/002-crossplane-s3-localstack/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── aws-s3-bucket.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Created by /speckit-tasks, not by this plan
```

### Source Code (repository root)

```text
applications/
├── crossplane.yaml                 # Existing; unchanged
├── localstack.yaml                 # Existing; unchanged
├── crossplane-aws-s3.yaml          # New: pinned AWS family and S3 Provider packages
└── localstack-s3-bucket.yaml       # New: ClusterProviderConfig and namespaced Bucket
README.md                           # Updated: link and operational summary
```

**Structure Decision**: Keep the desired state in the existing `applications/` directory watched by the root Argo CD application. Use two commits for initial rollout: provider packages first; after they report healthy and their APIs are served, add the provider configuration and bucket declaration. No changes to the root application, Crossplane Helm release, or LocalStack deployment are planned.

## Complexity Tracking

| Violation / exception | Why needed | Simpler alternative rejected because |
|-----------------------|------------|-------------------------------------|
| Use AWS provider packages `v2.4.0`, which Upbound Marketplace reports with 2 High and 2 Low vulnerabilities and 0 Critical vulnerabilities (as checked 2026-10-30). | The user selected the release after review. It is declared compatible with Crossplane 2.0+, and an open upstream report records successful LocalStack reconciliation on `v2.4.0` while reporting a custom-endpoint S3 observe regression from `v2.5.0`. | Newer `v2.9.5` is declared compatible and has no known vulnerabilities, but the upstream custom-endpoint issue remains open and the reported regression begins at `v2.5.0`; `v2.9.5` has not been verified against this LocalStack setup. Revisit the pin as soon as a secure, tested release is available. |
| Create the AWS credential Secret outside Git. | Crossplane's AWS provider requires credentials; LocalStack accepts non-production dummy credentials for this lab. | Committing even dummy credential material is unnecessary and conflicts with the repository's secret-handling principle; no external secrets controller is installed. |
