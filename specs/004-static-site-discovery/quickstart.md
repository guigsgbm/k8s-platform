# Quickstart: Automatic Static-Site Discovery

Run the commands from a workstation with `kubectl` access to the lab cluster. Replace `<ORG>` with the GitHub organization and `<repo>` with a site repository name. Never paste a token into a command line, manifest or note: create it only as described in Stage 5.

## Prerequisites

1. Spec 003 is working: `kubectl get xstaticsites -A` lists `docs-demo` as Ready, and the `xstaticsite-localstack` Composition exists.
2. Argo CD `v3.5.5` with the ApplicationSet controller: `kubectl -n argocd get deploy argocd-applicationset-controller`.
3. Tracking is annotation-based: `kubectl -n static-sites get xstaticsite docs-demo -o jsonpath='{.metadata.annotations}'` shows `argocd.argoproj.io/tracking-id`.
4. The site repositories live in a GitHub **organization** `<ORG>` (a personal account cannot be scanned). Check: `curl -s -o /dev/null -w '%{http_code}\n' https://api.github.com/orgs/<ORG>/repos` prints `200`.
5. A canary repository in `<ORG>` with a lowercase name, public, with `deploy/xstaticsite.yaml` (kind `XStaticSite`, no `metadata.namespace`), and **no** topic yet.

## Stage 1: Guardrails (project and policy)

1. Commit `applications/static-sites-project.yaml` and `applications/static-site-namespace-policy.yaml` from [contracts/discovery.md](./contracts/discovery.md) with `<ORG>` filled in, and push to `main`.
2. Verify: `kubectl -n argocd get appproject static-sites` and `kubectl get validatingadmissionpolicy static-site-own-namespace`; the root application `k8s-platform` is Synced and Healthy.

## Stage 2: Discovery definition

1. Commit `applications/static-site-discovery.yaml` and push.
2. Verify: `kubectl -n argocd get applicationset static-site-discovery -o jsonpath='{.status.conditions}'` shows no `ErrorOccurred=True`; no `site-*` Application exists yet.

## Stage 3: Canary site (user stories 1 and 2)

1. Add the topic `backstage-static-site` to the canary repository.
2. Within about 3 minutes plus sync time: `kubectl -n argocd get applications.argoproj.io | grep '^site-'` shows `site-<repo>` Synced and Healthy.
3. Ready check (Healthy does not wait for Crossplane): `kubectl -n site-<repo> get xstaticsites` shows `SYNCED=True`, `READY=True`. Zero commits were made to this repository.
4. Change `errorDocument` in the repository; within the next poll the live resource matches. Edit the live resource by hand and see it reverted.
5. Remove the manifest from `deploy/` and confirm it is pruned (this deletes the bucket); restore it afterwards.

## Stage 4: Guardrail and failure tests (user stories 3 and 4)

Run each against a throwaway tagged repository and record the Application message from the troubleshooting commands below.

| Test | Push to `deploy/` | Expected |
|------|-------------------|----------|
| Disallowed kind | a `Secret` or `ConfigMap` | `... not permitted in project static-sites`; nothing applied |
| Cluster-scoped kind | a `ClusterRole` | same rejection |
| Foreign namespace | `XStaticSite` with `metadata.namespace: default` | `namespace default is not permitted in project` |
| Another site's namespace | `XStaticSite` with `metadata.namespace: site-<other>` | sync fails with the admission message |
| Missing `deploy/` | remove the directory | Application shows `app path does not exist` |
| Invalid manifest | `indexDocument: ""` | sync failed with the validation error |
| Invalid name | repository named `My_Site` | no Application is created; documented in troubleshooting |
| Other owner | same topic on a repository outside `<ORG>` | never discovered |
| Retention | untag, archive, then delete the repository | Application, namespace, `XStaticSite` and bucket remain; deleted repository shows a clone error |
| Survival | restart `argocd-applicationset-controller`; then `kubectl -n argocd delete applicationset static-site-discovery`; then re-sync the root application | Applications and sites survive; the ApplicationSet is recreated |
| Rate limit | (read-only check) `curl -s https://api.github.com/rate_limit` | remaining requests are sufficient for `1 + N` per 3 minutes |

## Stage 5: Read token for private site repositories

Required: the runner group of spec 005 excludes public repositories, so site repositories are private. Discovery and Argo CD both need a read token, created outside Git. Create the Secrets **before** pushing the `tokenRef` in `applications/static-site-discovery.yaml`, otherwise the ApplicationSet reports `ErrorOccurred`.

1. Create a fine-grained GitHub token: resource owner `labs-guigsgbm`, all repositories, repository permissions **Contents: Read-only** and **Metadata: Read-only**, nothing else. Use a short expiry and note the renewal date.
2. Create both Secrets without echoing the value or storing it in shell history:

   ```sh
   set +x
   read -rsp "GitHub read token: " GH_TOKEN; echo
   kubectl -n argocd create secret generic github-discovery-token --from-literal=token="$GH_TOKEN"
   kubectl -n argocd create secret generic labs-guigsgbm-repo-creds \
     --from-literal=type=git \
     --from-literal=url=https://github.com/labs-guigsgbm \
     --from-literal=username=x-access-token \
     --from-literal=password="$GH_TOKEN"
   kubectl -n argocd label secret labs-guigsgbm-repo-creds argocd.argoproj.io/secret-type=repo-creds
   unset GH_TOKEN
   ```

3. Commit and push the `tokenRef` already present in `applications/static-site-discovery.yaml`.
4. Check: `kubectl -n argocd get applicationset static-site-discovery -o jsonpath='{.status.conditions}'` shows no `ErrorOccurred=True`.

## Troubleshooting

- List sites: `kubectl -n argocd get applications.argoproj.io | grep '^site-'`.
- A site is missing: check topic spelling, owner, archived status, and the name rule (`^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`); invalid names produce no Application.
- Application failing: `kubectl -n argocd get application site-<repo> -o jsonpath='{.status.conditions}{"\n"}{.status.operationState.message}{"\n"}'`.
- Discovery failing or rate-limited (`403 API rate limit exceeded`): `kubectl -n argocd get applicationset static-site-discovery -o jsonpath='{.status.conditions}'`. Existing Applications are unaffected; wait for the reset (`curl -s https://api.github.com/rate_limit`), raise `requeueAfterSeconds`, or add the token (Stage 5).
- Controller logs: `kubectl -n argocd logs deploy/argocd-applicationset-controller --tail=100`.

## Deliberate removal of a site

Untagging, archiving or deleting the repository removes nothing. The root application and the site Applications have `prune: true`, so also note that deleting a manifest from `deploy/` deletes that resource and its bucket on the next sync.

1. Untag or archive the repository so the ApplicationSet cannot recreate the Application.
2. Delete the Application (it has no deletion finalizer, so its resources stay): `kubectl -n argocd delete application site-<repo>`. Confirm: `kubectl -n argocd get application site-<repo>` reports not found.
3. Delete the site, which removes its bucket and contents (irrecoverable; LocalStack state is ephemeral anyway): `kubectl -n site-<repo> delete xstaticsites --all`, then wait until `kubectl -n site-<repo> get xstaticsites,buckets.s3.aws.m.upbound.io` is empty.
4. Delete the namespace: `kubectl delete namespace site-<repo>`.

To retire discovery entirely, delete the three manifests from `applications/` after removing sites you want gone; the ApplicationSet carries `Prune=false`, so also run `kubectl -n argocd delete applicationset static-site-discovery` (Applications and resources survive), then delete each Application and site as above if wanted.

## Constitution note

Pruning stays enabled as the constitution requires. Resources are protected from repository lifecycle events by the absence of deletion finalizers and by the ApplicationSet's delete policy, not by disabling pruning.
