# Research: LocalStack for the Kubernetes Lab

Researched 2026-10-19 against official LocalStack documentation, the official Helm repository index, and Docker Hub tags. No `NEEDS CLARIFICATION` items remain.

## R1. Image distribution and authentication

- **Decision**: Use `localstack/localstack` pinned to `2026.10.0` with a LocalStack Free (hobby) plan account and auth token, supplied through the `LOCALSTACK_AUTH_TOKEN` environment variable from a Kubernetes Secret.
- **Rationale**: Since 2026-03-23 LocalStack ships a single unified image, and the latest releases require an account and token, including for hobbyist use. The Free plan is intended for non-commercial individual use, which matches this lab. `2026.10.0` exists on Docker Hub for amd64 and arm64.
- **Alternatives considered**:
  - *Pin a pre-March-2026 `4.x` tag with no token*: avoids the account but receives no fixes, drifts from current AWS behavior, and accumulates CVEs. Kept as a documented fallback only. If chosen, the Secret and token wiring are simply removed.
  - *`latest` tag*: rejected by constitution Principle V.
  - *Paid plans*: not needed for study.
- **Caveat**: Which services and features the Free plan includes can change. Check the plan's entitlements when selecting study exercises.

## R2. Deployment mechanism

- **Decision**: Argo CD `Application` sourcing the official Helm chart `localstack` version `0.7.1` from `https://localstack.github.io/helm-charts`, with inline `valuesObject`.
- **Rationale**: Mirrors the existing `crossplane` application, keeps everything in one reviewable manifest, and uses the vendor-maintained chart. Chart `0.7.1` (published 2026-09-18) is the latest in the repository index.
- **Alternatives considered**: Raw Deployment/Service manifests (more to maintain, loses chart defaults for probes, RBAC and ports); a separate values file in Git (needs Argo CD multi-source, unnecessary complexity).
- **Chart defaults that must be overridden**: `image.repository` defaults to `localstack/localstack-pro` and `image.tag` to `latest`; `service.type` defaults to `NodePort`.

## R3. Network exposure and access

- **Decision**: `service.type: ClusterIP`, no ingress. In-cluster endpoint is `http://localstack.localstack.svc.cluster.local:4566`. Workstation access uses `kubectl port-forward svc/localstack 4566:4566 -n localstack`.
- **Rationale**: Satisfies FR-003, FR-004 and FR-010 with no public route and no node port. The chart's `NodePort` default would expose the endpoint on the Minikube node address, which is wider than needed.
- **Alternatives considered**: `NodePort`/`minikube service` (broader exposure); ingress (public-route risk and extra components); LocalStack DNS service (not needed for study).
- **Note**: The chart also exposes ports 4510-4559 for external service ports. They stay inside the cluster as `ClusterIP`.

## R4. Secret handling

- **Decision**: Create the Secret `localstack-auth-token` (key `auth-token`) in namespace `localstack` with `kubectl create secret generic`, outside Git. The manifest references it with `valueFrom.secretKeyRef`.
- **Rationale**: Meets FR-008 and Principle IV with no extra tooling. The token is user-specific and must not be committed or logged.
- **Alternatives considered**: External Secrets Operator, Sealed Secrets, SOPS (reasonable if the lab later adopts one; excessive for a single token now); `envFrom` a Secret (works equally but a named key is more explicit).
- **Ordering**: The Secret needs the namespace. Create the namespace first (Argo CD's `CreateNamespace=true` tolerates it already existing), or create the Secret after the first sync; the pod then recovers after a restart.

## R5. State and persistence

- **Decision**: Ephemeral by default (`persistence.enabled: false`). Resources are lost when the pod restarts.
- **Rationale**: Matches the spec assumption and makes resetting the lab as simple as restarting the pod. Avoids a PVC whose pruning could surprise the owner.
- **Alternatives considered**: Enable chart persistence (PVC on Minikube's default storage class). Documented as an opt-in change, not part of this feature.

## R6. Service selection

- **Decision**: No service allow-list by default. Optionally restrict services through a `SERVICES` entry in `extraEnvVars`.
- **Rationale**: FR-005 requires that services be configurable without changing the study client; an env var in the manifest does that. Whether `SERVICES` is still honored and which services the Free plan covers must be confirmed during implementation against the pinned release.
- **Alternatives considered**: Startup scripts (`startupScriptContent`) for seeding resources. Out of scope for the first iteration.

## R7. Compute resources and Lambda

- **Decision**: Set modest requests (`250m` CPU, `512Mi` memory) and a `2Gi` memory limit, no CPU limit. Leave Docker-in-Docker (`mountDind`) disabled.
- **Rationale**: Prevents a runaway emulator from starving Minikube while keeping the chart's guidance about explicit values on constrained clusters. Lambda container execution would need Docker access and is out of scope for the initial feature.
- **Alternatives considered**: No resources (risk of unbounded use); mounting the Docker socket (a significant privilege increase).

## R8. Sync and pruning policy

- **Decision**: `automated` with `selfHeal: true` and `prune: true`, plus `CreateNamespace=true`.
- **Rationale**: Principle III makes pruning the default and allows disabling it only when deletion could lose data. LocalStack state is ephemeral by design, so pruning is safe. This differs from the existing applications, which set `prune: false`.
- **Cleanup consequence**: The root application has `prune: false`, so deleting `applications/localstack.yaml` from Git does not remove the `localstack` `Application`. Removal requires an explicit cascading delete of that Application (see the quickstart), then deleting the Secret and namespace.

## R9. Validation approach

- **Decision**: Pre-merge `helm template` render with the planned values; post-merge Argo CD status, an in-cluster AWS CLI smoke test with dummy AWS credentials, and a port-forward check.
- **Rationale**: Covers FR-001 to FR-004 and SC-001 to SC-003 with tools already used in the lab. The render check was run during planning with chart `0.7.1` and produced a `ClusterIP` Service on port 4566.
- **Limitation**: `kubectl` and `minikube` are not available in the environment where this plan was written, so live cluster behavior is unverified until implementation.
