# Feature Specification: In-Cluster GitHub Actions Runners

**Feature Branch**: `main` (specification created without switching branches)

**Created**: 2026-11-02

**Status**: Draft

**Input**: User description: "GitHub Actions self-hosted runner inside the Minikube cluster for static-site deployments. Ephemeral, GitOps-managed runners registered at organization level, able to reach the in-cluster LocalStack S3 endpoint (a ClusterIP service not reachable from GitHub-hosted runners), usable by any static-site repository of the lab organization, restricted to trusted triggers, with no credential in Git. The real site workflow, including any end-to-end proof of uploading content, belongs to a later spec; this spec delivers no workflow."

## Clarifications

### Session 2026-11-02

- Q: Should this spec deliver a smoke-test workflow, and in which repository? → A: No smoke-test workflow is delivered; the runners are proven only through the later site-deployment spec. Acceptance here is limited to what is observable without a delivered workflow (registration, idle state, access control, resources, cleanup).
- Q: Which GitHub credential should the lab use? → A: A fine-grained personal access token (resource owner `labs-guigsgbm`) with the minimum organization permissions, accepted as the "App impractical" case of FR-003; its expiry and dependency on its owner are documented trade-offs, and a GitHub App can replace it later.
- Q: Which runner group should the pool use? → A: Use the existing `Default` group as configured to allow all organization repositories except public repositories. This intentionally allows every non-public repository in the organization to use the pool, not only repositories with the static-site topic.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Install runners through GitOps and see them register (Priority: P1)

As a platform engineer, I install the runner infrastructure through the existing root Argo CD application and see it register with GitHub and become idle/ready, with no manual runner registration.

**Why this priority**: Without a registered runner pool nothing else in this feature can be exercised.

**Independent Test**: With the prerequisites in place (organization, GitHub credential, Secrets), add the manifests to `applications/`, let the root application sync, and verify the runner pool appears in the GitHub organization as online with zero running jobs, without any command run against GitHub.

**Acceptance Scenarios**:

1. **Given** the prerequisites and Secrets exist, **When** the manifests are merged and synced, **Then** the controller and the runner pool become Healthy in the documented order and the pool is listed in the GitHub organization as registered and idle.
2. **Given** the pool is idle, **When** nobody runs a job, **Then** it uses the configured minimum number of runners (default zero) and consumes no runner resources beyond the controller.
3. **Given** the GitHub credential Secret is missing at install time, **When** the pool is synced, **Then** the pool reports a clear not-registered state, nothing crashes in a loop that affects other platform components, and creating the Secret later recovers it without editing Git.

---

### User Story 2 - A site workflow gets a clean ephemeral runner that reaches LocalStack (Priority: P1)

As a site repository workflow, I request the agreed runner label and get a clean, ephemeral runner (one job per runner, destroyed afterwards) that can reach the in-cluster LocalStack S3 endpoint.

**Why this priority**: This is the capability the feature exists to provide; it is what the later site-deployment spec builds on.

**Independent Test**: Run any throwaway job (not delivered by this spec) from a non-public repository in the lab organization with the agreed label that only performs an S3 listing against the in-cluster endpoint, then verify the job ran, no state remained, and the runner is gone. The delivered proof for this story is the later site-deployment spec.

**Acceptance Scenarios**:

1. **Given** a non-public repository in the lab organization and an idle pool, **When** a job requests the agreed label, **Then** a runner is created on demand, runs exactly that one job and is removed afterwards.
2. **Given** two jobs queued at the same time, **When** they run, **Then** each runs on its own fresh runner and neither sees files, environment or credentials left by the other.
3. **Given** a running job, **When** it calls the in-cluster LocalStack S3 endpoint with the lab's dummy credentials, **Then** the call succeeds without any cluster credentials or port-forward.
4. **Given** a site's `XStaticSite` has been deployed, **When** a workflow needs to know the site is ready, **Then** it can verify readiness using only the S3 endpoint and the dummy credentials (for example, that the site's bucket exists), without any Kubernetes access (FR-009).

---

### User Story 3 - See runner state and diagnose failures (Priority: P2)

As a platform engineer, I can see runner state (idle, running, failed to register) and diagnose why jobs are queued or failing.

**Why this priority**: A runner pool that fails silently is indistinguishable from "GitHub is slow"; diagnosis cuts the cost of every later incident.

**Independent Test**: Break each of the documented causes in turn (revoked credential, unreachable GitHub, unknown label, exhausted capacity) and verify the operator identifies each cause in the documented time using the quickstart.

**Acceptance Scenarios**:

1. **Given** a running platform, **When** the operator follows the quickstart, **Then** they can tell for the pool whether it is registered, how many runners are idle or running, and the most recent failure reason.
2. **Given** a job is queued, **When** the operator investigates, **Then** the quickstart distinguishes: no runner provides the requested label, public repository or repository outside the organization, maximum concurrency reached, credential invalid, GitHub unreachable.
3. **Given** a runner fails to register, **When** the operator checks, **Then** the failure and its cause are visible without reading secret values.

---

### User Story 4 - Scale to zero and remove the feature safely (Priority: P2)

As a platform engineer, I can scale runners down to zero when idle and remove the whole feature safely.

**Why this priority**: Minikube has limited resources and the lab must be able to reclaim them and clean up GitHub registrations deliberately.

**Independent Test**: Set the pool to its zero-runner state and verify no runner pods exist while the pool stays registered; then run the documented removal and verify the cluster and the GitHub organization hold no leftover runner, pool or running job.

**Acceptance Scenarios**:

1. **Given** the pool is configured with a minimum of zero, **When** no job is queued, **Then** no runner pod exists and a newly queued job still starts a runner.
2. **Given** the feature is removed from Git, **When** the root application syncs, **Then** nothing is deleted automatically (FR-016) and the documented removal procedure is the only way runners, registrations and Secrets are removed.
3. **Given** the documented removal procedure has been run, **When** the operator checks the cluster and the organization, **Then** no runner pod, pool registration, Secret or namespace of this feature remains, and LocalStack and the sites are unaffected.

---

### Edge Cases

- GitHub unreachable: running jobs may fail; no new runners register; the state is visible to the operator and recovers by itself when GitHub returns (US3).
- GitHub App credential or token expired, revoked or missing at install time: the pool reports not-registered with a visible cause; no credential value appears in logs; recovery needs only a new out-of-Git Secret.
- Runner pod evicted or OOM-killed mid-job: the job is reported as failed or cancelled by GitHub, the runner is not reused, and a re-run on a fresh runner succeeds; no stale runner remains registered.
- Node out of resources with many queued jobs: concurrency never exceeds the configured maximum; excess jobs wait in the queue and run as capacity frees; LocalStack, Argo CD and Crossplane are not starved by runners (requests and limits, FR-005).
- Job requests a label no runner provides: the job stays queued in GitHub and the quickstart explains how to recognize and fix it; it is never picked up by an unrelated runner.
- Jobs stuck in queue: the documentation states how long GitHub keeps queued jobs before cancelling them and how to find the cause.
- LocalStack restarts (non-persistent) during a job: the job fails at the next S3 call; buckets created by the job or by sites are gone after the restart (a known lab limitation from spec 001); workflows must not depend on buckets existing before the job.
- LocalStack unavailable when the runner starts: the runner starts normally; the failure surfaces in the job, not as a runner crash loop.
- Controller or pool deleted while jobs are running: running jobs fail visibly; Argo CD pruning of this feature is disabled (FR-016), so a Git removal does not terminate jobs or deregister the pool.
- Workflow from a public repository or a repository outside the lab organization requests the runner group: the job is never scheduled on a runner of this feature (FR-007, FR-008).
- Minikube restart: the controller, the pool and its registration recover without manual cleanup of stale runners in GitHub; leftover offline entries are removed automatically or are harmless and documented (FR-013).
- Repository from outside the lab organization with a matching label: it can never use these runners.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Runners MUST be registered at organization level in a single GitHub organization, configurable in one place, using the official GitHub-supported controller and runner-scale-set model. Chart and image versions MUST be pinned, following the pinned-version conventions of specs 001–003; no `latest` or floating tag.
- **FR-002**: The runner pool MUST be exposed under one documented runner label that site workflows request. A job that does not request this label MUST NOT run on these runners.
- **FR-003**: Authentication to GitHub MUST use a GitHub App (preferred) or, only if the App is impractical, a fine-grained token, in both cases with the minimum permissions required for organization-level runner registration. The credential MUST be created outside Git and delivered to the cluster as a Secret in the runner namespace. The quickstart MUST list the exact permissions and the procedure to create, rotate and revoke it.
- **FR-004**: Runners MUST be ephemeral: one job per runner, with the runner removed afterwards and nothing carried over to the next job. They MUST run as a non-root user in a dedicated namespace, separate from `localstack`, Argo CD and the site namespaces.
- **FR-005**: Each runner MUST have resource requests and limits suited to a small Minikube node, and the pool MUST have a configurable minimum (default zero) and maximum number of concurrent runners. The default maximum MUST leave enough capacity for LocalStack, Crossplane and Argo CD to remain healthy while the maximum number of runners is busy.
- **FR-006**: Runners MUST be able to reach GitHub and the in-cluster LocalStack S3 endpoint, and nothing else MUST be required. LocalStack MUST remain a ClusterIP service and MUST NOT be exposed outside the cluster for this feature.
- **FR-007**: The pool MUST use the organization's `Default` runner group, configured to allow all repositories in the lab organization except public repositories. Therefore, any non-public repository in the organization can use these runners; the static-site topic is not an access-control boundary. Public repositories and repositories outside the lab organization MUST NOT be able to use these runners.
- **FR-008**: The security posture for repositories using the `Default` group MUST be explicit. Only these triggers are allowed on these runners: push to the default branch and manual dispatch. Pull-request triggers (including from forks) and any trigger run by an untrusted contributor are NOT allowed. The documentation MUST state how this is enforced in layers: the `Default` group excludes public repositories, organization settings require approval for fork and outside-collaborator workflows, and each workflow using the label has a trigger/branch guard. The documentation MUST include a reference trigger/branch guard snippet; no workflow file is delivered by this feature.
- **FR-009**: Runners and their workflows MUST NOT receive cluster-admin or any broad Kubernetes permission. The minimum is **no Kubernetes API permission for the runner at all** (no service account token mounted, no Role or ClusterRole bound). Only the controller holds the permissions its upstream chart defines, scoped as the chart allows. A workflow that must wait for a site to be Ready MUST do so without cluster access, by checking observable results over the S3 endpoint with the dummy credentials (for example that the site's bucket exists and serves the expected object). If a future need requires reading the `XStaticSite` status, it MUST be provided as a read-only, namespace-scoped permission defined in a later spec, never write access.
- **FR-010**: The credentials a workflow uses against LocalStack MUST be the lab's dummy credentials, supplied to runners as a Secret created outside Git and never as repository content, workflow files or logs. No real AWS account, credential or resource MAY be used.
- **FR-011**: The runner image MUST provide the AWS CLI, curl, git and a Node.js toolchain for building frontends with pinned versions, or the documentation MUST state precisely how each tool is installed per job. The choice MUST be recorded during planning.
- **FR-012**: The feature MUST be delivered as manifests under `applications/` managed by the existing root Argo CD application, with automated sync and self-heal. The controller and the runner pool MUST be separate, ordered rollout stages because the pool depends on the controller's CRDs, and the order MUST be documented and verified in the quickstart.
- **FR-013**: After a Minikube restart, the controller and the pool MUST recover without manual intervention and without manually removing stale runners in GitHub.
- **FR-014**: The operator MUST be able to see the pool's registration, idle and running runner counts and the latest failure reason, and MUST be able to distinguish the queue causes listed in US3 using the quickstart.
- **FR-015**: No credential, token, private key or dummy-credential value MUST be stored in this repository, manifests, logs, workflow files or shell history. The quickstart MUST document the Secret creation procedure in a way that avoids shell history (for example reading from a file or a prompt).
- **FR-016**: The runner Applications MUST have automated pruning disabled, with the reason documented next to the configuration: deletion could terminate running jobs, orphan registrations in GitHub, or delete CRDs and Secrets still in use. This is an explicit exception to the constitution's default of pruning. Removal MUST be done only by the documented deliberate procedure, which covers running jobs, the GitHub registration, Secrets and the namespace.
- **FR-017**: The quickstart MUST cover prerequisites (organization, GitHub credential creation and permissions, Secrets), staged rollout, verification of registration and idle state, troubleshooting and deliberate cleanup, like the previous specs.
- **FR-018**: This feature MUST NOT change the `XStaticSite` API, Argo CD discovery, LocalStack, or Crossplane configuration, and MUST NOT require a real AWS resource.

### Key Entities *(include if feature involves data)*

- **Lab organization**: The single GitHub organization where runners are registered; configurable in one place.
- **Runner controller**: The cluster component that watches the pool and creates and deletes runners on demand.
- **Runner pool (scale set)**: The named group of ephemeral runners with a label, minimum and maximum size, registered to the organization.
- **Runner group**: The organization's `Default` group, which permits all non-public organization repositories and excludes public repositories.
- **Runner**: A single-use pod that executes exactly one job and is then destroyed.
- **GitHub credential**: The GitHub App (or fine-grained token) used to register runners, kept outside Git as a Secret.
- **Lab AWS credentials**: The dummy access key and secret used against LocalStack, kept outside Git as a Secret.
- **Site repository**: A repository of the lab organization carrying the static-site topic; private site repositories can use the runner, while the topic itself does not control runner access.

### Out of Scope

- The site repository's real workflow, build, content upload logic and file layout (later spec).
- Creating repositories or Backstage templates.
- Runners for anything other than the lab organization.
- Real AWS resources.
- Any smoke-test or other workflow file, and end-to-end proof of uploading content (later site-deployment spec).
- Changes to the discovery, `XStaticSite` API or LocalStack setup of specs 001–004.
- Restricting runner access to the static-site topic or automatically synchronizing group access with repository topics.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After the prerequisites exist and the manifests are merged, the runner pool is shown as registered in the GitHub organization and idle within 10 minutes with zero manual runner-registration steps.
- **SC-002**: A one-off job from a non-public repository in the lab organization requesting the agreed label starts on a new in-cluster runner within 2 minutes from an idle pool, can reach the in-cluster S3 endpoint, and the runner pod no longer exists within 2 minutes of the job ending.
- **SC-003**: Over 10 consecutive one-off jobs, 100% run on a new runner and none sees files or state from a previous job.
- **SC-004**: In testing, 100% of attempts from a public repository, a repository outside the lab organization, a fork, or a pull-request trigger are not executed on these runners.
- **SC-005**: A search of this repository, the cluster manifests, the workflow files, the job logs and the documented commands finds no GitHub credential and no AWS credential value, and the Secret creation procedure is documented.
- **SC-006**: With the maximum number of runners busy, LocalStack, Argo CD and Crossplane remain Healthy and no runner is OOM-killed by contention with them in normal use.
- **SC-007**: Using only the quickstart, an operator identifies the cause of each of the five queue/failure causes (US3) in under 10 minutes each.
- **SC-008**: After a Minikube restart, the pool is registered and idle again within 10 minutes with no manual cleanup, and a one-off job requesting the label runs afterwards.
- **SC-009**: After the documented removal procedure, no runner pod, pool registration, Secret or namespace of this feature remains, and sites and LocalStack still pass their previous checks.

## Assumptions

- Specs 001–004 are in place: Argo CD, Crossplane and LocalStack run on Minikube, LocalStack is a non-persistent ClusterIP service, and sites are discovered by topic.
- The lab organization is the same owner configured for spec 004, and the operator has permission to create the GitHub credential and to manage runner groups there.
- The existing `Default` runner group allows all non-public repositories in the organization and excludes public repositories. This is broader than the static-site topic: the topic controls discovery only, not runner access. Organization approval settings and workflow guards restrict untrusted triggers, but any user who can push to a non-public organization repository can run code on the pool.
- The runner label is a single documented default such as `lab-localstack`; planning confirms the final name.
- The default pool is a minimum of zero and a small maximum (planning chooses, probably 2) that fits the node; both are one-line configuration changes.
- The lab's dummy AWS credentials are the same as those already accepted by LocalStack; no new credential system is introduced.
- Readiness of a site for upload is established over S3 (bucket exists) instead of the Kubernetes API, since runners have no cluster access.
- Public repositories are excluded by the `Default` group; organization approval settings and workflow guards protect non-public repositories from untrusted triggers.
- Verification of runner behavior before the site-deployment spec uses a throwaway job in a non-public repository in the organization that the operator writes ad hoc and does not commit to this repository.
- The Minikube node has outbound access to GitHub, and its DNS resolves the in-cluster LocalStack service name from the runner namespace.
