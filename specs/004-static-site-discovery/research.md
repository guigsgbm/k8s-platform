# Research: Automatic Static-Site Discovery

Sources: live cluster inspection (Argo CD `v3.5.5`, Kubernetes `v1.37.0`), Argo CD `v3.5.5` documentation and source (`applicationset/services/scm_provider`, `applicationset/controllers`, `controller/sync.go`), gitops-engine `sync_context.go`, GitHub REST API.

## Decisions

### 1. Discovery mechanism: built-in SCM Provider generator, owner is an organization

- **Decision**: ApplicationSet `scmProvider.github` with `organization`, `cloneProtocol: https`, `excludeArchivedRepos: true`, `requeueAfterSeconds: 180`, one filter combining `repositoryMatch` and `labelMatch: ^backstage-static-site$` (GitHub topics are exposed as labels).
- **Evidence**: `ListRepos` calls only `Repositories.ListByOrg`. `GET /orgs/guigsgbm/repos` returns 404 and `/users/guigsgbm` is a user account. The user chose to use an organization.
- **Alternatives**: a lookup service plus the ApplicationSet plugin generator for a user account (rejected by the user: extra workload); Git generator with committed files (violates "no change to this repository per site").

### 2. Application shape

- **Decision**: name and namespace `site-{{.repository}}`; `source.repoURL: {{.url}}`, `targetRevision: HEAD` (default branch, survives a default-branch rename), `path: deploy`; project `static-sites`; `automated {selfHeal: true, prune: true}`; `CreateNamespace=true`.
- **Namespace rule**: `repositoryMatch: ^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$` keeps `site-` plus the name a valid, at most 63-character namespace. Invalid names are filtered out with no normalization (clarification), so two sites can never share a namespace.
- **Rationale for no `pathsExist` filter**: it costs one extra GitHub request per repository per cycle. Without it a repository lacking `deploy/` still gets an Application that shows `app path does not exist`, which is more visible than a silent filter.

### 3. Retention on repository removal (clarification: Application stays)

- **Decision**: `syncPolicy.applicationsSync: create-update` (the controller never deletes Applications) plus `preserveResourcesOnDeletion: true` (generated Applications get no `resources-finalizer`, so deleting an Application never cascades to its resources). The `Namespace` created by `CreateNamespace` is not tracked, so pruning never removes it.
- **Effect**: untagging, archiving, deleting or renaming a repository stops generation; the existing Application keeps syncing (a deleted or unreadable repository shows as a sync error) and nothing is deleted. A renamed repository becomes a second site with its own namespace and bucket (bucket names derive from the XR UID in spec 003).
- **Template propagation**: `create-update` still pushes template changes to existing Applications.

### 4. ApplicationSet survival (FR-014)

- **Deletion by `kubectl delete`**: the controller only gets to run its "release the Applications" logic (`removeOwnerReferencesOnDeleteAppSet`) while the ApplicationSet is held by the finalizer `resources-finalizer.argocd.argoproj.io`; with `create-update` the policy disallows delete, so owner references are removed and Applications survive.
- **Pruning by the root application**: the root Application prunes with foreground propagation, where Applications may be garbage-collected before the controller reacts. The annotation `argocd.argoproj.io/sync-options: Prune=false` on the ApplicationSet prevents the root application from deleting it at all.
- **Argo CD restart**: Applications and namespaces are cluster objects and survive; the generator resumes on restart.

### 5. Project guardrails

- **Decision**: AppProject `static-sites` with `sourceRepos: https://github.com/<ORG>/*`; `destinations: {server: https://kubernetes.default.svc, namespace: site-*}`; `namespaceResourceWhitelist: [platform.k8s-platform.io/XStaticSite]` (an empty whitelist would allow every namespaced kind); `clusterResourceWhitelist: [{group: '', kind: Namespace, name: site-*}]`.
- **Why the Namespace allowance**: `gitops-engine` `autoCreateNamespace` adds the Namespace to the task list and `controller/sync.go` `validateSyncPermissions` checks every task with `IsGroupKindNamePermitted`. An empty cluster whitelist would therefore block `CreateNamespace`. The whitelist supports a `name` glob (confirmed with `kubectl explain appproject.spec.clusterResourceWhitelist`).
- **Rejection visibility**: disallowed kind, cluster kind or namespace outside `site-*` fail with `resource <group>:<kind> is not permitted in project static-sites` or `namespace <ns> is not permitted in project`, shown on the Application's sync result and conditions.
- **Residual risk**: a repository can include a `Namespace` manifest named `site-*`, which the project permits; this creates or touches only an empty `site-*` namespace and no other kind.

### 6. Own-namespace enforcement

- **Problem**: project destinations are shared, so site A may declare `metadata.namespace: site-b` and pass the project.
- **Decision**: ValidatingAdmissionPolicy `static-site-own-namespace` (Kubernetes built-in, GA in the live `v1.37.0`). For `XStaticSite` create/update in a namespace starting with `site-`, if the `argocd.argoproj.io/tracking-id` annotation exists it must start with `<namespace>:`. The tracking id begins with the Application name, which equals the namespace. Argo CD sets that annotation itself on every resource it applies. `docs-demo` in `static-sites` is outside the match condition.
- **Limitation**: a manual `kubectl` create without the annotation is allowed (an operator action). Crossplane status updates use the status subresource and are not matched.
- **Alternatives**: AppProject per site (cannot be generated by an ApplicationSet); Kustomize namespace override (requires `kustomization.yaml` in every repository); a policy engine such as Kyverno (new dependency).

### 7. GitHub rate limit

- **Cost per cycle**: `ceil(repos / 100)` list calls plus one `GetBranch` per tagged repository; `repositoryMatch`/`labelMatch` use data from the list response. So a cycle costs `1 + N` for up to 100 repositories.
- **Unauthenticated** (60 requests/hour per source IP, shared by anything else on that IP):

| Interval | Cycles/hour | N=1 | N=2 | N=3 | N=4 |
|----------|-------------|-----|-----|-----|-----|
| 180 s | 20 | 40 | 60 (limit) | 80 (over) | 100 (over) |
| 300 s | 12 | 24 | 36 | 48 | 60 (limit) |

- **Authenticated** (5000/hour): at 180 s, 20 × (1 + N) ≤ 5000 allows more than 200 tagged repositories.
- **Decision**: ship without a token at 180 s (fits one tagged repository with headroom). From the second site, create the token Secret outside Git and add `tokenRef` (name and key only) in a Git change, or raise `requeueAfterSeconds`. A rate-limit error makes the generator fail the whole cycle; with no delete policy, existing Applications and sites are untouched and the error appears in the ApplicationSet `ErrorOccurred` condition.
- **Token scope**: a fine-grained read-only token is enough for private repositories; Argo CD itself also needs repository credentials to clone private repositories (`argocd-repo-creds` Secret, outside Git).

### 8. Health and readiness reporting

- No health check is configured for the `platform.k8s-platform.io` group, so a site Application reports `Healthy` when applied, not when the `XStaticSite` is `Ready`. Quickstart verifies both. Adding a custom health check would modify `argocd-cm`, which is installed outside this repository, so it is out of scope.

### 9. Resource tracking

- Annotation tracking is already effective (default in Argo CD 3.x; `application.resourceTrackingMethod` is unset). `docs-demo` carries `argocd.argoproj.io/tracking-id`. Nothing to change; the admission policy relies on it.

## Not in the plan

- No webhook (Minikube is not reachable from GitHub), no `pathsExist` filter, no per-site project, no custom health check, no notifications.
