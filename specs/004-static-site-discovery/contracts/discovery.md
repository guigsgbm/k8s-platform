# Contract: Static-Site Discovery

Reference manifests (replace `<ORG>`; the draft was accepted by `kubectl apply --dry-run=server`). They are copied to `applications/` during implementation.

## `applications/static-sites-project.yaml` (FR-004, FR-005)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: static-sites
  namespace: argocd
spec:
  description: Discovered static-site repositories
  sourceRepos:
    - https://github.com/<ORG>/*
  destinations:
    - server: https://kubernetes.default.svc
      namespace: site-*
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
      name: site-*
  namespaceResourceWhitelist:
    - group: platform.k8s-platform.io
      kind: XStaticSite
```

## `applications/static-site-discovery.yaml` (FR-001 to FR-003, FR-006, FR-007, FR-011, FR-014)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: static-site-discovery
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  syncPolicy:
    preserveResourcesOnDeletion: true
    applicationsSync: create-update
  generators:
    - scmProvider:
        github:
          organization: <ORG>
          excludeArchivedRepos: true
          # tokenRef: {secretName: github-discovery-token, key: token}   # optional, Secret created outside Git
        cloneProtocol: https
        requeueAfterSeconds: 180
        filters:
          - repositoryMatch: '^[a-z0-9]([a-z0-9-]{0,56}[a-z0-9])?$'
            labelMatch: '^backstage-static-site$'
  template:
    metadata:
      name: 'site-{{.repository}}'
    spec:
      project: static-sites
      source:
        repoURL: '{{.url}}'
        targetRevision: HEAD
        path: deploy
      destination:
        server: https://kubernetes.default.svc
        namespace: 'site-{{.repository}}'
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
```

## `applications/static-site-namespace-policy.yaml` (FR-005, FR-013)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: static-site-own-namespace
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [platform.k8s-platform.io]
        apiVersions: ['*']
        operations: [CREATE, UPDATE]
        resources: [xstaticsites]
  matchConditions:
    - name: site-namespace-only
      expression: object.metadata.namespace.startsWith('site-')
  validations:
    - expression: >-
        !has(object.metadata.annotations) ||
        !('argocd.argoproj.io/tracking-id' in object.metadata.annotations) ||
        object.metadata.annotations['argocd.argoproj.io/tracking-id'].startsWith(object.metadata.namespace + ':')
      message: An XStaticSite in a site namespace may only be deployed by that site's own Application.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: static-site-own-namespace
spec:
  policyName: static-site-own-namespace
  validationActions: [Deny]
```

## Site repository contract (consumed, not created, by this feature)

- Topic `backstage-static-site` on a repository of the configured organization, not archived, name matching the namespace rule.
- `deploy/` on the default branch contains only `XStaticSite` (`platform.k8s-platform.io/v1alpha1`) manifests at its top level.
- `metadata.namespace` is omitted or equals `site-<repo>`.
- Removing a manifest from `deploy/` deletes the resource and its bucket on the next sync.

## Operator contract

| Question | Command |
|----------|---------|
| Which sites are discovered? | `kubectl -n argocd get applications.argoproj.io \| grep '^site-'` |
| Why is a repository missing or failing? | `kubectl -n argocd get application site-<repo> -o jsonpath='{.status.conditions}{"\n"}{.status.operationState.message}{"\n"}'` |
| Is discovery itself failing (rate limit, bad owner)? | `kubectl -n argocd get applicationset static-site-discovery -o jsonpath='{.status.conditions}'` |
