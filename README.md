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

Automated sync and self-healing are enabled for the root and Crossplane applications. Pruning is disabled, so deleting a manifest will not automatically delete its application or resources.

## LocalStack study environment

LocalStack is managed by [applications/localstack.yaml](./applications/localstack.yaml) through the root Argo CD application. It pins the official chart at `0.7.1` and the `localstack/localstack` image at `2026.10.0`; the chart's AWS CLI test image is pinned to `amazon/aws-cli:2.38.1`.

Current LocalStack releases require an account auth token. Before Argo CD starts the pod, create the `localstack` namespace and the `localstack-auth-token` Secret (key `auth-token`) outside Git by following [the LocalStack quickstart](./specs/001-localstack-integration/quickstart.md). Never put the token in a manifest, shell argument, log, or commit.

Check reconciliation and workload readiness with:

```sh
kubectl get applications.argoproj.io -n argocd
kubectl get pods,svc -n localstack
```

The endpoint is cluster-internal at `http://localstack.localstack.svc.cluster.local:4566`; use the quickstart's temporary `kubectl port-forward` procedure for workstation access. The quickstart also documents an AWS CLI smoke test and safe removal.

Persistence is disabled, so simulated resources are lost when the pod restarts. The root application has `prune: false`: removing `applications/localstack.yaml` from Git alone does not remove the LocalStack Argo CD application. Follow the quickstart's explicit cascade-delete and namespace cleanup steps when intentionally uninstalling it.

## Crossplane S3 buckets in LocalStack

The [AWS S3 provider manifest](./applications/crossplane-aws-s3.yaml) pins both Upbound packages to `v2.4.0`, compatible with the lab's Crossplane `v2.4.2`. The second-stage [LocalStack bucket manifest](./applications/localstack-s3-bucket.yaml) configures S3, S3 Control, and STS to use only the in-cluster LocalStack endpoint and declares a sample bucket. Do not publish that second-stage manifest until both providers are healthy and the dummy-credentials Secret has been created outside Git.

This provider release is for this study lab only: the Upbound Marketplace reports two High and two Low vulnerability findings for each package. Do not use this pin for real AWS resources; review the accepted risk and compatibility evidence in [the provider research](./specs/002-crossplane-s3-localstack/research.md) before publishing.

Follow the [S3 bucket quickstart](./specs/002-crossplane-s3-localstack/quickstart.md) for staged rollout, LocalStack-only smoke testing, troubleshooting, and safe cleanup. No real AWS account or credentials are used.

## Static sites in LocalStack

The namespaced `XStaticSite` API provisions an isolated S3 website bucket and its website configuration, public-access settings, and object-read policy through Crossplane. All provider operations use the existing LocalStack endpoint; anonymous reads are limited to that site's objects inside the lab. LocalStack remains a `ClusterIP` service, and website access requires the documented workstation port-forward. Do not use this lab-only public-read configuration with real AWS or expose LocalStack through a public route.

Each new site serves a minimal placeholder `index.html` right away, created by the Composition. Real static files are not managed by the declaration: upload them separately through the explicit LocalStack S3 endpoint, then verify the reported website URL while the port-forward is active. The bucket name and URL are published in the `XStaticSite` status.

Follow the [static-site quickstart](./specs/003-static-site-hosting/quickstart.md) for staged Function/API/site rollout, content upload, index and error-page checks, isolation testing, failure inspection, and deliberate cleanup. The root Argo CD application has `prune: false`; deleting a declaration from Git alone does not delete its XR or bucket. Deleting the XR also deletes its bucket contents.

Review the [static-site research](./specs/003-static-site-hosting/research.md) before rollout. The feature reuses the lab-only AWS provider `v2.4.0` pin and its accepted Marketplace findings; it is not intended for real AWS resources.
