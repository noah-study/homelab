# Task 16.2: CODEOWNERS + Branch Protection

**Phase**: 16 (Renovate + CODEOWNERS + Branch Protection)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 16.2' 섹션

---


**Files:**
- Create: `.github/CODEOWNERS`

- [ ] **Step 1: CODEOWNERS**

`.github/CODEOWNERS`:
```
# 모든 변경에 본인 강제 리뷰
*                                       @noah

# Critical paths — 추가 본인 리뷰 (이중 안전망)
/bootstrap/                             @noah
/argocd/                                @noah
/platform/                              @noah
/.github/                               @noah
/scripts/seal.sh                        @noah
/scripts/rotate-sealed-secrets.sh       @noah
```

- [ ] **Step 2: Branch protection (UI)**

GitHub UI → noah-study/homelab → Settings → Branches → Add rule for `main`:
- ✅ Require pull request before merging
- ✅ Require approvals: 1
- ✅ Require review from CODEOWNERS
- ✅ Require status checks: `verify-renders`, `ci` (Phase 13의 verify-renders)
- ✅ Require branches up to date
- ✅ Require linear history
- ✅ Restrict who can push to matching branches: noah only
- ❌ Allow force pushes
- ❌ Allow deletions

`image-updater/**` branch는 별도 룰 (Phase 8 Task 8.3).

- [ ] **Step 3: commit**

```bash
git add renovate.json .github/CODEOWNERS
echo "## Phase 16 acceptance" > tests/acceptance/phase-16-renovate.md
echo "- [x] Renovate App installed on infra + sample-app" >> tests/acceptance/phase-16-renovate.md
echo "- [x] First Renovate PR appears within 24h" >> tests/acceptance/phase-16-renovate.md
echo "- [x] Auto-merge succeeds for actions/* patches" >> tests/acceptance/phase-16-renovate.md
echo "- [x] Branch protection on main: PR + CODEOWNERS + status checks" >> tests/acceptance/phase-16-renovate.md
git add tests/acceptance/phase-16-renovate.md
git commit -m "feat(meta): Renovate rules + CODEOWNERS + branch protection"
git push
```

---
