# Implementation Plan: Static-Site Repository Contract

**Branch**: `main` (feature directory: `006-site-repository-contract`) | **Date**: 2026-11-08 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/006-site-repository-contract/spec.md`

## Summary

Define a single repository contract for a discoverable and publishable static site. The platform dependency updates `XStaticSite` to derive `status.bucketName` and `status.websiteURL` from `metadata.name`, and removes the Composition-managed placeholder object so GitHub Actions owns published content. The contract, data model, and quickstart specify the repository files, catalog descriptor, workflow behavior, and one manual end-to-end run. No Backstage template, discovery change, or runner change is included.

## Technical Context

**Language/Version**: Kubernetes YAML, Markdown, GitHub Actions YAML; Crossplane `v2.4.2` (existing)

**Primary Dependencies**: Existing Crossplane `XStaticSite` Composition and Upbound AWS S3 provider `v2.4.0`; Argo CD ApplicationSet discovery; ephemeral `lab-localstack` runner; AWS CLI `2.38.1`; Node.js `24.11.1` installed with `actions/setup-node` pinned to a full immutable commit SHA

**Storage**: One ephemeral LocalStack S3 bucket per site, named `site-<repository-name>`; no migration of the existing UID-derived bucket contents

**Testing**: One manual end-to-end run with a hand-created private repository, plus the required server-side dry-run before applying the platform manifest. No additional automated, negative, repeated, load, or cleanup testing.

**Target Platform**: Existing Minikube cluster with Argo CD, Crossplane, and LocalStack; GitHub Actions runner pool labeled `lab-localstack`

**Project Type**: Kubernetes platform GitOps repository with an external static-site repository contract

**Performance Goals**: Bucket readiness wait bounded to 10 minutes; discovery and infrastructure readiness use the existing behavior from specs 003–005.

**Constraints**: Repository names match `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`; no real AWS, credentials, repository secrets, or Kubernetes access from the workflow; only the deterministic bucket-name and managed-placeholder platform dependencies change; no Backstage template or new build system.

**Scale/Scope**: One lab organization, one bucket and website per conforming site repository; a handful of sites.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. Platform configuration stays in Git; site files and deployment declaration live in the site repository. |
| II. Focus on platform infrastructure | Pass. The platform change is limited to the `XStaticSite` infrastructure contract; the external site repository remains the consuming workload. |
| III. Reconcile automatically, prune deliberately | Pass. Existing automated reconciliation remains. The lab's old UID-derived site bucket is deliberately recreated; deleting an `XStaticSite` still deletes its bucket and published content. |
| IV. Keep secrets out of Git | Pass. The runner supplies dummy LocalStack credentials; the site repository and workflow contain no credentials or repository secrets. |
| V. Make deployments reproducible | Pass. Existing provider versions remain pinned; AWS CLI and Node.js are pinned, and `setup-node` uses a full immutable commit SHA. |
| VI. Review changes for cluster impact | Pass with an explicit destructive lab reset. The existing `docs-demo` declaration uses the old bucket identity; its bucket is deleted and recreated without migrating content. The quickstart requires a server-side dry-run and documents the impact. |
| VII. Write only what the task needs | Pass. Reuse the current Composition, discovery, runner and LocalStack; add no package, build framework, template, or speculative validation. |
| VIII. Learning Lab: shortest path to the goal | Pass. The only feature validation is the single real end-to-end site run and the server-side dry-run; no guardrail, failure-injection, repeat, or cleanup rehearsal is added. |

## Project Structure

### Documentation (this feature)

```text
specs/006-site-repository-contract/
├── plan.md
├── research.md
├── data-model.md
├── contracts/
│   └── site-repository.md
├── quickstart.md
└── tasks.md             # Created by /speckit-tasks, not by this plan
```

### Source Code (repository root)

```text
applications/
└── crossplane-static-site-api.yaml  # Derive bucket identity from XR name; remove placeholder Object
README.md                            # Update static-site and runner descriptions for workflow publishing
```

`applications/docs-demo.yaml` remains the sample declaration. For rollout, its old UID-derived bucket is intentionally removed and recreated as described in [quickstart.md](./quickstart.md).

**Structure Decision**: Keep the infrastructure API in its existing manifest under `applications/`, which is watched by the root Argo CD application. Keep the consuming repository contract and its one-run validation guide alongside this feature's spec. No files are added to the Backstage repository because its template is out of scope.

**Related documentation updates**: Update the root `README.md` and the current operating references in `specs/003-static-site-hosting/data-model.md`, `specs/003-static-site-hosting/contracts/static-site.md`, and `specs/003-static-site-hosting/quickstart.md` so they describe deterministic repository-derived bucket names and workflow-owned content instead of the old UID-derived bucket and placeholder object.

## Complexity Tracking

No constitution violations or new dependencies.
