# Kubernetes Platform

GitOps repository for infrastructure applications running on Kubernetes. Crossplane is the first managed application. Add future Argo CD `Application` manifests for tools such as Argo Rollouts, KEDA, and Turbonomic to `applications/`.

## Repository layout

```text
bootstrap/
  root-application.yaml
applications/
  crossplane.yaml
```

The root Argo CD `Application` watches `applications/`. It creates and manages each `Application` manifest found there.

## Install Crossplane with Argo CD

This procedure assumes Argo CD is already installed in the Minikube cluster and Crossplane is not yet installed.

### 1. Check the cluster context

Run commands from a terminal with `kubectl` configured for the intended cluster:

```sh
kubectl config current-context
kubectl get pods -n argocd
```

Confirm the context is `minikube` and the Argo CD pods are ready before continuing.

### 2. Confirm Argo CD can read this repository

The root application points to the public GitHub repository `https://github.com/guigsgbm/k8s-platform.git`, branch `main`. For a public repository, it is not necessary to add it manually under **Settings → Repositories**. Argo CD reads the source URL declared in the `Application`.

For a private repository, configure repository credentials in Argo CD before creating the root application. Do not store credentials in this Git repository.

### 3. Configure Crossplane-compatible resource tracking

Crossplane recommends annotation-based resource tracking in Argo CD. First inspect the current setting:

```sh
kubectl get configmap argocd-cm -n argocd \
  -o jsonpath='{.data.application\.resourceTrackingMethod}{"\n"}'
```

If it is not `annotation`, review the impact on other Argo CD applications because this is an Argo CD-wide setting. If appropriate for the cluster, set it with:

```sh
kubectl patch configmap argocd-cm -n argocd --type merge \
  -p '{"data":{"application.resourceTrackingMethod":"annotation"}}'
```

See the [Crossplane and Argo CD configuration guide](https://docs.crossplane.io/v2.4/guides/crossplane-with-argo-cd/).

### 4. Create the root application

From the root of a local clone of this repository, run:

```sh
kubectl apply -n argocd -f bootstrap/root-application.yaml
```

Argo CD will fetch the repository, observe `applications/`, and automatically create the `crossplane` application. That application installs the official Crossplane Helm chart version `2.4.2` in the `crossplane-system` namespace, creating the namespace if needed.

### 5. Verify the installation

Check the Argo CD applications and Crossplane workloads:

```sh
kubectl get applications.argoproj.io -n argocd
kubectl get deployments,pods -n crossplane-system
```

Wait for `k8s-platform` and `crossplane` to report `Synced` and `Healthy`, and for the Crossplane pods to become `Running` and `Ready`.

Argo CD renders the Helm chart to Kubernetes resources; it does not create a Helm release as `helm install` does. Verify the Argo CD application and Kubernetes workloads rather than expecting Crossplane to appear in `helm list`.

## Add another platform application

1. Add an Argo CD `Application` manifest under `applications/`.
2. Commit and push the change to the `main` branch.
3. Argo CD detects the new manifest and creates the application. Use **Refresh** in the UI to request an immediate check instead of waiting for the next repository poll.

Automated sync and self-healing are enabled for the root and Crossplane applications. Pruning is enabled, so deleting a manifest from Git deletes its application and, through the application's finalizer, its resources. Assess what a removal will delete before committing it.

## LocalStack study environment

LocalStack is managed by [applications/localstack.yaml](./applications/localstack.yaml) through the root Argo CD application. It pins the official chart at `0.7.1` and the `localstack/localstack` image at `2026.10.0`; the chart's AWS CLI test image is pinned to `amazon/aws-cli:2.38.1`.

Current LocalStack releases require an account auth token. Before Argo CD starts the pod, create the `localstack` namespace and the `localstack-auth-token` Secret (key `auth-token`) outside Git by following [the LocalStack quickstart](./specs/001-localstack-integration/quickstart.md). Never put the token in a manifest, shell argument, log, or commit.

Check reconciliation and workload readiness with:

```sh
kubectl get applications.argoproj.io -n argocd
kubectl get pods,svc -n localstack
```

The endpoint is cluster-internal at `http://localstack.localstack.svc.cluster.local:4566`; use the quickstart's temporary `kubectl port-forward` procedure for workstation access. The quickstart also documents an AWS CLI smoke test and safe removal.

Persistence is disabled, so simulated resources are lost when the pod restarts. The root application has `prune: true`: removing `applications/localstack.yaml` from Git removes the LocalStack Argo CD application. Follow the quickstart's cascade-delete and namespace cleanup steps when intentionally uninstalling it.

## Crossplane S3 buckets in LocalStack

The [AWS S3 provider manifest](./applications/crossplane-aws-s3.yaml) pins both Upbound packages to `v2.4.0`, compatible with the lab's Crossplane `v2.4.2`. The second-stage [LocalStack bucket manifest](./applications/localstack-s3-bucket.yaml) configures S3, S3 Control, and STS to use only the in-cluster LocalStack endpoint and declares a sample bucket. Do not publish that second-stage manifest until both providers are healthy and the dummy-credentials Secret has been created outside Git.

This provider release is for this study lab only: the Upbound Marketplace reports two High and two Low vulnerability findings for each package. Do not use this pin for real AWS resources; review the accepted risk and compatibility evidence in [the provider research](./specs/002-crossplane-s3-localstack/research.md) before publishing.

Follow the [S3 bucket quickstart](./specs/002-crossplane-s3-localstack/quickstart.md) for staged rollout, LocalStack-only smoke testing, troubleshooting, and safe cleanup. No real AWS account or credentials are used.

## Static sites in LocalStack

The namespaced `XStaticSite` API provisions an isolated S3 website bucket and its website configuration, public-access settings, and object-read policy through Crossplane. The bucket name is deterministically `site-<XStaticSite name>` and is reported in `status.bucketName`; `status.websiteURL` uses the matching LocalStack website address. All provider operations use the existing LocalStack endpoint; anonymous reads are limited to that site's objects inside the lab. LocalStack remains a `ClusterIP` service, and website access requires the documented workstation port-forward. Do not use this lab-only public-read configuration with real AWS or expose LocalStack through a public route.

Crossplane manages infrastructure only; it does not create a placeholder or manage website objects. A conforming site repository's GitHub Actions workflow publishes its static files on default-branch pushes or manual dispatch using the runner-provided LocalStack endpoint and credentials. `XStaticSite` readiness therefore does not indicate that website content has been uploaded.

Follow the [static-site quickstart](./specs/003-static-site-hosting/quickstart.md) for the API rollout and destructive bucket-identity change. The repository layout and publication contract are in [specs/006-site-repository-contract](./specs/006-site-repository-contract/). The root Argo CD application has `prune: true`; deleting a declaration from Git deletes its XR and, with it, the bucket contents.

Review the [static-site research](./specs/003-static-site-hosting/research.md) before rollout. The feature reuses the lab-only AWS provider `v2.4.0` pin and its accepted Marketplace findings; it is not intended for real AWS resources.

## Static-site discovery

The [discovery manifests](./applications/static-site-discovery.yaml) scan the GitHub organization `labs-guigsgbm` for non-archived repositories with the `backstage-static-site` topic. Repository names must match `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`; each discovered repository gets an Application named `site-<repo>` that syncs its default branch's `deploy/` directory into namespace `site-<repo>`. That directory should contain only `XStaticSite` manifests.

Discovery only scans the configured organization. Common reasons a site is not discovered or does not sync:

| Case | Where it appears |
|------|------------------|
| Wrong topic, another owner, or archived repository | No `site-<repo>` Application is created. |
| Repository name fails the rule above | No Application is created. |
| Missing `deploy/` directory | Application reports `ComparisonError: app path does not exist`. |
| Invalid manifest | Application reports an `OutOfSync` or sync failure with the validation error. |
| Disallowed kind or namespace | Application sync failure reports the AppProject denial or the own-namespace policy message. |

Repository access or GitHub API errors appear as an Application `ComparisonError` or an ApplicationSet `ErrorOccurred` condition. List discovered sites with:

```sh
kubectl -n argocd get applications.argoproj.io | grep '^site-'
```

Diagnose a site's sync condition and operation message with:

```sh
kubectl -n argocd get application site-<repo> \
  -o jsonpath='{.status.conditions}{"\n"}{.status.operationState.message}{"\n"}'
```

Each scan costs about `1 + N` GitHub API requests for up to 100 organization repositories, where `N` is the number of matching repositories. At the default three-minute interval:

| Access | GitHub limit | Approximate hourly use |
|--------|--------------|------------------------|
| Unauthenticated | 60 requests/hour | 20 scans/hour; one matching repo uses about 40 requests, two use about 60. |
| Authenticated | 5,000 requests/hour | 20 scans/hour at `1 + N` requests per scan. |

For more sites or private repositories, use a fine-grained, read-only token; create its Secret outside Git and reference only the Secret name and key in the ApplicationSet. Increasing `requeueAfterSeconds` also reduces request volume. See the [discovery quickstart](./specs/004-static-site-discovery/quickstart.md) for staged rollout, troubleshooting, and token setup.

### Removing a static site

Repository lifecycle events (untagging, archiving, deletion, or renaming) do not delete the Application, namespace, site, or bucket. To remove a site deliberately:

1. Untag or archive the repository so discovery cannot recreate its Application.
2. Delete `application site-<repo>` in `argocd`; its resources are retained because the generated Application has no deletion finalizer.
3. Delete the `XStaticSite` in `site-<repo>` and wait for its bucket to be removed. This irreversibly deletes the bucket contents.
4. Delete namespace `site-<repo>`.

Pruning remains enabled: removing an `XStaticSite` manifest from the site's `deploy/` directory deletes that resource and its bucket. Removing ordinary managed manifests from `applications/` prunes their resources too. The discovery ApplicationSet is an exception protected by `Prune=false`; removing its Git manifest does not delete the live ApplicationSet, so delete it explicitly when retiring discovery. Follow the [quickstart removal procedure](./specs/004-static-site-discovery/quickstart.md#deliberate-removal-of-a-site) before cleanup.

## GitHub Actions runners

The [runner Applications](./applications/arc-controller.yaml) and [runner pool](./applications/arc-runner-set.yaml) provide ephemeral in-cluster GitHub Actions runners that can reach LocalStack. The pool uses the organization's `Default` runner group, which allows all repositories except public repositories; the static-site topic does not restrict runner access. Anyone with write access to any non-public repository in `labs-guigsgbm` can run code on a runner, so workflows must follow the documented trigger guard. Create the GitHub token and LocalStack dummy-credential Secrets outside Git. The [static-site repository contract](./specs/006-site-repository-contract/contracts/site-repository.md) defines the site publication workflow.

Pruning is disabled for both runner Applications. Removing their manifests from Git does not remove the resources; follow the [runner quickstart](./specs/005-github-actions-runner/quickstart.md) for staged rollout, credential setup, troubleshooting, and deliberate removal.
