# Data Model: In-Cluster GitHub Actions Runners

Spec: [spec.md](./spec.md). There is no application data; this describes the objects, their identity and states. Names and values are in [contracts/runner-pool.md](./contracts/runner-pool.md).

## Entities

| Entity | Where it lives | Identity | Key attributes | Created by |
|--------|----------------|----------|----------------|------------|
| Lab organization | GitHub | Organization login (configured once, in `githubConfigUrl`) | One organization only | Operator (prerequisite) |
| GitHub token | GitHub | Token owner and expiry date | Fine-grained personal access token with resource owner `labs-guigsgbm`; organization permissions "Self-hosted runners" read/write and "Administration" read; expires, so it must be renewed | Operator, outside Git |
| Runner group | GitHub | Name `Default` | All repositories in the organization except public repositories | Organization setting, outside Git |
| `arc-github-token` Secret | `arc-runners` | Name | Key `github_token` | Operator, outside Git |
| `lab-aws-credentials` Secret | `arc-runners` | Name | Keys `access-key-id`, `secret-access-key` (dummy lab values) | Operator, outside Git |
| `arc-controller` Application | `argocd` | Name | Controller chart `0.15.1`, namespace `arc-systems`, wave 0, `prune: false` | This repository |
| `arc-runner-set` Application | `argocd` | Name | Pool chart `0.15.1`, namespace `arc-runners`, wave 1, `prune: false` | This repository |
| Runner controller | `arc-systems` | Deployment from the chart | Watches only `arc-runners` | Argo CD |
| Runner pool (`AutoscalingRunnerSet`) | `arc-runners` | Name `lab-localstack` (also the label) | `minRunners` 0, `maxRunners` 2, group `Default` | Argo CD |
| Listener | `arc-systems` | Pod created by the controller for the pool | Long-polls GitHub for job requests | Controller |
| Runner (`EphemeralRunner` and pod) | `arc-runners` | Generated name | One job, then deleted; no Kubernetes token | Controller |

## Relationships

- The organization has one default runner group and one registration token; all non-public repositories can use the group. The static-site topic controls discovery only, not runner access.
- The `arc-runner-set` Application renders one pool; the pool references `arc-github-token` and registers in the runner group; the controller turns each job request into one runner.
- Runners read `lab-aws-credentials` as environment variables and nothing else from the cluster.

## Runner states

`Pending` (pod scheduled) → `Running` (registered, executing one job) → `Succeeded` (job finished) → deleted. Failure paths: `Failed` (could not register or the pod was evicted or OOM-killed) → deleted and replaced by the controller if jobs are still waiting. Pool states visible to the operator: registered and idle (no runners, or only the minimum), busy (up to the maximum), not registered (credential or GitHub problem).

## Validation rules

- Every credential field exists only in a Secret created outside Git (FR-003, FR-010, FR-015).
- `maxRunners` is at least `minRunners`; `minRunners` is 0 or more (FR-005).
- The pool must be in the namespace the controller watches (`arc-runners`).
- The removal order is: pool, then controller, then CRDs, Secrets and namespaces (FR-016).
