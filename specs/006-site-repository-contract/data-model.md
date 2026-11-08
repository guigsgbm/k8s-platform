# Data Model: Static-Site Repository Contract

## Site repository

A private, non-archived repository owned by `labs-guigsgbm`, tagged `backstage-static-site`, with a name matching `^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$`. The rule allows at most 58 characters, so prefixing the name with `site-` produces a valid S3 bucket name of at most 63 characters.

The default branch contains:

| Path | Role |
|------|------|
| `src/` | Publish-ready static content; includes the configured index and error documents. |
| `deploy/xstaticsite.yaml` | Exactly one namespaced `XStaticSite` declaration; discovery deploys it to the site's namespace. |
| `catalog-info.yaml` | Backstage `Component` metadata for the repository. |
| `.github/workflows/deploy.yml` | Default-branch and manual publication workflow. |
| `README.md` | Repository layout, discovery topic, document settings, publication, URL and runner credentials. |

Plain static files do not need a build step. If a build is later added, it writes to `dist/`, and the workflow publishes `dist/` instead of `src/`.

## XStaticSite and bucket

| Field | Contract |
|-------|----------|
| API / kind | `platform.k8s-platform.io/v1alpha1`, `XStaticSite` |
| `metadata.name` | Exactly the GitHub repository name. |
| `metadata.namespace` | Omitted in the repository declaration; Argo CD deploys it into `site-<repository-name>`. |
| `spec.indexDocument` | The filename used by the S3 website configuration and present in `src/`; initially `index.html`. |
| `spec.errorDocument` | The filename used by the S3 website configuration and present in `src/`; initially `error.html`. |
| `status.bucketName` | Deterministically `site-<repository-name>`. |
| `status.websiteURL` | `http://site-<repository-name>.s3-website.localhost.localstack.cloud:4566/`. |
| Readiness | Crossplane infrastructure readiness only; it does not assert that site files have been uploaded. |

One `XStaticSite` owns one LocalStack bucket and its website configuration and read policy. Crossplane does not manage website objects; the repository workflow owns all published objects. Removing an `XStaticSite` deletes its bucket and content.

## Published website

The workflow synchronizes the configured content directory into the repository's bucket through the runner's `AWS_ENDPOINT_URL`, using the dummy credentials and region supplied by the runner. It derives the bucket from the repository name, waits no longer than 10 minutes for the bucket, then prints the URL matching `status.websiteURL`. It has no Kubernetes access and does not read repository secrets.

## Repository placeholders

Repository-specific files use only these template-time values:

| Placeholder | Initial value | Used for |
|-------------|---------------|----------|
| `__REPOSITORY_NAME__` | The created repository name | XStaticSite and catalog identity and project-slug. The workflow derives this value from GitHub Actions context. |
| `__INDEX_DOCUMENT__` | `index.html` | XStaticSite index setting and matching file under `src/`. |
| `__ERROR_DOCUMENT__` | `error.html` | XStaticSite error setting and matching file under `src/`. |
