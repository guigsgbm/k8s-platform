# Kubernetes Platform

GitOps repository for infrastructure applications running on Kubernetes. Crossplane is the first managed application; add future Argo CD `Application` manifests for tools such as Argo Rollouts, KEDA, and Turbonomic to `applications/`.

## Repository layout

```text
bootstrap/
  root-application.yaml
applications/
  crossplane.yaml
```

The root Argo CD `Application` watches `applications/`. Each manifest in that directory defines an application to be managed by Argo CD.

## Before bootstrapping Argo CD

1. Create a remote Git repository named `k8s-platform` and push this repository to its `main` branch.
2. Replace the `repoURL` placeholder in `bootstrap/root-application.yaml` with the remote URL. Do not apply the bootstrap manifest while the placeholder remains.
3. If the Git repository is private, add its credentials to Argo CD using the UI or CLI; never commit credentials to this repository.
4. Crossplane recommends annotation-based resource tracking in Argo CD. Inspect the existing Argo CD configuration and applications before changing this cluster-wide setting. If appropriate, set it with:

   ```sh
   kubectl patch configmap argocd-cm -n argocd --type merge \
     -p '{"data":{"application.resourceTrackingMethod":"annotation"}}'
   ```

   See the [Crossplane and Argo CD configuration guide](https://docs.crossplane.io/v2.4/guides/crossplane-with-argo-cd/).

## Move the current Crossplane installation to Argo CD

The current Minikube installation is a fresh Crossplane 2.4.2 release with no Crossplane resources that need to be preserved. Uninstalling it will temporarily stop the Crossplane controllers.

1. Complete the remote-repository and Argo CD prerequisites above first. This avoids leaving Crossplane unavailable while the Git source is being configured.
2. Remove the manually installed release:

   ```sh
   helm uninstall crossplane --namespace crossplane-system
   helm list --namespace crossplane-system
   ```

   Do not manually delete Crossplane CRDs. If this cluster now has Crossplane resources that must be retained, stop and reassess the migration before uninstalling.
3. Immediately create the root application:

   ```sh
   kubectl apply -n argocd -f bootstrap/root-application.yaml
   ```
4. Wait for both `k8s-platform` and `crossplane` to become Synced/Healthy in Argo CD. Verify the workloads:

   ```sh
   kubectl get deployments,pods -n crossplane-system
   ```

Argo CD uses Helm to render the chart; it does not create a Helm release in the same way as `helm install`. Therefore, `helm list -A` may not show Crossplane after Argo CD takes over. Use the Argo CD application status and Kubernetes workloads to verify it.

Both applications enable automated sync and self-healing but leave pruning disabled, so removing a manifest will not automatically uninstall its application.
