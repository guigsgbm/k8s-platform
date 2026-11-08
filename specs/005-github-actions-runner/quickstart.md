# Quickstart: In-Cluster GitHub Actions Runners

Validation guide for [spec.md](./spec.md). The manifest content is produced during implementation (`/speckit-tasks`, `/speckit-implement`). Values are in [contracts/runner-pool.md](./contracts/runner-pool.md). Commands assume kubectl context `minikube` and Argo CD in `argocd`.

## Prerequisites

- Specs 001–004 in place: Argo CD healthy and root application `k8s-platform` `Synced`, LocalStack `Running` in `localstack`.
- The GitHub **organization** `labs-guigsgbm` (verified: it exists and is of type Organization) and owner permission to create runner groups in it and to allow fine-grained personal access tokens (Organization → Settings → Personal access tokens: allow fine-grained tokens; if approval is required, approve the token request).
- Outbound access from the node to `github.com`, `api.github.com`, `*.actions.githubusercontent.com` and `ghcr.io`.

### GitHub token (once, outside Git)

The lab uses a **fine-grained personal access token** (a GitHub App is the preferred alternative and only changes the Secret contents; see the contract).

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token.
2. **Resource owner**: `labs-guigsgbm`. **Repository access**: public repositories only (the organization-level runner permissions do not depend on a repository).
3. **Organization permissions**: **Self-hosted runners: Read and write** and **Administration: Read**. Nothing else.
4. **Expiration**: the shortest lifetime you can maintain, for example 90 days. When it expires the pool stops registering (see Troubleshooting); write the renewal date down. Copy the token once; it is shown only at creation.
5. The token belongs to your user: if you leave the organization or lose owner rights, it stops working.

### Runner group and Actions settings (once, outside Git)

1. Organization → Settings → Actions → Runner groups → verify the existing `Default` group is set to **All repositories, excluding public repositories**. This gives every non-public repository in `labs-guigsgbm` access to the runner; the static-site topic does not limit access. Do not create or delete a separate group.
2. Organization → Settings → Actions → General: require approval for fork pull-request workflows, including outside collaborators.

### Secrets (outside Git; no secret in shell history)

```sh
kubectl create namespace arc-runners

set +x
umask 077
secret_dir="$(mktemp -d)"
trap 'rm -f -- "$secret_dir/github_token" "$secret_dir/access-key-id" "$secret_dir/secret-access-key"; rmdir -- "$secret_dir"' EXIT

read -rsp "GitHub token: " GH_TOKEN; echo
printf '%s' "$GH_TOKEN" > "$secret_dir/github_token"
kubectl create secret generic arc-github-token -n arc-runners \
  --from-file=github_token="$secret_dir/github_token"
unset GH_TOKEN

read -rsp "Access key id: " AK; echo
read -rsp "Secret access key: " SK; echo
printf '%s' "$AK" > "$secret_dir/access-key-id"
printf '%s' "$SK" > "$secret_dir/secret-access-key"
kubectl create secret generic lab-aws-credentials -n arc-runners \
  --from-file=access-key-id="$secret_dir/access-key-id" \
  --from-file=secret-access-key="$secret_dir/secret-access-key"
unset AK SK
```

Use only dummy LocalStack values, never real AWS credentials. LocalStack accepts any non-empty dummy pair; use `test` for both keys unless the lab already uses another dummy pair. These values do not come from GitHub or AWS. The prompts do not echo; temporary files have owner-only permissions and are removed when the shell exits. Never type credential values as command arguments or paste them into GitHub Actions logs.

## Stage 1: Review before merging

```sh
helm template arc-controller oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version 0.15.1 -n arc-systems --set flags.watchSingleNamespace=arc-runners | grep -E '^kind:' | sort | uniq -c
```

Expected: no `ClusterRole` or `ClusterRoleBinding`. Then render the pool chart with the values from the contract and confirm: image `ghcr.io/actions/actions-runner:2.338.2`, `minRunners: 0`, `maxRunners: 2`, the no-permission ServiceAccount, and `automountServiceAccountToken: false`. Confirm `githubConfigUrl` is `https://github.com/labs-guigsgbm` and that no credential value appears anywhere in the diff.

## Stage 2: Controller (user story 1)

Merge only `applications/arc-controller.yaml` first (or both; the pool Application retries until the CRDs exist).

```sh
kubectl get application arc-controller -n argocd
kubectl get pods -n arc-systems
kubectl get crd | grep actions.github.com
kubectl get clusterrole | grep -i gha-rs   # expected: nothing
```

**Expected**: Application `Synced`/`Healthy`, controller pod `Running`, four `actions.github.com` CRDs, no cluster role.

## Stage 3: Runner pool (user story 1)

Merge `applications/arc-runner-set.yaml`.

```sh
kubectl get application arc-runner-set -n argocd
kubectl get autoscalingrunnerset -n arc-runners
kubectl get pods -n arc-systems     # a listener pod appears
kubectl get pods -n arc-runners     # expected: no runner pods while idle (minimum 0)
```

**Expected**: Application `Healthy`; in the organization (Settings → Actions → Runner groups → `Default`) the pool `lab-localstack` is listed with no running jobs, within 10 minutes (SC-001), with no manual registration step.

## Stage 4: Behavior checks (user story 2, SC-002/SC-003)

The real workflow is a later spec. Use a throwaway workflow in a non-public repository in `labs-guigsgbm` (do not commit it to this repository), triggered by `workflow_dispatch`, with the guard from the contract and `runs-on: lab-localstack`. It performs:

```sh
curl -fsSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.38.1.zip -o /tmp/aws.zip
unzip -q /tmp/aws.zip -d /tmp && /tmp/aws/install -i "$HOME/aws-cli" -b "$HOME/bin"
export PATH="$HOME/bin:$PATH"
aws s3 ls
```

While it runs: `kubectl get pods -n arc-runners` shows one runner pod. **Expected**: the job succeeds; the runner pod is gone within 2 minutes after the job; two jobs started together run on two different pods (maximum 2); a third waits in the queue. The job log contains no credential value. Confirm the pod has no cluster access while a job runs:

```sh
kubectl get pod -n arc-runners -o jsonpath='{.items[0].spec.automountServiceAccountToken}{" "}{.items[0].spec.securityContext}{"\n"}'
```

**Expected**: `false` and `runAsNonRoot: true`, `runAsUser: 1001`.

## Stage 5: Access-control tests (SC-004)

| Test | Expected |
|------|----------|
| Workflow from a public repository in the organization requests `lab-localstack` | Not run on this pool because the `Default` group excludes public repositories |
| Pull request from a fork of a private organization repository | Not run on these runners (organization approval settings); the job-level guard also skips it |
| `pull_request` workflow from a private organization repository that requests the label | Skipped by the guard |
| Repository from another organization | It cannot use this organization-level runner pool |
| Workflow from another non-public repository in `labs-guigsgbm` | Eligible to use the pool; the static-site topic is not an access restriction |

## Stage 6: Failure and recovery tests (user story 3)

| Test | How | Expected |
|------|-----|----------|
| Missing credential at install | Delete `arc-github-token`, restart the controller, wait | Pool not registered; controller log shows the GitHub error; no secret value in logs. Recreate the Secret and restart the controller (`kubectl rollout restart deploy/arc-controller-gha-rs-controller -n arc-systems`): pool registers again |
| Revoked or expired credential | Revoke the token in GitHub (or wait for its expiry) | Same as above; create a new token, replace the Secret (delete and create again) and restart the controller |
| LocalStack down | `kubectl scale deploy/localstack -n localstack --replicas=0` then run the throwaway job | The runner starts; the S3 step fails with a connection error. Scale LocalStack back to 1 |
| OOM or eviction mid-job | Run a job that allocates more than 2 GiB | The job fails; the pod is replaced; a re-run on a fresh runner works |
| Unknown label | `runs-on: lab-typo` | Job stays queued (visible in the workflow run) and never runs |
| Capacity | Start 4 jobs | At most 2 runner pods; the other jobs queue and run as pods free up; LocalStack, Argo CD and Crossplane stay Healthy |
| Minikube restart | `minikube stop` then `minikube start` | Controller and listener return; pool shows registered and idle within 10 minutes; no leftover `Running` runners; offline entries (if any) in the organization are harmless and can be deleted from the runner group page; a new job runs (SC-008) |

## Troubleshooting (user story 3, SC-007)

```sh
kubectl get application arc-controller arc-runner-set -n argocd
kubectl get autoscalingrunnerset,ephemeralrunnerset,ephemeralrunner -n arc-runners
kubectl get pods -n arc-systems
kubectl get pods -n arc-runners
kubectl logs deploy/arc-controller-gha-rs-controller -n arc-systems --tail=100
kubectl describe autoscalingrunnerset lab-localstack -n arc-runners
```

| Symptom | Likely cause | Check |
|---------|--------------|-------|
| Pool not listed in GitHub; controller logs a 401/403/404 | Token missing, expired, revoked, wrong resource owner, missing permission, or the organization requires approval for the token | `kubectl get secret arc-github-token -n arc-runners`; the token's expiry, owner and permissions; the organization's token requests |
| Pool not listed; controller logs a runner group error | The configured group name is invalid or the `Default` group is unavailable | Group name in the organization and in the manifest |
| Pool Application `Missing`/`OutOfSync` with unknown kind | CRDs not installed yet | `kubectl get crd \| grep actions.github.com`; the Application retries |
| CRD sync error `annotations: Too long` | `ServerSideApply=true` missing | Sync options of `arc-controller` |
| Job queued, pool idle | Label typo, public repository excluded by `Default`, or repository outside the organization | `runs-on` value; repository visibility and owner |
| Job queued, runner pods at 2 | Maximum concurrency reached | `kubectl get pods -n arc-runners`; `maxRunners` |
| Job queued and nothing in `arc-runners` | Listener down or GitHub unreachable | `kubectl get pods -n arc-systems`; listener and controller logs |
| Runner pod `Pending` | Node out of CPU or memory | `kubectl describe pod`; node allocation |
| Runner pod `OOMKilled` | Job exceeds the 2 GiB limit | `kubectl describe pod`; reduce the build or raise the limit in Git |
| Step fails connecting to `localstack.localstack.svc.cluster.local` | LocalStack down, restarted, or DNS problem | `kubectl get pods -n localstack`; spec 001 quickstart |
| `sudo` or package install fails in a job | Intended: runners cannot escalate | Install tools into the user's home |

GitHub cancels jobs that stay queued for 24 hours.

## Pause or scale to zero (user story 4)

The pool already idles at zero runners. To stop all new runners without removing anything, set `maxRunners: 0` in `applications/arc-runner-set.yaml` and merge: jobs queue and nothing is created. Revert the change to resume.

## Deliberate removal (user story 4, SC-009)

Argo CD does not remove this feature when the manifests are deleted from Git (`Prune=false`). Remove it in this order.

1. Stop new jobs and wait for running ones: set `maxRunners: 0`, then check that `kubectl get ephemeralrunner -n arc-runners` is empty.
2. Delete `applications/arc-runner-set.yaml` and `applications/arc-controller.yaml` from Git. The Applications and their resources stay.
3. Delete the pool first, with its resources:

   ```sh
   kubectl patch application arc-runner-set -n argocd --type merge \
     -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}'
   kubectl delete application arc-runner-set -n argocd
   kubectl get autoscalingrunnerset -n arc-runners   # wait until empty
   ```

4. Then the controller, only after the pool is gone (its finalizers need the controller):

   ```sh
   kubectl patch application arc-controller -n argocd --type merge \
     -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}'
   kubectl delete application arc-controller -n argocd
   ```

5. Remove what remains outside Argo CD's cascade: `kubectl delete crd autoscalinglisteners.actions.github.com autoscalingrunnersets.actions.github.com ephemeralrunners.actions.github.com ephemeralrunnersets.actions.github.com`, then `kubectl delete secret arc-github-token lab-aws-credentials -n arc-runners` and `kubectl delete namespace arc-runners arc-systems`.
6. In GitHub: delete the pool and any offline runners from the `Default` group, then revoke the token. Do not delete the built-in `Default` group.

**Check**: `kubectl get ns,crd | grep -E 'arc-|actions.github.com'` returns nothing; the organization shows no runner; LocalStack and the sites are unaffected.

If a resource is stuck deleting because the controller is already gone, remove its `actions.github.com/cleanup-protection` finalizer with `kubectl patch ... --type merge -p '{"metadata":{"finalizers":null}}'`.

## Security notes

- Anyone who can push to any non-public repository in `labs-guigsgbm` can run code on a runner. This is broader than limiting access to static-site repositories. Public repositories are excluded by the `Default` group, fork/outside-collaborator workflows require approval, and pull-request triggers must never use the label.
- Runners hold only the dummy LocalStack credentials and no Kubernetes permission; the blast radius is LocalStack.
- Search for leaks before merging: `git grep -nE 'BEGIN (RSA )?PRIVATE KEY|ghp_|github_pat_|AKIA'` must return nothing (SC-005).

## Constitution note

Principle III: pruning is disabled for both Applications, with the reason in each manifest (jobs in flight, GitHub registrations, finalizer ordering, CRDs and Secrets in use); the deliberate removal above replaces it.
