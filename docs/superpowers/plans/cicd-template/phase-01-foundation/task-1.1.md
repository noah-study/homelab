# Task 1.1: Repo 초기화 + 디렉토리 골격

**Phase**: 1 (Foundation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 1.1' 섹션

---


**Files:**
- Create: `.gitignore`
- Create: `README.md`
- Create: 디렉토리 골격 (`bootstrap/`, `argocd/`, `platform/`, `apps/`, `scripts/`, `tests/`, `docs/`)

- [ ] **Step 1: git init + branch 명명**

```bash
cd /Users/noah/Development/homelab
git init
git branch -m main
```

- [ ] **Step 2: `.gitignore` 작성**

`.gitignore`:
```
# Local k8s context
*.kubeconfig
kubeconfig*
.envrc
.env
.env.*

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp

# Secrets (실수 방지 — *.sealed.yaml만 commit)
*.secret.yaml
*.unsealed.yaml
secrets/
master.key.backup

# Build artifacts
node_modules/
dist/
*.log
```

- [ ] **Step 3: 디렉토리 골격 생성**

```bash
mkdir -p bootstrap/{argo-cd,cluster-resources}
mkdir -p argocd/{projects,applicationsets}
mkdir -p platform/{cilium,ingress-nginx,cert-manager,cloudflared,sealed-secrets,argocd-image-updater,grafana-alloy,trivy-operator,actions-runner-controller,kyverno}
mkdir -p apps/_base
mkdir -p scripts
mkdir -p tests/acceptance
mkdir -p docs
mkdir -p .github/workflows
```

- [ ] **Step 4: 최소 README 작성**

`README.md`:
```markdown
# infra

Mac mini 셀프호스팅 GitOps 프레임워크. 자세한 설계는 `docs/superpowers/specs/`.

## Quick start

신규 셋업:
1. `bootstrap/README.md` 절차 따라 ArgoCD 부트스트랩
2. `docs/onboarding.md` 따라 첫 앱 추가

운영:
- 사고 대응: `docs/runbook.md`
- 규모 확장: `docs/scaling-playbook.md`
```

- [ ] **Step 5: 첫 commit**

```bash
git add .gitignore README.md
git commit -m "chore: initialize infra repo with directory skeleton"
```

- [ ] **Step 6: GitHub remote 생성 + push**

```bash
gh repo create noah-study/homelab --private --source=. --remote=origin --push
```

기대 결과: `https://github.com/noah-study/homelab` 접근 가능.
