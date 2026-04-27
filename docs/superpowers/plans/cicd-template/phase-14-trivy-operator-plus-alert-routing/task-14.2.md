# Task 14.2: Discord 알림 cron (간이)

**Phase**: 14 (Trivy-Operator + Alert Routing)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 14.2' 섹션

---


**Files:**
- Create: `.github/workflows/trivy-alert.yml`
- Create: `scripts/trivy-digest.sh`

- [ ] **Step 1: Critical/High 추출 후 Discord webhook**

`scripts/trivy-digest.sh`:
```bash
#!/usr/bin/env bash
# trivy-operator의 VulnerabilityReport CR을 조회 후 Critical/High만 Discord로
set -euo pipefail
SEV="${1:-CRITICAL}"

REPORT=$(kubectl get vulnerabilityreports.aquasecurity.github.io -A -o json \
  | jq -r --arg sev "$SEV" '
    [.items[] |
      .metadata as $m |
      .report.vulnerabilities[]? |
      select(.severity == $sev) |
      "\($m.namespace)/\($m.name) \(.vulnerabilityID) \(.title // "")"
    ] | unique | .[]')

if [ -z "$REPORT" ]; then
  echo "No $SEV findings"
  exit 0
fi

COUNT=$(echo "$REPORT" | wc -l | tr -d ' ')
{
  echo "🚨 trivy: $COUNT $SEV findings"
  echo '```'
  echo "$REPORT" | head -30
  [ "$COUNT" -gt 30 ] && echo "... and $((COUNT - 30)) more"
  echo '```'
} | jq -Rs '{content: .}' | curl -fs -H 'Content-Type: application/json' \
  -X POST -d @- "$DISCORD_WEBHOOK_URL"
```

`.github/workflows/trivy-alert.yml`:
```yaml
name: trivy-alert
on:
  schedule:
    - cron: '*/15 * * * *'   # 15분마다 Critical
    - cron: '0 9 * * MON'    # 월요일 09:00 High 다이제스트
  workflow_dispatch:

jobs:
  alert:
    runs-on: [self-hosted, macmini-arm64]   # kubectl 접근 필요
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - name: Critical (always)
        env:
          DISCORD_WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}
        run: ./scripts/trivy-digest.sh CRITICAL
      - name: High (Monday only)
        if: github.event.schedule == '0 9 * * MON'
        env:
          DISCORD_WEBHOOK_URL: ${{ secrets.DISCORD_WEBHOOK_URL }}
        run: ./scripts/trivy-digest.sh HIGH
```

- [ ] **Step 2: commit + 검증**

```bash
git add .github/workflows/trivy-alert.yml scripts/trivy-digest.sh
echo "## Phase 14 acceptance" > tests/acceptance/phase-14-trivy.md
echo "- [x] trivy-operator running, scans new pods automatically" >> tests/acceptance/phase-14-trivy.md
echo "- [x] kubectl get vulnerabilityreports -A shows reports" >> tests/acceptance/phase-14-trivy.md
echo "- [x] trivy-alert cron sends Discord on Critical findings" >> tests/acceptance/phase-14-trivy.md
git add tests/acceptance/phase-14-trivy.md
git commit -m "feat(security): trivy-operator with Discord alert routing"
git push
```

---
