# Quickstart: Validate a Static-Site Repository

This guide covers the single manual end-to-end run in [spec.md](./spec.md). The repository layout and interface are defined in [contracts/site-repository.md](./contracts/site-repository.md); entity details are in [data-model.md](./data-model.md).

## Prerequisites

- Specs 001–005 are deployed: Argo CD discovery, the `XStaticSite` API, LocalStack, and the `lab-localstack` runner are healthy.
- The GitHub organization `labs-guigsgbm` and permission to create a private repository and set its topic are available.
- The LocalStack dummy credentials are configured in the runner environment; no repository secret or real AWS credential is needed.
- `kubectl` and `curl` are available on the workstation. The workstation reaches the website through the LocalStack port-forward.

## Roll out the bucket-name change

The existing `applications/docs-demo.yaml` declaration uses the prior UID-derived bucket. The approved lab rollout treats that bucket and its contents as disposable; they will not be migrated. The `docs-demo` site will instead be provisioned from its discovered repository, so do not restore the platform repository declaration.

1. Remove `applications/docs-demo.yaml` from Git. The root Argo CD Application has `prune: false`, so explicitly delete the existing XR and wait for Crossplane to remove its old bucket:

   ```sh
   kubectl delete xstaticsite docs-demo -n static-sites --wait=true --timeout=10m
   ```
2. Before publishing the Composition change, run a server-side dry-run:

   ```sh
   kubectl apply --dry-run=server -f applications/crossplane-static-site-api.yaml
   ```

3. Publish the Composition update.

This is a destructive lab reset, not a content-preserving migration.

## Run the end-to-end scenario once

1. Create or update a private, non-archived repository in `labs-guigsgbm` whose name matches the contract. Add the `backstage-static-site` topic.
2. On the default branch, add the paths and values from [contracts/site-repository.md](./contracts/site-repository.md), including the matching files under `src/` and a `src/site.css` asset linked from the index. Do not add a build system or credentials for the initial plain-static site.
3. Push the default branch. Wait for discovery to create the site's Argo CD Application and for Crossplane to report the `XStaticSite` ready:

   ```sh
   REPOSITORY_NAME=your-site-name
   kubectl -n argocd get applications.argoproj.io
   kubectl get xstaticsite "$REPOSITORY_NAME" -n "site-$REPOSITORY_NAME" \
     -o jsonpath='{.status.bucketName}{"\n"}{.status.websiteURL}{"\n"}'
   ```

   Confirm `status.bucketName` is `site-$REPOSITORY_NAME` and `status.websiteURL` has the LocalStack format in the contract.
4. Let the default-branch push run the workflow, or dispatch it manually. Confirm it installs the pinned AWS CLI and Node.js, waits for the bucket, syncs the configured directory, and prints the matching website URL. The workflow must complete without Kubernetes access or repository secrets.
5. Start a workstation port-forward and request the printed URL:

   ```sh
   kubectl port-forward svc/localstack 4566:4566 -n localstack
   ```

   In another terminal, set `REPOSITORY_NAME` to the created repository name and run:

   ```sh
   SITE_URL="http://site-$REPOSITORY_NAME.s3-website.localhost.localstack.cloud:4566/"
   curl --fail "$SITE_URL"
   curl --fail "${SITE_URL}site.css"
   ```

   Confirm the configured index document and its linked static asset are served.
