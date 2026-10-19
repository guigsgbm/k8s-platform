# Contract: LocalStack Endpoint

The lab's only interface is an AWS-compatible HTTP endpoint plus a few documented operational conventions. Clients and the lab owner can rely on the following.

## Endpoints

| Consumer | Address | Availability |
|----------|---------|--------------|
| In-cluster workloads | `http://localstack.localstack.svc.cluster.local:4566` | While the `localstack` Application is `Healthy` |
| Workstation | `http://localhost:4566` after `kubectl port-forward svc/localstack 4566:4566 -n localstack` | While the port-forward is running |
| Public internet / Minikube node address | None | Never exposed |

The local port may be changed (for example `4567:4566`); only the left-hand side changes.

## Client conventions

- Credentials: any non-empty dummy values, for example `AWS_ACCESS_KEY_ID=test` and `AWS_SECRET_ACCESS_KEY=test`. Real AWS credentials must never be used.
- Region: any valid region, for example `us-east-1`.
- Clients must set the endpoint explicitly, for example `aws --endpoint-url <endpoint> ...`.
- Account ID reported by the emulator is `000000000000`.
- For an example AWS-compatible check, run `sts get-caller-identity`, then create and list a disposable S3 bucket at the in-cluster endpoint using the pinned `amazon/aws-cli:2.38.1` image; the procedure is in the [quickstart](../quickstart.md).

## Service selection

All services available to the account plan are exposed through the same edge endpoint by default. The operator can restrict startup services by adding a `SERVICES` entry to the chart's `extraEnvVars` in `applications/localstack.yaml` (for example `s3,sqs`). This does not change either endpoint. Verify service names and plan entitlements against the pinned LocalStack release.

## Health

`GET <endpoint>/_localstack/health` returns HTTP 200 with a JSON document of service states when the instance is ready. The pod's readiness probe uses this path.

## Argo CD / Git contract

| Item | Contract |
|------|----------|
| Manifest | `applications/localstack.yaml`, picked up by the existing root application |
| Application name / namespace | `localstack` in `argocd` |
| Target namespace | `localstack`, created by `CreateNamespace=true` or beforehand |
| Required out-of-band object | Secret `localstack-auth-token`, key `auth-token`, in namespace `localstack` |
| Versions | Chart `0.7.1`, image `2026.10.0`; changes require an explicit commit |

## Stability

State is ephemeral: after a restart, previously created simulated resources are not guaranteed to exist.
