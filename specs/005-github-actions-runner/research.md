# Research: In-Cluster GitHub Actions Runners

Spec: [spec.md](./spec.md). All items were checked against the real charts (`helm pull`/`helm template`) and the live cluster unless marked otherwise.

## 1. Runner model and versions (FR-001, FR-012)

- **Decision**: GitHub's official runner scale set model: controller chart `gha-runner-scale-set-controller` and pool chart `gha-runner-scale-set`, both pinned to `0.15.1`, pulled by Argo CD from the OCI repository `ghcr.io/actions/actions-runner-controller-charts` (`repoURL` without the `oci://` prefix). Runner image pinned to `ghcr.io/actions/actions-runner:2.338.2`.
- **Rationale**: It is the GitHub-supported model, it has no inbound connectivity requirement (the listener long-polls GitHub), it creates one pod per job and it scales to zero. The chart default image tag is `latest`, so the runner image tag must be set explicitly in the pod template; the controller image tag defaults to the chart's app version.
- **Alternatives considered**: The legacy community `RunnerDeployment` model (not the official one, webhook or polling driven, not ephemeral by default). A permanent runner pod (violates the ephemeral requirement, wastes memory). GitHub-hosted runners (cannot reach the ClusterIP LocalStack).
- **To verify during implementation**: Argo CD `v3.5.5` pulling the public OCI chart without a registered repository.

## 2. Ordered rollout (FR-012)

- **Decision**: Two Applications. `arc-controller` (default wave 0) installs the controller and CRDs; `arc-runner-set` carries `argocd.argoproj.io/sync-wave: "1"` and `syncPolicy.retry` with backoff. Both set `ServerSideApply=true`, and the controller Application also sets `CreateNamespace=true`.
- **Rationale**: The `AutoscalingRunnerSet` kind does not exist until the controller chart's CRDs are applied. The wave gives the order under the root Application; the retry makes the pool Application recover on its own if the wave is not honored (Application health ordering under app-of-apps depends on Argo CD's Application health check, so it is a safety net and not the only mechanism).
- **Server-side apply is mandatory**: a server-side dry run of the four CRDs succeeds; a client-side dry run of `autoscalingrunnersets` (1.26 MB) fails with `metadata.annotations: Too long: may not be more than 262144 bytes`.
- **Alternatives considered**: Putting the pool manifests in the same Application (CRD race), a hook Job to wait for CRDs (extra moving part).

## 3. Controller RBAC (FR-009)

- **Decision**: Set `flags.watchSingleNamespace: arc-runners`, `flags.logLevel: info`, and modest requests and limits.
- **Rationale**: Default rendering produces a `ClusterRole` and `ClusterRoleBinding`; with `watchSingleNamespace` the chart renders only namespaced `Role` and `RoleBinding` objects (controller, single-namespace watch and listener roles). The controller then cannot touch any other namespace. The default log level is `debug`, which is noisy for a lab.
- **Alternatives considered**: Cluster-wide watching (broader than needed).
- **Constraint**: The runner pool must therefore live in `arc-runners`; the namespace name is fixed in both manifests.

## 4. Runner permissions (FR-009)

- **Decision**: Runners get no Kubernetes permissions. The chart's own no-permission ServiceAccount is used (it is created when container mode is not `kubernetes`), and the pod template sets `automountServiceAccountToken: false`. Container mode stays the default (a runner process in one container); Docker-in-Docker and Kubernetes mode are not used because the latter requires pod and job permissions.
- **Waiting for an `XStaticSite` to be Ready**: not through the Kubernetes API. A workflow checks over S3 with the dummy credentials, for example that the site's bucket exists. If reading `XStaticSite` status is ever required, a later spec must add a namespaced, read-only Role for it; this feature adds none.
- **Rationale**: Verified live that a pod with `automountServiceAccountToken: false` has no `/var/run/secrets/kubernetes.io`.

## 5. Authentication and Secrets (FR-003, FR-010, FR-015)

- **Decision** (user-confirmed): a **fine-grained personal access token** with resource owner `labs-guigsgbm`, Organization permissions **Self-hosted runners: Read and write** and **Administration: Read** (the permissions the GitHub documentation lists for organization runners), and a short expiry. The Secret `arc-github-token` in `arc-runners` has the single key `github_token` and is referenced by name (`githubConfigSecret: arc-github-token`). It is created from an interactive prompt (`read -s`) so the value never enters shell history. A classic token needs `admin:org`, which is broader and is not used.
- **Trade-offs accepted**: the token is tied to the user who created it (it stops working if that user loses organization ownership) and it expires, after which the pool stops registering until a new token replaces it; the quickstart documents the symptom and the renewal. A GitHub App (organization-owned, Self-hosted runners read/write and Metadata read-only, no expiry, no user dependency) remains the preferred option and only changes the Secret contents, so it can replace the token later without touching the manifests except the Secret name.
- **Dummy AWS credentials**: Secret `lab-aws-credentials` in `arc-runners` with keys `access-key-id` and `secret-access-key`, injected as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` through `secretKeyRef` in the runner template. `AWS_ENDPOINT_URL` and `AWS_DEFAULT_REGION` are not secret and are plain environment values in the manifest. The value `test`/`test` is accepted by LocalStack; the lab's existing dummy values are used if different.
- **Namespace ordering for Secrets**: the Secrets must exist in `arc-runners` before the pool syncs, but Argo CD creates that namespace. The quickstart creates the namespace first with `kubectl`; `CreateNamespace=true` then adopts it without error.
- **Alternatives considered**: Classic PAT (too broad), storing Secrets in Git even encrypted (violates Principle IV and adds a tool), External Secrets (no secret store exists in the lab).

## 6. Runner group and access control (FR-002, FR-007, FR-008)

- **Decision**: The pool registers into the organization's existing `Default` runner group, configured for all repositories except public repositories. The pool name `lab-localstack` is the label a workflow requests in `runs-on`; no dedicated group or repository allow-list is created.
- **Rationale**: This follows the operator's choice to reuse the ready `Default` group. Consequently, every non-public repository in the organization can use the pool; the static-site topic used by spec 004 is not an access-control boundary. Public repositories and repositories outside the organization cannot use the pool.
- **Trigger policy** (layers): (1) the `Default` group excludes public repositories; (2) organization settings require approval for workflows from outside collaborators/forks; (3) a job-level `if` guard in each workflow using the label allows only `github.event_name` of `push` to the repository's default branch or `workflow_dispatch`; (4) pull-request triggers are never combined with this label. The reference snippet is in [contracts/runner-pool.md](./contracts/runner-pool.md). The group and organization settings are enforcement boundaries; the workflow guard protects against mistakes in workflow configuration.
- **Residual risk (documented)**: anyone with write access to any non-public repository in the organization can run code on the runner. This is broader than a static-site-only allow-list and is accepted by the operator. Ephemerality, no Kubernetes permission, dummy credentials only, exclusion of public repositories, and the small LocalStack blast radius mitigate the risk.

## 7. Pod hardening and resources (FR-004, FR-005)

- **Decision**: Pod security context `runAsNonRoot: true`, `runAsUser: 1001`, `seccompProfile: RuntimeDefault`; container security context `allowPrivilegeEscalation: false`, capabilities dropped `ALL`; requests `250m` CPU and `512Mi` memory; memory limit `2Gi`, no CPU limit. `minRunners: 0`, `maxRunners: 2`. Controller requests `50m`/`64Mi`, memory limit `256Mi`.
- **Rationale**: The image already runs as uid 1001; with escalation disabled `sudo` cannot work (verified: "no new privileges" blocks it), so per-job tooling must install into the user's home. Node builds can need more than 512 MiB, hence the 2 GiB limit. A CPU limit would only slow builds; requests protect the neighbors. The node has 8 CPU and about 16 GiB, with LocalStack limited to 2 GiB.
- **Network**: the cluster's CNI is not assumed to enforce NetworkPolicy, and the spec only requires that runners can reach GitHub and LocalStack, so no NetworkPolicy is added (Principle VII). This is recorded as a known limitation.

## 8. Tools in the runner (FR-011)

- **Decision**: No custom image. The upstream image provides git, curl, unzip and jq. Workflows install the rest per job, pinned: the AWS CLI by downloading the versioned zip and installing it under the user's home (`aws-cli/2.38.1`, the same version LocalStack's chart uses for its tests), and Node.js with `actions/setup-node` pinned to a commit SHA and a pinned Node version. The quickstart records the install snippet; the real workflow is a later spec.
- **Verified live**: the AWS CLI install, `s3 mb`, `cp` upload, `cp` read-back and `rb` all worked from the hardened pod against the in-cluster endpoint.
- **Alternatives considered**: A custom image (needs a build, a registry or `minikube image load` and a rebuild process for every version bump, adding a pipeline this lab does not have).

## 9. LocalStack and DNS (FR-006)

- **Decision**: Use `http://localstack.localstack.svc.cluster.local:4566` as `AWS_ENDPOINT_URL` (supported by AWS CLI v2). Region `us-east-1`.
- **Behavior**: LocalStack has no persistence, so a restart empties it (it already restarted 3 times in this cluster); jobs must create what they need. LocalStack being down surfaces as a failed job step, not as a runner failure, because the runner does not contact it at startup.

## 10. Failure modes and recovery (FR-013, FR-014)

- **Visibility**: `kubectl get autoscalingrunnerset,ephemeralrunnerset,ephemeralrunner -n arc-runners`, the listener pod in `arc-systems`, the controller logs, the Argo CD Application health, and the pool page in the GitHub organization (runner groups and runners).
- **Missing or invalid credential**: the pool does not register and the controller logs the GitHub API error; recreating the Secret recovers it, and the controller reads the Secret on reconcile (restart the controller if it does not pick it up; the quickstart states the command).
- **Minikube restart**: custom resources are kept in the cluster, the Deployment restarts the controller, the listener is recreated, and ephemeral runners that cannot restart are replaced by the controller, which also removes their registrations from GitHub through the chart's finalizers. Stale offline entries cannot take jobs; the quickstart explains how to confirm and how to remove any leftovers from the organization page. To confirm during implementation.
- **Deleting out of order**: the chart's resources hold `actions.github.com/cleanup-protection` finalizers that only the controller removes. Removal must delete the pool before the controller; the quickstart gives that order and the repair step for a stuck finalizer.
- **Queued jobs**: GitHub cancels jobs that stay queued for 24 hours. A wrong label never matches the pool and the job just waits.

## 11. Prune protection (FR-016)

- **Decision**: Each Application manifest has `argocd.argoproj.io/sync-options: Prune=false` (stops the root, which now has `prune: true`, from deleting the Application when the file is removed from Git), `syncPolicy.automated.prune: false`, and no `resources-finalizer.argocd.argoproj.io`, so even deleting the Application object by hand leaves the workloads. A comment in each manifest records the reason.
- **Rationale**: Same pattern as the discovery ApplicationSet of spec 004.
