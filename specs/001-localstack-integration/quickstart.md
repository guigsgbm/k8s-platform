# Quickstart: Validating LocalStack in the Lab

Validation guide for [spec.md](./spec.md). The manifest content itself is produced during implementation (`/speckit-tasks`, `/speckit-implement`). Endpoint details are in [contracts/localstack-endpoint.md](./contracts/localstack-endpoint.md).

## Prerequisites

- Minikube running, kubectl context `minikube`, Argo CD healthy in namespace `argocd`, root application `k8s-platform` `Synced`.
- A LocalStack account on the Free plan and an auth token, created in the LocalStack web app. Treat it like a password.
- The AWS CLI on your workstation (for step 5).

## 1. Review before merging

```sh
helm template localstack localstack --repo https://localstack.github.io/helm-charts \
  --version 0.7.1 -n localstack -f <values extracted from applications/localstack.yaml>
```

Confirm: Service type is `ClusterIP`; image is `localstack/localstack:2026.10.0` (not `latest`); `LOCALSTACK_AUTH_TOKEN` comes from `secretKeyRef`; no token value appears anywhere in the repository (`git grep -i auth-token` shows only the Secret reference).

## 2. Create the namespace and token Secret (out of Git)

```sh
kubectl create namespace localstack
kubectl create secret generic localstack-auth-token -n localstack \
  --from-literal=auth-token="$LOCALSTACK_AUTH_TOKEN"
```

Export `LOCALSTACK_AUTH_TOKEN` from a source that does not echo or log it (for example, a password manager). Avoid typing it directly on the command line, where it enters shell history.

## 3. Publish and wait for sync

Commit and push `applications/localstack.yaml` to `main`. Then:

```sh
kubectl get applications.argoproj.io -n argocd
kubectl get pods,svc -n localstack
```

**Expected**: `localstack` is `Synced` and `Healthy`, the pod is `1/1 Running`, and the service is `ClusterIP` on 4566, within 10 minutes (SC-001). If the pod is not ready, check `kubectl logs -n localstack deploy/localstack` for an authentication error and verify the Secret exists with the correct key.

## 4. In-cluster smoke test

```sh
kubectl run aws-smoke -n localstack --rm -it --restart=Never \
  --image=<pinned amazon/aws-cli tag> \
  --env=AWS_ACCESS_KEY_ID=test --env=AWS_SECRET_ACCESS_KEY=test --env=AWS_DEFAULT_REGION=us-east-1 \
  --command -- sh -c '
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 sts get-caller-identity &&
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 s3 mb s3://lab-smoke &&
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 s3 ls'
```

**Expected**: account `000000000000`, bucket `lab-smoke` created and listed, no real AWS account involved (SC-002). Pick a specific, pinned AWS CLI image tag when implementing.

## 5. Workstation access

```sh
kubectl port-forward svc/localstack 4566:4566 -n localstack
```

In another terminal:

```sh
curl -s http://localhost:4566/_localstack/health
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test AWS_DEFAULT_REGION=us-east-1 \
  aws --endpoint-url http://localhost:4566 s3 ls
```

**Expected**: health JSON and the `lab-smoke` bucket (SC-003). If port 4566 is in use, use another local port such as `4567:4566`. Confirm there is no external route: `kubectl get ingress,svc -n localstack` shows no ingress and no `NodePort`.

## 6. State and reset

```sh
kubectl rollout restart deploy/localstack -n localstack
kubectl rollout status deploy/localstack -n localstack
```

Repeat the `s3 ls` check. **Expected**: the `lab-smoke` bucket is gone, confirming ephemeral state.

## 7. Removal

1. Delete `applications/localstack.yaml` from Git and push. The root application does not prune, so the LocalStack `Application` remains.
2. Delete the Application with its resources (cascade), using the Argo CD UI (**Delete → Foreground/Cascade**) or `argocd app delete localstack --cascade`.
3. Delete the Secret and namespace: `kubectl delete namespace localstack`.
4. Verify: `kubectl get applications.argoproj.io -n argocd` no longer lists `localstack`, and `kubectl get ns localstack` reports not found.
