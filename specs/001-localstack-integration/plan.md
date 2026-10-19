# Implementation Plan: LocalStack for the Kubernetes Lab

**Branch**: `main` (no feature branch; the spec directory is pinned by `.specify/feature.json`) | **Date**: 2026-10-19 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-localstack-integration/spec.md`

## Summary

Add LocalStack to the Minikube lab as one more Argo CD-managed infrastructure application. A single new manifest, `applications/localstack.yaml`, installs the official LocalStack Helm chart (pinned) with a pinned LocalStack image into a dedicated `localstack` namespace. The service is `ClusterIP` only: in-cluster workloads use `http://localstack.localstack.svc.cluster.local:4566`, and the workstation uses `kubectl port-forward`. State is ephemeral by default.

Research found one fact that shapes the whole design: since 2026-03-23 every current LocalStack release requires an account and an auth token (a free plan exists for non-commercial hobby use). The token is created as a Kubernetes Secret outside Git and referenced by the chart; it is never committed. See [research.md](./research.md).

## Technical Context

**Language/Version**: N/A (declarative YAML and Markdown only)

**Primary Dependencies**: Argo CD (existing); LocalStack Helm chart `localstack` `0.7.1` from `https://localstack.github.io/helm-charts`; image `localstack/localstack:2026.10.0`

**Storage**: None by default (ephemeral). Optional PVC-backed persistence is documented but disabled.

**Testing**: Manual, documented validation in [quickstart.md](./quickstart.md): `helm template` render check before merge, Argo CD `Synced`/`Healthy`, an in-cluster AWS CLI smoke test, and a port-forward check. Chart rendering with the planned values was already verified (ClusterIP service on 4566, secret-backed token env var).

**Target Platform**: Minikube single-node cluster (kubectl context `minikube`), managed by Argo CD in namespace `argocd`

**Project Type**: GitOps infrastructure repository (no application source code)

**Performance Goals**: Ready within 10 minutes of publishing (SC-001); smoke test under 10 minutes (SC-002); workstation access under 5 minutes (SC-003)

**Constraints**: No plaintext secrets in Git; no public exposure; pinned versions; personal non-commercial use (Free plan terms); Minikube resource limits

**Scale/Scope**: One replica, one owner, one namespace, one Argo CD `Application`

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. Desired state lives in `applications/localstack.yaml`. The Secret is the only out-of-band object (see IV). |
| II. Focused on platform infrastructure | Pass. LocalStack is a platform tool with an explicit `Application` under `applications/`. |
| III. Reconcile automatically, prune deliberately | Pass. Automated sync, self-heal and prune enabled; pruning is safe because state is ephemeral. Note: the existing `crossplane` and root applications use `prune: false`, which differs from the constitution default. That is pre-existing and out of scope. Cleanup consequences are documented. |
| IV. Keep secrets out of Git | Pass with a lab-scoped exception. The repository contains only a `secretKeyRef`. The Secret is created manually with `kubectl` because no external secrets system exists in this lab. Recorded in Complexity Tracking. |
| V. Reproducible deployments | Pass. Chart `0.7.1` and image `2026.10.0` are pinned; the chart default `latest` tag is overridden. |
| VI. Review for cluster impact | Pass. Review checklist in the quickstart. Impact is limited to one new namespace; nothing existing is changed. |

**Post-design re-check**: Pass. No new violations were introduced by the Phase 1 design.

## Project Structure

### Documentation (this feature)

```text
specs/001-localstack-integration/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── localstack-endpoint.md   # Phase 1 output
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 output (/speckit-tasks - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
applications/
├── crossplane.yaml
└── localstack.yaml      # NEW: Argo CD Application with inline Helm values
README.md                # UPDATED: LocalStack section (prerequisites, access, cleanup, state)
```

**Structure Decision**: Follow the existing one-manifest-per-application convention. Helm values are inline in the `Application` (`spec.source.helm.valuesObject`), so no new directories, charts, or bootstrap changes are needed. The root application already watches `applications/`.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Auth token Secret created manually, outside Git (Principle IV: "external secrets system") | Current LocalStack releases cannot start without a token, and the lab has no external secrets system | Installing an external secrets system or sealed-secrets for one personal token is disproportionate; committing the token is forbidden. Manual Secret keeps Git clean and is documented as an operational assumption. |
