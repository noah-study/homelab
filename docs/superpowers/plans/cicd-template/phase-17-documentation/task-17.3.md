# Task 17.3: docs/scaling-playbook.md

**Phase**: 17 (Documentation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 17.3' 섹션

---


**Files:**
- Create: `docs/scaling-playbook.md`

- [ ] **Step 1: 규모별 트리거 + 액션 매트릭스**

`docs/scaling-playbook.md`:
```markdown
# Scaling Playbook

| 규모 | 트리거 신호 | 액션 | 시간 |
|---|---|---|---|
| 1~10 svcs | — | 본 설계 그대로 | — |
| 10~20 | Renovate PR > 30/주 | grouping 강화, runner 2~3개로 (ARC maxRunners) | 30분 |
| 20~30 | Mac mini RAM > 70% | 32GB 업그레이드 또는 ARC 동적 스케일 검증 | 1일 (HW) |
| 30~50 | sqlite latency P99 > 200ms / GitHub API > 70% | k3s `--datastore-endpoint=postgres://...`, ApplicationSet/ArgoCD webhook 전환 | 반나절 |
| 50~100 | RAM > 80% / ArgoCD reconcile lag > 2분 | 2nd Mac mini node, ArgoCD controller sharding (replicas=3), PR Preview 도입 | 1~2일 |
| 100+ | 단일 머신 한계 분명 | k3s 멀티노드 또는 vanilla k8s/managed로 마이그레이션 | 1주~ |

## 유료 전환 우선순위

1. **Grafana Cloud Pro** $19/월 — 가장 빨리 깨짐 (15-20 svcs)
2. **Mac mini 32GB → 64GB** $300 1회 — 메모리 천장
3. **1Password** $3/월 — 사용 중 아니면
4. **GHCR Pro** $5/월 또는 public 전환 — 5+ svcs
5. **CF Pro** $20/월 — 20+ svcs + WAF rules 부족
6. **2nd Mac mini node** $600+ — 50+ svcs

## 원칙
**분명히 깨질 때까지 스택을 바꾸지 말 것.** 임계치 모니터링 알림에 따라서만 액션.
```

- [ ] **Step 2: commit**

```bash
git add docs/onboarding.md docs/runbook.md docs/scaling-playbook.md
echo "## Phase 17 acceptance" > tests/acceptance/phase-17-docs.md
echo "- [x] onboarding.md walks through new app in 5 min" >> tests/acceptance/phase-17-docs.md
echo "- [x] runbook.md covers 7 incident scenarios" >> tests/acceptance/phase-17-docs.md
echo "- [x] scaling-playbook.md has explicit triggers" >> tests/acceptance/phase-17-docs.md
git add tests/acceptance/phase-17-docs.md
git commit -m "docs: onboarding + runbook + scaling playbook"
git push
```

---
