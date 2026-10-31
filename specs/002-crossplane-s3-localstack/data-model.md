# Data Model: Manage LocalStack S3 Buckets with Crossplane

## Provider Package

Represents an installed Crossplane package that adds Kubernetes APIs and controllers for AWS-compatible resources.

| Attribute | Requirement |
|-----------|-------------|
| Package | AWS provider family and S3 service provider |
| Version | Exact `v2.4.0` for both packages |
| Compatibility | Declared compatible with Crossplane `2.0+`; installed Crossplane is `v2.4.2` |
| Lifecycle state | Installed and healthy before ProviderConfig or S3 Bucket resources are added |

The provider packages are declared in Git as Crossplane `Provider` resources and reconciled by Argo CD. They do not contain AWS credentials.

## Provider Configuration

Represents the AWS connection settings shared by managed resources targeting the LocalStack emulator.

| Attribute | Requirement |
|-----------|-------------|
| Scope | Cluster-scoped provider configuration for namespaced S3 resources |
| Endpoint | `http://localstack.localstack.svc.cluster.local:4566` |
| Endpoint services | Route `s3`, `s3control`, and any API used by reconciliation to the LocalStack endpoint |
| S3 addressing | Path-style requests enabled |
| Credentials | Reference to a `crossplane-system` Secret containing only LocalStack dummy credentials |
| Safety | No real AWS credentials; no silent fallback to public AWS endpoints |

The Secret is an operational prerequisite created outside Git. The `ClusterProviderConfig` reference is version controlled.

## S3 Bucket

Represents a locally simulated S3 bucket managed by Crossplane.

| Attribute | Requirement |
|-----------|-------------|
| API | Namespaced S3 Bucket API installed by the pinned provider package |
| Namespace | `crossplane-system` |
| Region | `us-east-1` for the LocalStack exercise |
| Provider reference | The LocalStack `ClusterProviderConfig` |
| Desired state | Declared in Git and reconciled by Argo CD and Crossplane |
| Readiness | Crossplane reports `Ready=True` and `Synced=True`; the bucket is visible through LocalStack |
| Deletion | Explicitly documented; Git removal alone does not prune the resource with the current root application |

## Relationships

- The provider package installs the ProviderConfig and S3 Bucket APIs.
- The ProviderConfig references the dummy-credentials Secret and the LocalStack endpoint.
- Each Bucket references the ProviderConfig and is reconciled against LocalStack.
