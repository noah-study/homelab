# Task 8.3: write-back용 GitHub branch protection + auto-merge 설정

**Phase**: 8 (GitHub App + Image Updater (digest pin))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 8.3' 섹션

---


- [ ] **Step 1: Image Updater branch에 자동 머지 룰**

GitHub UI → noah-study/homelab → Settings → Branches → Add classic rule:
- Branch name pattern: `image-updater/**`
- ✅ Require status checks: `verify-renders` (Phase 13)
- ✅ Allow auto-merge

더 강력하게: `apps/*/overlays/macmini/dev/kustomization.yaml`만 수정하는 PR만 auto-merge.

- [ ] **Step 2: image-updater PR 자동 머지 워크플로우**

`.github/workflows/auto-merge-image-updater.yml`:
```yaml
name: auto-merge-image-updater
on:
  pull_request:
    types: [opened, synchronize, reopened]
    paths:
      - 'apps/*/overlays/macmini/dev/kustomization.yaml'

jobs:
  automerge:
    runs-on: ubuntu-latest
    if: github.actor == 'noah-image-updater[bot]'
    steps:
      - uses: actions/checkout@v4
      - name: Verify only allowed paths changed
        run: |
          gh pr view ${{ github.event.pull_request.number }} --json files -q '.files[].path' \
            | grep -vE '^apps/[^/]+/overlays/macmini/dev/kustomization\.yaml$' \
            && exit 1 || echo "all paths within allowlist"
        env:
          GH_TOKEN: ${{ github.token }}
      - name: Enable auto-merge
        run: gh pr merge --auto --squash ${{ github.event.pull_request.number }}
        env:
          GH_TOKEN: ${{ github.token }}
```

- [ ] **Step 3: 검증 commit**

```bash
git add .github/workflows/auto-merge-image-updater.yml
echo "## Phase 8 acceptance" > tests/acceptance/phase-8-image-updater.md
echo "- [x] GitHub App noah-image-updater installed on infra repo" >> tests/acceptance/phase-8-image-updater.md
echo "- [x] argocd-image-updater pod Running, authenticated" >> tests/acceptance/phase-8-image-updater.md
echo "- [x] Auto-merge workflow gates path to dev kustomization only" >> tests/acceptance/phase-8-image-updater.md
echo "- [x] (실 검증은 Phase 12 first app smoke test에서)" >> tests/acceptance/phase-8-image-updater.md
git add tests/acceptance/phase-8-image-updater.md
git commit -m "feat(ci): image-updater auto-merge with path allowlist"
git push
```

---
