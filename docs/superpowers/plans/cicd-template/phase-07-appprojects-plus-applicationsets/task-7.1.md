# Task 7.1: AppProject 정의

**Phase**: 7 (AppProjects + ApplicationSets)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 7.1' 섹션

---


**Files:**
- Create: `argocd/projects/platform.yaml`
- Create: `argocd/projects/apps.yaml`

- [ ] **Step 1: platform AppProject (cluster-wide 리소스 허용)**

`argocd/projects/platform.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Platform components (cluster-wide CRDs allowed)
  sourceRepos:
    - https://github.com/noah-study/homelab
  destinations:
    - namespace: "*"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  orphanedResources:
    warn: true
```

- [ ] **Step 2: apps AppProject (앱 namespace + 좁은 권한)**

`argocd/projects/apps.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Application workloads (namespace-scoped only)
  sourceRepos:
    - https://github.com/noah-study/homelab
  destinations:
    - namespace: "*-dev"
      server: https://kubernetes.default.svc
    - namespace: "*-prod"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  orphanedResources:
    warn: true
```
