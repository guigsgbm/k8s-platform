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
