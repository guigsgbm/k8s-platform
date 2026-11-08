# Research: Static-Site Repository Contract

Spec: [spec.md](./spec.md). The existing platform, discovery, and runner behavior was checked against specs 003–005 and the current manifests.

## 1. Bucket identity and status (FR-004, FR-005)

- **Decision**: Derive the S3 bucket external name from the `XStaticSite` `metadata.name` as `site-<name>`. Use the bucket's observed ID for `status.bucketName`, and format `status.websiteURL` from that same ID.
- **Rationale**: The repository workflow has no Kubernetes access, so its repository name must deterministically produce the bucket name that Crossplane reports. The discovery name rule permits up to 58 characters; adding the five-character `site-` prefix stays within S3's 63-character bucket limit.
- **Alternatives considered**: Retain the UID-derived bucket and grant the workflow Kubernetes access to read status; rejected because the runner contract explicitly provides no Kubernetes permission. Guess another bucket; rejected because it could publish to a different site's storage.

## 2. Content ownership and readiness (FR-006)

- **Decision**: Remove the Composition-managed `Object` resource. GitHub Actions owns all published objects, and `XStaticSite` readiness describes infrastructure only.
- **Rationale**: The current Composition reasserts its placeholder at the configured index key and can overwrite the real site file uploaded by the workflow.
- **Alternatives considered**: Keep the placeholder or move it to another key; rejected because it adds no value to the publication contract and leaves Crossplane managing website content.

## 3. Workflow and runner integration (FR-008–FR-013)

- **Decision**: Use the existing `lab-localstack` runner environment and its dummy credentials and LocalStack endpoint. Allow default-branch pushes and manual dispatch, filtering non-default pushes with a job-level condition before runner allocation. Derive the repository name from the Actions event context, wait for the corresponding bucket using AWS CLI only for at most 10 minutes, then use `aws s3 sync`.
- **Rationale**: The runner is ephemeral and has no Kubernetes permission; the bucket name and S3 endpoint are sufficient for readiness and upload. A job-level branch condition prevents non-default-branch pushes from allocating a runner while preserving manual dispatch from the selected branch.
- **Alternatives considered**: Kubernetes polling; prohibited by the runner contract. A workflow trigger with a literal branch filter; it cannot follow arbitrary repositories' default-branch names. Unbounded bucket polling or fallback to AWS; both violate the repository contract.

## 4. Pinned Node.js action (FR-009)

- **Decision**: Install Node.js `24.11.1` with `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1` (`v7.1.0`).
- **Rationale**: The full immutable SHA fixes the action implementation; the explicit Node.js version fixes the runtime. The v7.1.0 release and its tag SHA were checked against GitHub on 2026-11-08. Re-verify the release and SHA immediately before merging the eventual workflow.
- **Alternatives considered**: `@v7` or another version tag; rejected because tags can move. A floating Node.js version such as `24` or `lts/*`; rejected because it resolves differently over time.
- **Sources**: [setup-node v7.1.0 release](https://github.com/actions/setup-node/releases/tag/v7.1.0); [GitHub tag reference](https://api.github.com/repos/actions/setup-node/git/refs/tags/v7.1.0); [GitHub guidance on pinning actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions#using-third-party-actions); [Node.js 24.11.1 release archive](https://nodejs.org/download/release/v24.11.1/).

## 5. Pinned AWS CLI install on the ephemeral runner (FR-009, FR-010)

- **Decision**: Download `https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.38.1.zip` and use its installer in user mode under the runner's home directory. Add the user `bin` directory to `GITHUB_PATH`; do not use `sudo`.
- **Rationale**: The runner already provides `curl` and `unzip`, and the official installer supports a user-local `--install-dir` and `--bin-dir`. The versioned archive avoids installing an unpinned latest release.
- **Alternatives considered**: The unversioned AWS CLI archive; rejected because the installed CLI would float. A system-wide install; rejected because it requires elevated permission. pip/snap; rejected because the runner does not need another installation mechanism and the official CLI v2 installer is sufficient.
- **Install shape**:

  ```sh
  curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.38.1.zip" -o "$RUNNER_TEMP/awscliv2.zip"
  unzip -q "$RUNNER_TEMP/awscliv2.zip" -d "$RUNNER_TEMP"
  "$RUNNER_TEMP/aws/install" --install-dir "$HOME/.local/aws-cli" --bin-dir "$HOME/.local/bin"
  echo "$HOME/.local/bin" >> "$GITHUB_PATH"
  ```

- **Sources**: [AWS CLI v2 Linux installation instructions](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html); [AWS CLI instructions for past releases](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-version.html); [versioned AWS CLI 2.38.1 Linux x86_64 archive](https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.38.1.zip).

## 6. Repository contents and migration scope (FR-001–FR-003, FR-014–FR-016)

- **Decision**: Keep the initial site plain static content under `src/`, with `index.html` and `error.html` matching the `XStaticSite` configuration. Define only the three specified template-time placeholders. Do not add a build framework or Backstage template.
- **Rationale**: This is the shortest repository contract that satisfies discovery, cataloging, infrastructure provisioning, and publication. A later build can target `dist/` without changing the bucket ownership model.
- **Alternatives considered**: Add a framework, package manifest, or build step for plain HTML; rejected as unnecessary. Add repository creation to Backstage; rejected because it is out of scope.

- **Decision**: Treat the current `docs-demo` UID-derived LocalStack bucket as disposable and recreate the lab site without migrating its contents.
- **Rationale**: The existing declaration is in this repository and uses the old bucket identity. Recreating it gives Crossplane a clean managed resource with the deterministic name required by the workflow.
- **Alternatives considered**: Preserve and copy existing bucket contents; rejected by the approved lab assumption and would require a separate migration process.
