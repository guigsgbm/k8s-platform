# Quickstart: Validating LocalStack in the Lab

Validation guide for [spec.md](./spec.md). The manifest content itself is produced during implementation (`/speckit-tasks`, `/speckit-implement`). Endpoint details are in [contracts/localstack-endpoint.md](./contracts/localstack-endpoint.md).

## Prerequisites

- Minikube running, kubectl context `minikube`, Argo CD healthy in namespace `argocd`, root application `k8s-platform` `Synced`.
- A LocalStack account on the Free plan and an auth token, created in the LocalStack web app. Treat it like a password.
- The AWS CLI on your workstation (for step 5).

## 1. Review before merging

```sh
helm template localstack localstack --repo https://localstack.github.io/helm-charts \
  --version 0.7.1 -n localstack \
  --set image.repository=localstack/localstack \
  --set-string image.tag=2026.10.0 \
  --set service.type=ClusterIP \
  --set ingress.enabled=false \
  --set persistence.enabled=false \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=512Mi \
  --set resources.limits.memory=2Gi \
  --set tests.image.repository=amazon/aws-cli \
  --set-string tests.image.tag=2.38.1 \
  --set-json 'extraEnvVars=[{"name":"LOCALSTACK_AUTH_TOKEN","valueFrom":{"secretKeyRef":{"name":"localstack-auth-token","key":"auth-token"}}}]'
```

These flags mirror the values in `applications/localstack.yaml`. Confirm the rendered LocalStack image is `localstack/localstack:2026.10.0`, the AWS CLI test image is `amazon/aws-cli:2.38.1`, `LOCALSTACK_AUTH_TOKEN` comes from `secretKeyRef`, the Service is `ClusterIP` on port 4566, and no PersistentVolumeClaim is rendered. Neither image uses a floating `latest` tag.

## 2. Create the namespace and token Secret (out of Git)

If the namespace does not already exist, create it:

```sh
kubectl create namespace localstack
```

Read the token without echoing it, and send it to `kubectl` through standard input. The token is not included in shell history or a process argument:

```sh
set +x
set -o pipefail
read -r -s -p "LocalStack auth token: " LOCALSTACK_AUTH_TOKEN
printf '\n'
if [ -z "$LOCALSTACK_AUTH_TOKEN" ]; then
  unset LOCALSTACK_AUTH_TOKEN
  printf 'The LocalStack auth token cannot be empty.\n' >&2
  exit 1
fi
if ! printf '%s' "$LOCALSTACK_AUTH_TOKEN" |
  kubectl create secret generic localstack-auth-token -n localstack \
    --from-file=auth-token=/dev/stdin --dry-run=client -o yaml |
  kubectl apply -f -; then
  unset LOCALSTACK_AUTH_TOKEN
  printf 'Failed to create the LocalStack token Secret.\n' >&2
  exit 1
fi
unset LOCALSTACK_AUTH_TOKEN
kubectl get secret localstack-auth-token -n localstack \
  -o go-template='{{if index .data "auth-token"}}auth-token key exists{{else}}auth-token key is missing{{end}}{{"\n"}}'
```

The final command reports only whether the key exists; it does not display the token. If the Secret already exists, update it only through a similarly non-echoing stdin procedure.

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
  --image=amazon/aws-cli:2.38.1 \
  --env=AWS_ACCESS_KEY_ID=test --env=AWS_SECRET_ACCESS_KEY=test --env=AWS_DEFAULT_REGION=us-east-1 \
  --command -- sh -c '
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 sts get-caller-identity &&
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 s3 mb s3://lab-smoke &&
    aws --endpoint-url http://localstack.localstack.svc.cluster.local:4566 s3 ls'
```

**Expected**: account `000000000000`, bucket `lab-smoke` created and listed, no real AWS account involved (SC-002). The smoke test uses the pinned `amazon/aws-cli:2.38.1` image, also used by the chart's tests.

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

Persistence is disabled. Simulated resources are held in ephemeral pod state and are lost when LocalStack restarts or is recreated; only use disposable resources for this check.

```sh
kubectl rollout restart deploy/localstack -n localstack
kubectl rollout status deploy/localstack -n localstack
```

Repeat the `s3 ls` check. **Expected**: the `lab-smoke` bucket is gone, confirming ephemeral state.

## 7. Removal

1. Delete `applications/localstack.yaml` from Git and push. The root application has `prune: false`, so the LocalStack `Application` remains.
2. Delete the Application with its resources (cascade), using the Argo CD UI (**Delete → Foreground/Cascade**) or `argocd app delete localstack --cascade`.
3. Delete the Secret and namespace: `kubectl delete namespace localstack`.
4. Verify: `kubectl get applications.argoproj.io -n argocd` no longer lists `localstack`, and `kubectl get ns localstack` reports not found.

## Service selection

By default, LocalStack makes all services available under the same edge endpoint, subject to the selected plan's entitlements. To restrict which services start, add a `SERVICES` entry under `spec.source.helm.valuesObject.extraEnvVars` in `applications/localstack.yaml`, for example `name: SERVICES` and `value: s3,sqs`. This changes the emulator configuration, not the client endpoint. Confirm the service names and plan availability for the pinned LocalStack release before using them.
