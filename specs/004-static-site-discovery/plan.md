# Implementation Plan: Automatic Static-Site Discovery

**Branch**: `main` (specification directory is pinned by `.specify/feature.json`) | **Date**: 2026-11-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/004-static-site-discovery/spec.md`

## Summary

Use Argo CD's built-in ApplicationSet **SCM Provider generator** to scan one GitHub organization for repositories with the topic `backstage-static-site`. For each match it generates an Application `site-<repo>` that syncs the repository's `deploy/` directory (default branch) into namespace `site-<repo>`. Three manifests under `applications/` deliver it: a restrictive AppProject, the ApplicationSet, and a small admission policy. No new workload, image or chart is installed.

Planning findings that shaped the design (details in [research.md](./research.md)):

- The GitHub SCM generator only lists **organizations**; `guigsgbm` is a personal account (the API returns 404 for `/orgs/guigsgbm/repos`). Decision (user-confirmed): the owner is a GitHub organization.
- An AppProject cannot pin each Application to its own namespace, and Argo CD validates the auto-created Namespace against the project. So the project allows `site-*` namespaces plus the Namespace kind named `site-*`, and a ValidatingAdmissionPolicy enforces "own namespace only".
- Unauthenticated GitHub access (60 requests/hour) supports one tagged repository at a 3-minute interval; more sites need an out-of-Git token.

Live-verified on the cluster: Argo CD `v3.5.5`, annotation resource tracking (`argocd.argoproj.io/tracking-id` on `docs-demo`), `ApplicationSet` CRD and controller present, `admissionregistration.k8s.io/v1` ValidatingAdmissionPolicy served (Kubernetes `v1.37.0`), draft manifests accepted by `kubectl apply --dry-run=server`.

## Technical Context

**Language/Version**: Kubernetes YAML and Markdown; Argo CD `v3.5.5` (existing, installed outside this repository)

**Primary Dependencies**: Argo CD ApplicationSet controller (SCM Provider generator), AppProject, ValidatingAdmissionPolicy (built into Kubernetes); Crossplane `XStaticSite` API from spec 003. No new upstream package, image or chart, so nothing new to pin.

**Storage**: N/A. Site buckets remain ephemeral LocalStack storage (spec 003).

**Testing**: Server-side dry-run of manifests; canary repository tagged end to end; negative tests for disallowed kind, cluster-scoped kind, foreign namespace, another site's namespace, missing `deploy/`, invalid manifest, invalid repository name, other-owner topic; restart and ApplicationSet-deletion survival; untag/archive/delete retention; rate-limit failure visibility. See [quickstart.md](./quickstart.md).

**Target Platform**: Existing Minikube cluster; Argo CD in `argocd`.

**Project Type**: Kubernetes platform GitOps repository

**Performance Goals**: Tagged repository reflected in an Application within 3 minutes plus sync time; Healthy Application and Ready `XStaticSite` within 5 minutes (SC-001).

**Constraints**: One organization; topic is the only discovery criterion; site namespace `site-<repo>` with no name normalization; `prune: true` with no cascade deletion on repository removal; annotation tracking; no credential in Git; no real AWS; no new public route.

**Scale/Scope**: One lab; a handful of site repositories. GitHub cost per cycle is `1 + N` requests for up to 100 repositories in the organization (N = tagged repositories).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Assessment |
|-----------|------------|
| I. Git is the source of truth | Pass. Discovery, project and policy are Git-managed; each site's desired state lives in its own Git repository. |
| II. Focus on platform infrastructure | Pass. The platform owns discovery and guardrails; site content and templates stay out of scope. |
| III. Reconcile automatically, prune deliberately | Pass. Site Applications use automated sync, self-heal and `prune: true`. Deletion of the Applications' resources on repository removal is prevented separately (no resources finalizer), and the ApplicationSet is protected from pruning and deletion cascades. Quickstart states that removing an `XStaticSite` from `deploy/` deletes its bucket. |
| IV. Keep secrets out of Git | Pass. Public repositories need no credential. The optional GitHub token is a Secret created with `kubectl` outside Git and referenced by name only. |
| V. Make deployments reproducible | Pass. No new upstream dependency; the existing Argo CD version is recorded above. The repository source is pinned to the organization via the project's `sourceRepos`. |
| VI. Review cluster impact | Pass. See Complexity Tracking for the one cluster-scoped allowance. The ApplicationSet can create Applications that deploy into `site-*` namespaces only. |
| VII. Write only what the task needs | Pass. Built-in generator, no service, no extra filters: `deploy/` existence is shown by the Application itself instead of an extra per-repository GitHub call. |

**Post-design re-check**: Pass. Each of the three manifests exists to satisfy a numbered requirement (see [contracts/discovery.md](./contracts/discovery.md)).

## Project Structure

### Documentation (this feature)

```text
specs/004-static-site-discovery/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── discovery.md
├── checklists/
│   └── requirements.md
└── tasks.md             # Created by /speckit-tasks, not by this plan
```

### Source Code (repository root)

```text
applications/
├── static-sites-project.yaml            # New: AppProject `static-sites`
├── static-site-discovery.yaml           # New: ApplicationSet (SCM Provider generator)
└── static-site-namespace-policy.yaml    # New: ValidatingAdmissionPolicy and binding
README.md                                # Updated: discovery operations, rate limit, removal
```

**Structure Decision**: Follow spec 003 and place the manifests directly under `applications/`, where the root Argo CD application already syncs them into `argocd` (cluster-scoped objects are unaffected by that destination namespace). Roll out in stages: project and policy first, then the ApplicationSet with one canary repository, then optional token. The organization name is the only value to supply at implementation; it appears in the project's `sourceRepos` and the generator's `organization`.

## Complexity Tracking

| Violation / exception | Why needed | Simpler alternative rejected because |
|-----------------------|------------|-------------------------------------|
| Project allows cluster kind `Namespace` named `site-*` (spec asked for no cluster-scoped resources) | Argo CD validates the namespace created by `CreateNamespace=true` against the project, so without it no site namespace can be created | Pre-creating namespaces needs a manual commit per site, defeating the feature. Name-restricted allowance limits the exposure to `site-*` namespaces. |
| ValidatingAdmissionPolicy `static-site-own-namespace` | A project's destination list is shared by all site Applications, so it cannot stop site A from deploying into `site-B`; the spec requires that rejection | A project per site cannot be generated by an ApplicationSet; forcing the namespace through Kustomize would require every repository to ship a `kustomization.yaml` |
