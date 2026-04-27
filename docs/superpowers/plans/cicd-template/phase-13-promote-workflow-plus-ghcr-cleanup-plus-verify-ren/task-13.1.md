# Task 13.1: promote workflow

**Phase**: 13 (Promote Workflow + GHCR Cleanup + Verify-Renders)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 13.1' 섹션

---


**Files:**
- Create: `.github/workflows/promote.yml`
- Create: `scripts/promote.sh`

- [ ] **Step 1: workflow + helper script**

`scripts/promote.sh`:
```bash
#!/usr/bin/env bash
# prod overlay의 image tag를 주어진 SHA로 변경 + PR 생성.
# 사용법: ./scripts/promote.sh <service> <sha-short>
set -euo pipefail
SVC="$1"; SHA="$2"
INFRA_ROOT="$(cd "$(dirname "$0")/.." && pwd)"

KUST="$INFRA_ROOT/apps/$SVC/overlays/macmini/prod/kustomization.yaml"
[ -f "$KUST" ] || { echo "ERROR: $KUST not found"; exit 1; }

# yq로 image tag 변경
yq -i "(.images[] | select(.name == \"PLACEHOLDER_IMAGE\")).newTag = \"sha-$SHA\"" "$KUST"

BRANCH="promote/$SVC-sha-$SHA"
cd "$INFRA_ROOT"
git checkout -b "$BRANCH"
git add "$KUST"
git commit -m "promote: $SVC to sha-$SHA"
git push -u origin "$BRANCH"

gh pr create --title "promote: $SVC → sha-$SHA" \
  --body "Promote $SVC to sha-$SHA.\n\nReview manifest diff before merging." \
  --base main --head "$BRANCH" \
  --label promotion
```

```bash
chmod +x scripts/promote.sh
```

`.github/workflows/promote.yml`:
```yaml
name: promote
on:
  workflow_dispatch:
    inputs:
      service:
        required: true
      sha:
        required: true
        description: "short sha (7 chars)"

jobs:
  create-pr:
    runs-on: ubuntu-latest   # cloud runner — 자체 권한만 사용
    environment: production   # GH Environment 게이트 (5분 wait + 본인 승인)
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: mikefarah/yq@bbdd97482f2d439126582a59689eb1c855944955   # v4.44.6
      - run: |
          git config user.name "promote-bot"
          git config user.email "promote-bot@noreply.github.com"
          ./scripts/promote.sh "${{ inputs.service }}" "${{ inputs.sha }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Discord notify
        run: |
          curl -fs -H 'Content-Type: application/json' -X POST \
            -d "{\"content\": \"🚀 Promote PR opened: ${{ inputs.service }} → sha-${{ inputs.sha }}\"}" \
            "${{ secrets.DISCORD_WEBHOOK_URL }}"
```

- [ ] **Step 2: GitHub Environment + Discord 시크릿**

GitHub UI → noah-study/homelab → Settings → Environments → New: `production`:
- Required reviewers: noah
- Wait timer: 5 minutes
- Deployment branches: main only
- Environment secrets: `DISCORD_WEBHOOK_URL` (Discord 채널 webhook)
