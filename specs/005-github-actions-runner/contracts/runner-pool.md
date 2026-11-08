# Contract: Runner Pool

What the platform promises to site workflows and what the two Application manifests must contain. Rationale is in [research.md](./../research.md); objects are in [data-model.md](./../data-model.md).

## 1. For site workflows

| Item | Value |
|------|-------|
| `runs-on` | `lab-localstack` (single label; the pool name) |
| Who may use it | Any repository in `labs-guigsgbm` except public repositories; this is the `Default` group's current policy |
| Allowed triggers | `push` to the default branch; `workflow_dispatch`. Never `pull_request`, `pull_request_target`, or anything triggered by a fork |
| One job per runner | Fresh pod per job; nothing persists between jobs |
| Environment provided | `AWS_ENDPOINT_URL=http://localstack.localstack.svc.cluster.local:4566`, `AWS_DEFAULT_REGION=us-east-1`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (dummy lab values) |
| Tools present | git, curl, unzip, jq. Everything else (AWS CLI `2.38.1`, Node.js) is installed in the job, into the user's home; `sudo` does not work |
| Kubernetes access | None. Site readiness is checked over S3, for example that the site's bucket exists |
| Do not | print the environment, or write the credential variables to logs or artifacts |

Reference guard for any job that uses the label (documentation snippet only; no workflow file is delivered by this feature):

```yaml
jobs:
  publish:
    if: >-
      github.repository_owner == 'labs-guigsgbm' &&
      (github.event_name == 'workflow_dispatch' ||
       (github.event_name == 'push' && github.ref == format('refs/heads/{0}', github.event.repository.default_branch)))
    runs-on: lab-localstack
```

## 2. `applications/arc-controller.yaml`

| Field | Value |
|-------|-------|
| Kind / name / namespace | `Application` / `arc-controller` / `argocd` |
| Annotation | `argocd.argoproj.io/sync-options: Prune=false` |
| Source | `repoURL: ghcr.io/actions/actions-runner-controller-charts`, `chart: gha-runner-scale-set-controller`, `targetRevision: 0.15.1`, `releaseName: arc-controller` |
| Values | `flags.watchSingleNamespace: arc-runners`; `flags.logLevel: info`; `resources` requests `50m` CPU / `64Mi`, memory limit `256Mi` |
| Destination | `https://kubernetes.default.svc`, namespace `arc-systems` |
| Sync policy | `automated` with `selfHeal: true`, `prune: false`; `syncOptions`: `CreateNamespace=true`, `ServerSideApply=true` |
| Finalizer | None (no `resources-finalizer.argocd.argoproj.io`) |
| Comment | Why pruning is disabled, and the removal order |

## 3. `applications/arc-runner-set.yaml`

| Field | Value |
|-------|-------|
| Kind / name / namespace | `Application` / `arc-runner-set` / `argocd` |
| Annotations | `argocd.argoproj.io/sync-wave: "1"`; `argocd.argoproj.io/sync-options: Prune=false` |
| Source | `repoURL: ghcr.io/actions/actions-runner-controller-charts`, `chart: gha-runner-scale-set`, `targetRevision: 0.15.1`, `releaseName: lab-localstack` |
| Values: pool | `githubConfigUrl: https://github.com/labs-guigsgbm` (the only place the organization is set); `githubConfigSecret: arc-github-token`; `runnerScaleSetName: lab-localstack`; `runnerGroup: Default`; `minRunners: 0`; `maxRunners: 2`; `controllerServiceAccount: {namespace: arc-systems, name: arc-controller-gha-rs-controller}` |
| Values: pod | `automountServiceAccountToken: false`; pod `securityContext` `runAsNonRoot: true`, `runAsUser: 1001`, `seccompProfile.type: RuntimeDefault`; container `runner` with image `ghcr.io/actions/actions-runner:2.338.2`, command `["/home/runner/run.sh"]`, container `securityContext` `allowPrivilegeEscalation: false`, `capabilities.drop: ["ALL"]`; resources requests `250m` / `512Mi`, memory limit `2Gi` |
| Values: env | `AWS_ENDPOINT_URL`, `AWS_DEFAULT_REGION` plain; `AWS_ACCESS_KEY_ID` from Secret `lab-aws-credentials` key `access-key-id`; `AWS_SECRET_ACCESS_KEY` from the same Secret key `secret-access-key` |
| Destination | `https://kubernetes.default.svc`, namespace `arc-runners` |
| Sync policy | `automated` with `selfHeal: true`, `prune: false`; `syncOptions`: `CreateNamespace=true`, `ServerSideApply=true`; `retry` with exponential backoff (a limited number of attempts, for example 5, from 30 seconds up to 5 minutes) |
| Finalizer | None |

The lab organization is `labs-guigsgbm`; `githubConfigUrl` is the only place it is set in the manifests (the workflow guard above carries it too). The chart's own `Role`/`RoleBinding` in `arc-runners` and the no-permission ServiceAccount are rendered by the chart and are not written by hand.

## 4. Secrets (created outside Git, in `arc-runners`)

| Secret | Keys | Created from |
|--------|------|--------------|
| `arc-github-token` | `github_token` | A fine-grained personal access token (resource owner `labs-guigsgbm`), read from an interactive prompt, never typed as an argument |
| `lab-aws-credentials` | `access-key-id`, `secret-access-key` | The lab's dummy values, from files or an interactive prompt; never from workflow files |

Alternative (preferred in general, not used in this lab): a GitHub App with Organization permission "Self-hosted runners" Read and write and Repository permission "Metadata" Read-only, stored as `github_app_id`, `github_app_installation_id` and `github_app_private_key` in a Secret; only the Secret name in `githubConfigSecret` changes.

## 5. Kubernetes permissions

| Subject | Permission |
|---------|------------|
| Runner pods | **None.** No ServiceAccount token is mounted; the bound ServiceAccount has no Role or ClusterRole |
| Controller | Namespaced `Role` and `RoleBinding` objects rendered by the chart in `arc-systems` and `arc-runners`; no `ClusterRole` because `watchSingleNamespace` is set |
| This feature, cluster-scoped | The four `actions.github.com` CRDs only |

## 6. GitHub-side settings (outside Git)

| Setting | Value |
|---------|-------|
| GitHub token | Fine-grained personal access token, resource owner `labs-guigsgbm`; Organization permissions "Self-hosted runners" Read and write and "Administration" Read; short expiry (for example 90 days); the organization must allow fine-grained tokens |
| Runner group | Existing `Default` group; all repositories except public repositories |
| Actions settings | Fork pull-request workflows require approval, including for outside collaborators |
