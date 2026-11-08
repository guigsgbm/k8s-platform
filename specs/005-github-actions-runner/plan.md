# Implementation Plan: In-Cluster GitHub Actions Runners

**Branch**: `main` (specification directory is pinned by `.specify/feature.json`) | **Date**: 2026-11-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/005-github-actions-runner/spec.md`

## Summary

Use GitHub's official **Actions Runner Controller runner scale set** model, version `0.15.1`, delivered as two Argo CD Applications in `applications/`: the controller chart (`arc-controller`) and the runner scale set chart (`arc-runner-set`). The scale set registers one pool named `lab-localstack` at organization level in the existing `Default` runner group. That group allows all repositories in the organization except public repositories. The pool scales between 0 and 2 ephemeral, non-root runner pods that have no Kubernetes API access and reach LocalStack through its in-cluster DNS name.

Planning findings that shaped the design (details in [research.md](./research.md)):

- Both charts are published as OCI charts at `ghcr.io/actions/actions-runner-controller-charts` and are accepted by Argo CD `v3.5.5`. The chart's `crds/` ship four CRDs of 0.6–1.3 MB each; a server-side dry run showed that client-side apply fails on `autoscalingrunnersets` with `metadata.annotations: Too long`, so the controller Application **must** use `ServerSideApply=true`.
- With `flags.watchSingleNamespace=arc-runners` the controller chart renders **no ClusterRole or ClusterRoleBinding**, only namespaced Roles. This satisfies the least-privilege requirement.
- The scale set chart creates a dedicated no-permission ServiceAccount for runners. Together with `automountServiceAccountToken: false` the runner pod has no Kubernetes credential at all (minimum RBAC: none).
- The upstream runner image `ghcr.io/actions/actions-runner:2.338.2` includes git, curl, unzip and jq, but no AWS CLI and no Node.js on `PATH`. **Live-verified on the cluster**: a pod with the final security context (uid 1001, no privilege escalation, all capabilities dropped, no ServiceAccount token) installed the pinned AWS CLI `2.38.1` into the user's home, then created, uploaded, read back and removed an object in LocalStack through `http://localstack.localstack.svc.cluster.local:4566`; `sudo` was blocked. So no custom image is built; tools are installed per job (FR-011).
- The existing root Application now has `prune: true` (changed after specs 001–003). The runner Applications therefore carry the annotation `argocd.argoproj.io/sync-options: Prune=false` so the root cannot delete them, plus per-Application `prune: false` and no `resources-finalizer`, as spec 004 did for its ApplicationSet.
- The selected `Default` group is already configured for all repositories except public repositories. This intentionally makes every non-public organization repository eligible to use the pool; the static-site topic does not restrict runner access.

Cluster facts checked live: Kubernetes `v1.37.0`, one node with 8 CPU and about 16 GiB, currently using about 17% CPU and 9% memory in requests; Argo CD `v3.5.5`; `localstack` Service is ClusterIP.

## Technical Context

**Language/Version**: Kubernetes YAML and Markdown; Argo CD `v3.5.5` (existing). Helm charts `gha-runner-scale-set-controller` and `gha-runner-scale-set`, both `0.15.1` (app version `0.15.1`); runner image `ghcr.io/actions/actions-runner:2.338.2`.

**Primary Dependencies**: Argo CD OCI Helm source; the four `actions.github.com` CRDs shipped by the controller chart; GitHub fine-grained token (a GitHub App is the alternative) created outside Git; LocalStack from spec 001.

**Storage**: N/A. Runners use ephemeral container storage only.

**Testing**: `helm template` of both charts with the final values; server-side dry run of the CRDs and of the Application manifests; end-to-end verification with a throwaway job from a private organization repository (not committed); negative tests for a public repository, a repository outside the organization, a fork or pull-request trigger, an unknown label, a missing and a revoked credential, LocalStack down, node restart; removal procedure.

**Target Platform**: Existing Minikube cluster; Argo CD in `argocd`; controller in `arc-systems`; runners in `arc-runners`.

**Project Type**: Kubernetes platform GitOps repository

**Performance Goals**: Pool registered and idle within 10 minutes of merge (SC-001); a job starts on a new runner within 2 minutes from idle (SC-002).

**Constraints**: One organization, configured in one place; pinned chart, image and tool versions; no credential in Git; runners with no Kubernetes permission; `prune: false` on these Applications; no real AWS; no public exposure of LocalStack.

**Scale/Scope**: One lab; a maximum of 2 concurrent runners, each limited to 2 GiB of memory. At the maximum, runners reserve 0.5 CPU and 1 GiB in requests, which fits the node with large margin.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. Both Applications are in Git and self-heal. Only the Secrets and GitHub-side settings (token, runner group) are outside Git, by design. |
| II. Focus on platform infrastructure | Pass. Two Application manifests; the workflows that use the runners belong to site repositories. |
| III. Reconcile automatically, prune deliberately | **Documented exception.** Automated sync and self-heal are on, but `prune: false` for both Applications, with the reason in a manifest comment: pruning could terminate running jobs, orphan GitHub registrations, or delete CRDs and Secrets in use. The exception is the one the constitution allows when "deletion could cause data loss or otherwise requires separate lifecycle management". |
| IV. Keep secrets out of Git | Pass. Only Secret names and keys appear in manifests. The private key is created from a file, never from a literal. |
| V. Make deployments reproducible | Pass. Chart versions, runner image tag and the AWS CLI version are pinned; no `latest`. |
| VI. Review cluster impact | Pass. Cluster-scoped additions are only the four CRDs. The controller has namespaced Roles only. See Complexity Tracking for the two deliberate choices. |
| VII. Write only what the task needs | Pass. Upstream images only (no custom image or registry), no NetworkPolicy, no extra controllers, no Kubernetes mode or Docker-in-Docker. |

**Post-design re-check**: Pass, with the same single exception for Principle III.

## Project Structure

### Documentation (this feature)

```text
specs/005-github-actions-runner/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── runner-pool.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Created by /speckit-tasks, not by this plan
```

### Source Code (repository root)

```text
applications/
├── arc-controller.yaml     # New: Application for the controller chart (wave 0)
└── arc-runner-set.yaml     # New: Application for the runner scale set chart (wave 1)
README.md                   # Updated: runner operations, prerequisites, removal
```

**Structure Decision**: Follow the existing convention of one Application manifest per component directly under `applications/`, where the root Argo CD application syncs them. No new directory is needed because no raw Kubernetes manifest is added; Namespaces are created by `CreateNamespace=true` and everything else is rendered by the charts.

## Complexity Tracking

| Violation / exception | Why needed | Simpler alternative rejected because |
|-----------------------|------------|-------------------------------------|
| `prune: false` on both Applications (Principle III exception) | See the Constitution Check. The charts also put `actions.github.com/cleanup-protection` finalizers on their resources, so deletion order matters. | Pruning would let a Git edit delete the pool mid-job or delete the controller before the pool, leaving stuck finalizers. |
| Two Applications with a sync-wave and retry | The pool chart's `AutoscalingRunnerSet` needs the CRDs from the controller chart. | A single Application cannot install CRDs and use them in the same sync reliably, and the spec requires separate ordered stages (FR-012). |
| `ServerSideApply=true` on the controller Application | CRDs exceed the annotation size limit of client-side apply (verified). | There is no other way to install them through Argo CD, short of stripping the CRDs. |
