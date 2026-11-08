# Contract: Static-Site Repository

This contract defines the repository consumed by the discovery ApplicationSet, Crossplane `XStaticSite` API, Backstage catalog, and `lab-localstack` workflow runner. It does not create repositories or a Backstage template.

## Repository eligibility

| Property | Required value |
|----------|----------------|
| Owner | `labs-guigsgbm` |
| Visibility / state | Private and not archived |
| Topic | `backstage-static-site` |
| Name | `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$` (maximum 58 characters) |
| Branch | Required files are committed to the default branch |

The name rule ensures `site-<repository-name>` is a valid S3 bucket name of at most 63 characters.

## Required repository layout

```text
.
├── .github/
│   └── workflows/
│       └── deploy.yml
├── catalog-info.yaml
├── deploy/
│   └── xstaticsite.yaml
├── src/
│   ├── index.html
│   └── error.html
└── README.md
```

`src/` contains publish-ready static files. Plain HTML needs no build step. If a build is added, it writes to `dist/` and the workflow publishes `dist/`; it must not silently continue to publish `src/`.

The three template-time placeholders are `__REPOSITORY_NAME__`, `__INDEX_DOCUMENT__`, and `__ERROR_DOCUMENT__`. Replace them with concrete values before committing deployable manifests. Initially the document values are `index.html` and `error.html`. The files under `src/` must use the configured document names.

## Deployment declaration: `deploy/xstaticsite.yaml`

The file contains exactly one `XStaticSite` and no `metadata.namespace`:

```yaml
apiVersion: platform.k8s-platform.io/v1alpha1
kind: XStaticSite
metadata:
  name: __REPOSITORY_NAME__
spec:
  indexDocument: __INDEX_DOCUMENT__
  errorDocument: __ERROR_DOCUMENT__
```

Argo CD discovery deploys this declaration to namespace `site-<repository-name>`. The resource name is the unprefixed GitHub repository name.

The platform contract is:

- The bucket name and `status.bucketName` are `site-<repository-name>`.
- `status.websiteURL` is `http://site-<repository-name>.s3-website.localhost.localstack.cloud:4566/`.
- `Ready` represents provisioned infrastructure, not uploaded website content.
- Crossplane does not manage a placeholder or any other website object. The workflow owns the objects it publishes.

## Catalog descriptor: `catalog-info.yaml`

It declares a Backstage `Component` of type `website` and identifies the GitHub project:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: __REPOSITORY_NAME__
  annotations:
    github.com/project-slug: labs-guigsgbm/__REPOSITORY_NAME__
spec:
  type: website
```

## Publication workflow: `.github/workflows/deploy.yml`

- Triggers are pushes and `workflow_dispatch`. The deploy job condition allows a push only when `github.ref` matches the default branch in the event payload. This job-level guard is evaluated before runner allocation. Manual dispatch is allowed from the selected branch and publishes that checked-out revision.
- The deploy job uses `runs-on: lab-localstack`. It checks out the selected revision, installs AWS CLI `2.38.1` from its versioned Linux x86_64 archive using a user-local install (no `sudo`), and installs Node.js `24.11.1` using `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1` (`v7.1.0`). Pin GitHub Actions to immutable full commit SHAs, not tags or version aliases.
- The workflow derives the unprefixed repository name from GitHub Actions context and constructs `site-<repository-name>`. It must not duplicate the repository name as a literal or repository-specific workflow input.
- Before syncing, it uses AWS CLI commands only to wait at most 10 minutes for the matching bucket. A timeout is an explicit failure; the workflow must not guess another bucket or fall back to AWS.
- It runs `aws s3 sync` for `src/` (or `dist/` if the repository adds a build) through the runner-provided `AWS_ENDPOINT_URL`, then prints the matching website URL.
- The runner supplies `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` as dummy LocalStack environment values. The workflow does not print or override them, read repository secrets, use `kubectl`, or access the Kubernetes API.

## README

`README.md` briefly documents the required layout, discovery topic, index/error settings, push and manual publication triggers, reported LocalStack URL, and that credentials come from the runner and are not stored in the repository.
