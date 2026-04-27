# Task 7.2: ApplicationSet for platform/

**Phase**: 7 (AppProjects + ApplicationSets)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 7.2' 섹션

---


**Files:**
- Create: `argocd/applicationsets/platform.yaml`

- [ ] **Step 1: platform/* 자동 등록**

`argocd/applicationsets/platform.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/noah-study/homelab
        revision: main
        directories:
          - path: platform/*
  template:
    metadata:
      name: 'platform-{{path.basename}}'
    spec:
      project: platform
      source:
        repoURL: https://github.com/noah-study/homelab
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```
