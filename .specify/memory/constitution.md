# Kubernetes Infrastructure GitOps Constitution

## Core Principles

### I. Git Is the Source of Truth
The repository defines the desired state for platform applications running on Kubernetes. Argo CD reconciles that state. Routine changes must be made in Git rather than applied directly to the cluster. Emergency out-of-band changes must be reconciled back into Git.

### II. Keep the Repository Focused on Platform Infrastructure
This repository manages Kubernetes infrastructure applications and their deployment configuration, not application workloads. Each managed application must have an explicit Argo CD `Application` manifest under `applications/`, which is watched by the root application.

### III. Reconcile Automatically, Prune Deliberately
Applications should use automated synchronization and self-healing. Automated pruning is enabled by default so resources removed from Git do not remain unmanaged in the cluster. Disable pruning for an individual application only when deletion could cause data loss or otherwise requires separate lifecycle management; document the reason alongside that application's configuration. Before deleting or renaming a manifest, assess the resources and dependencies that Argo CD may remove.

### IV. Keep Secrets Out of Git
Never commit plaintext credentials, tokens, private keys, or other secret values. Store secret material in an external secrets system and commit only the references and configuration needed to retrieve it. Grant applications and controllers only the access required for their purpose.

### V. Make Deployments Reproducible
Pin production chart, image, and other upstream dependency versions. Do not use floating references such as `latest` in production. Keep each application's upstream source, version, destination, and relevant sync behavior explicit in its manifest.

### VI. Review Changes for Their Cluster Impact
Before merging or applying a change, manually review its YAML syntax, Kubernetes API/resource compatibility, source and destination, version changes, and synchronization or deletion effects. Use available validators when practical, but CI is not a prerequisite. Changes that can cause data loss or broad cluster impact require particular scrutiny.

## Repository Constraints

- The root Argo CD application watches `applications/`; add platform applications there as individual manifests.
- Keep bootstrap configuration limited to what is needed to install and manage the root application.
- Document operational assumptions and meaningful exceptions where maintainers can find them.

## Change Workflow

- Express desired-state changes as focused, reviewable Git changes.
- For upgrades, removals, or changes to sync and pruning behavior, assess affected resources and dependencies before applying the change.
- When an emergency requires a direct cluster change, record the resulting desired state in Git as soon as practical.

## Governance

This constitution guides the design, implementation, and review of all changes in this repository. Exceptions must be explicit, scoped, and documented with their rationale. Amendments require updating this document and its version and amendment date.

**Version**: 1.0.0 | **Ratified**: 2026-10-19 | **Last Amended**: 2026-10-19
