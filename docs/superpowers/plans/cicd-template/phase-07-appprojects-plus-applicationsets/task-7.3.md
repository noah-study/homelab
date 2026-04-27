# Task 7.3: ApplicationSet for apps/ (dev + prod)

**Phase**: 7 (AppProjects + ApplicationSets)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 7.3' 섹션

---


**Files:**
- Create: `argocd/applicationsets/apps-dev.yaml`
- Create: `argocd/applicationsets/apps-prod.yaml`

- [ ] **Step 1: apps-dev (Image Updater 활성)**

`argocd/applicationsets/apps-dev.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-dev
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/noah-study/homelab
        revision: main
        directories:
          - path: apps/*/overlays/macmini/dev
          - path: apps/_base
            exclude: true   # _base는 디렉토리지만 Application 아님
  template:
    metadata:
      name: 'dev-{{index .path.segments 1}}'
      annotations:
        argocd-image-updater.argoproj.io/image-list: 'app=ghcr.io/noah-study/{{index .path.segments 1}}'
        argocd-image-updater.argoproj.io/app.update-strategy: digest
        argocd-image-updater.argoproj.io/app.allow-tags: 'regexp:^(develop|feat-.*|fix-.*)-[0-9a-f]{7,}$'
        argocd-image-updater.argoproj.io/write-back-method: 'git:secret:argocd/image-updater-github-token'
        argocd-image-updater.argoproj.io/write-back-target: 'kustomization'
        argocd-image-updater.argoproj.io/git-branch: 'image-updater/{{index .path.segments 1}}-dev'   # 별도 브랜치 → PR
    spec:
      project: apps
      source:
        repoURL: https://github.com/noah-study/homelab
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{index .path.segments 1}}-dev'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 2: apps-prod (Image Updater 비활성, 수동 promotion)**

`argocd/applicationsets/apps-prod.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-prod
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/noah-study/homelab
        revision: main
        directories:
          - path: apps/*/overlays/macmini/prod
  template:
    metadata:
      name: 'prod-{{index .path.segments 1}}'
      # Image Updater 어노테이션 없음 — 수동 promotion만
    spec:
      project: apps
      source:
        repoURL: https://github.com/noah-study/homelab
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{index .path.segments 1}}-prod'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

- [ ] **Step 3: commit + ArgoCD 자동 sync (root Application이 argocd/ recurse)**

```bash
git add argocd/projects/ argocd/applicationsets/
git commit -m "feat(argocd): AppProjects + ApplicationSets for platform and apps"
git push

argocd app sync root
sleep 30
argocd app list
# platform-cilium, platform-cert-manager 등이 보여야 함 (apps-* 는 아직 0개)
```

- [ ] **Step 4: 검증 commit**

```bash
echo "## Phase 7 acceptance" > tests/acceptance/phase-7-applicationsets.md
echo "- [x] AppProjects platform + apps registered" >> tests/acceptance/phase-7-applicationsets.md
echo "- [x] ApplicationSet platform syncs all platform/* dirs" >> tests/acceptance/phase-7-applicationsets.md
echo "- [x] ApplicationSets apps-dev/apps-prod ready (0 apps yet)" >> tests/acceptance/phase-7-applicationsets.md
git add tests/acceptance/phase-7-applicationsets.md
git commit -m "test: phase 7 acceptance"
git push
```

---
