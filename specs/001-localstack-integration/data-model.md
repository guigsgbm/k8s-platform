# Data Model: LocalStack for the Kubernetes Lab

This feature has no application database. The entities below describe the configuration and runtime objects involved, derived from the spec's key entities.

## LocalStack lab service

| Field | Value | Source |
|-------|-------|--------|
| Argo CD Application | `localstack` in namespace `argocd` | `applications/localstack.yaml` |
| Chart | `localstack` `0.7.1` from `https://localstack.github.io/helm-charts` | pinned (FR-006) |
| Image | `localstack/localstack:2026.10.0` | pinned (FR-006) |
| Destination | cluster `https://kubernetes.default.svc`, namespace `localstack` | |
| Service | `ClusterIP`, name `localstack`, edge port `4566` | FR-003, FR-010 |
| Enabled services | All entitled services by default; optional `SERVICES` env var | FR-005 |
| Readiness | Chart probe on `/_localstack/health` | edge cases in spec |

**Lifecycle states**: `Absent` → `Syncing` → `Starting` (pod running, not ready) → `Ready` → `Degraded` (probe failing, missing token, Git unreachable) → `Removed`.

**Validation rules**: image tag must not be `latest`; chart version must be pinned; no secret value may appear in the manifest.

## Auth token Secret (out of Git)

| Field | Value |
|-------|-------|
| Name / key | `localstack-auth-token` / `auth-token` |
| Namespace | `localstack` |
| Created by | Owner, with `kubectl create secret generic` |
| Consumed as | `LOCALSTACK_AUTH_TOKEN` via `secretKeyRef` |

**Rule**: never committed to Git, printed in logs, or shared; a missing Secret prevents the pod from becoming ready.

## Study client

| Attribute | In-cluster workload | Workstation tool |
|-----------|---------------------|------------------|
| Endpoint | `http://localstack.localstack.svc.cluster.local:4566` | `http://localhost:4566` via port-forward |
| Credentials | Dummy values (`test` / `test`), never real AWS keys | same |
| Region | `us-east-1` (any valid region) | same |

## Simulated resource state

- Held in the LocalStack pod's memory/ephemeral storage.
- **Transitions**: created by exercises → lost on pod restart, Argo CD recreate, or removal.
- Optional future state: PVC-backed persistence (not enabled by this feature).
