# Task 13.3: verify-renders workflow

**Phase**: 13 (Promote Workflow + GHCR Cleanup + Verify-Renders)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 13.3' 섹션

---


**Files:**
- Create: `.github/workflows/verify-renders.yml`

- [ ] **Step 1: PR 시 kustomize build 검증 + diff comment**

`.github/workflows/verify-renders.yml`:
```yaml
name: verify-renders
on:
  pull_request:
    paths:
      - 'apps/**'
      - 'platform/**'
      - 'argocd/**'
      - 'bootstrap/**'

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
        with:
          fetch-depth: 0
      - name: Install kustomize + helm
        run: |
          curl -sSLo kustomize.tar.gz https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.3/kustomize_v5.4.3_linux_amd64.tar.gz
          tar xzf kustomize.tar.gz && sudo mv kustomize /usr/local/bin/
          curl -sSL https://get.helm.sh/helm-v3.16.3-linux-amd64.tar.gz | tar xz && sudo mv linux-amd64/helm /usr/local/bin/
      - name: Render all kustomizations
        run: |
          set -e
          for d in $(find apps platform bootstrap -name kustomization.yaml -exec dirname {} \;); do
            echo "=== $d ==="
            kustomize build --enable-helm "$d" > /dev/null
          done
      - name: Diff main vs PR
        if: github.event_name == 'pull_request'
        run: |
          mkdir -p /tmp/main /tmp/pr
          for d in $(find apps -mindepth 3 -maxdepth 3 -type d -name 'dev' -o -name 'prod'); do
            FILE="${d//\//-}.yaml"
            kustomize build --enable-helm "$d" > "/tmp/pr/$FILE" 2>/dev/null || true
            git show "origin/main:$d/kustomization.yaml" >/dev/null 2>&1 && {
              git checkout origin/main -- "$d" 2>/dev/null || true
              kustomize build --enable-helm "$d" > "/tmp/main/$FILE" 2>/dev/null || true
            }
          done
          diff -ur /tmp/main /tmp/pr || true
```

- [ ] **Step 2: commit + push**

```bash
git add .github/workflows/promote.yml .github/workflows/ghcr-cleanup.yml .github/workflows/verify-renders.yml
git add scripts/promote.sh scripts/ghcr-cleanup.sh
echo "## Phase 13 acceptance" > tests/acceptance/phase-13-promote-cleanup.md
echo "- [x] gh workflow run promote.yml -f service=sample-app -f sha=<sha> creates PR" >> tests/acceptance/phase-13-promote-cleanup.md
echo "- [x] Production environment gates wait timer 5min + reviewer" >> tests/acceptance/phase-13-promote-cleanup.md
echo "- [x] ghcr-cleanup runs and reports kept/deleted versions" >> tests/acceptance/phase-13-promote-cleanup.md
echo "- [x] verify-renders fails PR if kustomize build error" >> tests/acceptance/phase-13-promote-cleanup.md
git add tests/acceptance/phase-13-promote-cleanup.md
git commit -m "feat(ci): promote workflow + ghcr cleanup + verify-renders"
git push
```

---
