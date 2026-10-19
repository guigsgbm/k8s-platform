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
