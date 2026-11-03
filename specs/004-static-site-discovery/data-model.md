# Data Model: Automatic Static-Site Discovery

| Entity | Kind / location | Key fields and rules |
|--------|-----------------|----------------------|
| Site repository | GitHub repository in the configured organization | Topic `backstage-static-site`; name matches `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`; not archived; `deploy/` on the default branch holds only `XStaticSite` manifests (top level, no recursion). |
| Discovery definition | `ApplicationSet static-site-discovery` in `argocd` | Organization, topic filter, `requeueAfterSeconds: 180`, `applicationsSync: create-update`, `preserveResourcesOnDeletion: true`, finalizer, `Prune=false` annotation, optional `tokenRef`. |
| Site Application | `Application site-<repo>` in `argocd` | Project `static-sites`; source = repository `deploy/` at `HEAD`; destination `site-<repo>`; automated sync with self-heal and prune; `CreateNamespace=true`; no deletion finalizer. |
| Site namespace | `Namespace site-<repo>` | Created by the first sync; not tracked by Argo CD, so never pruned. |
| Site project | `AppProject static-sites` | Source `https://github.com/<ORG>/*`; destination `site-*`; namespaced kind `XStaticSite` only; cluster kind `Namespace` named `site-*` only. |
| Own-namespace policy | `ValidatingAdmissionPolicy` and binding `static-site-own-namespace` | Denies `XStaticSite` in `site-*` whose tracking id does not start with `<namespace>:`. |
| Site | `XStaticSite` in `site-<repo>` | Spec 003 API; Crossplane composes the bucket resources; bucket named from the XR UID. |
| GitHub token (optional) | `Secret` in `argocd`, created with `kubectl` outside Git | Read-only; referenced by name and key only. |

## Lifecycle

```text
untagged/new repo ──tag──▶ discovered ──▶ Application created ──▶ namespace + XStaticSite ──▶ Ready
                                  │                                        ▲
 untag / archive / delete / rename│ stops generation only                  │ self-heal, prune of removed manifests
                                  ▼                                        │
                      Application + namespace + bucket retained ───────────┘ (until explicit removal)

explicit removal: untag → delete Application → delete XStaticSite (bucket removed) → delete namespace
```

## Non-discovery reasons and where each is visible

| Reason | Visible as |
|--------|------------|
| Wrong topic, other owner, archived | Not in `kubectl -n argocd get applications`; compare with the GitHub topic search |
| Invalid repository name | Same; the name fails the regex |
| Missing `deploy/` | Application with condition `ComparisonError: app path does not exist` |
| Invalid manifest | Application `OutOfSync`/sync failed with the validation error |
| Disallowed kind, cluster kind, namespace outside `site-*` | Application condition `... is not permitted in project static-sites` |
| Another site's namespace | Sync failed with the admission policy message |
| Unreadable repository or GitHub error | Application `ComparisonError` (clone failure) or ApplicationSet `ErrorOccurred` condition |
