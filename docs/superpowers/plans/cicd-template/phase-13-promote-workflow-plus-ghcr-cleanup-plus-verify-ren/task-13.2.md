# Task 13.2: GHCR cleanup workflow

**Phase**: 13 (Promote Workflow + GHCR Cleanup + Verify-Renders)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 13.2' 섹션

---


**Files:**
- Create: `.github/workflows/ghcr-cleanup.yml`
- Create: `scripts/ghcr-cleanup.sh`

- [ ] **Step 1: cleanup 스크립트**

`scripts/ghcr-cleanup.sh`:
```bash
#!/usr/bin/env bash
# GHCR 이미지의 오래된 버전을 정리한다 (sha-prod-* 영구 보관, 그 외 최근 10개 유지).
# 사용법: GH_TOKEN=... ./scripts/ghcr-cleanup.sh
set -euo pipefail
ORG="noah"
KEEP=10

PACKAGES=$(gh api "/orgs/$ORG/packages?package_type=container" --paginate -q '.[].name')

for PKG in $PACKAGES; do
  echo "=== $PKG ==="
  # 모든 버전 (생성일 내림차순)
  VERSIONS=$(gh api "/orgs/$ORG/packages/container/$PKG/versions" --paginate -q \
    'sort_by(.created_at) | reverse | .[] | {id, tags: .metadata.container.tags, created_at}')

  echo "$VERSIONS" | jq -c . | while read -r V; do
    ID=$(echo "$V" | jq -r .id)
    TAGS=$(echo "$V" | jq -r '.tags | join(",")')
    if echo "$TAGS" | grep -q "sha-prod-\|^prod-"; then
      echo "  KEEP (prod tagged): $ID $TAGS"
      continue
    fi
    # KEEP 카운터 — 처음 N개만 유지
    : "${COUNT:=0}"
    COUNT=$((COUNT + 1))
    if [ "$COUNT" -le "$KEEP" ]; then
      echo "  KEEP (recent #$COUNT): $ID $TAGS"
    else
      echo "  DELETE: $ID $TAGS"
      gh api -X DELETE "/orgs/$ORG/packages/container/$PKG/versions/$ID" || echo "    failed"
    fi
  done
  unset COUNT
done
```

```bash
chmod +x scripts/ghcr-cleanup.sh
```

- [ ] **Step 2: cron workflow**

`.github/workflows/ghcr-cleanup.yml`:
```yaml
name: ghcr-cleanup
on:
  schedule:
    - cron: '0 2 * * SUN'   # 매주 일요일 02:00 UTC
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - run: ./scripts/ghcr-cleanup.sh
        env:
          GH_TOKEN: ${{ secrets.GHCR_CLEANUP_PAT }}   # packages:delete 권한 PAT 필요
```

(주: `GITHUB_TOKEN`은 packages 삭제 권한이 없어 별도 PAT 필요. fine-grained PAT 생성 후 secret으로 등록.)
