# Task 17.2: docs/runbook.md

**Phase**: 17 (Documentation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 17.2' 섹션

---


**Files:**
- Create: `docs/runbook.md`

- [ ] **Step 1: 사고 대응 플레이북**

`docs/runbook.md`:
```markdown
# 사고 대응 플레이북

## 1. GitHub 계정 침해 의심
1. `gh auth logout` + 모든 PAT 즉시 회수 (https://github.com/settings/tokens)
2. GitHub App 4개 (noah-image-updater, noah-arc-runner, Renovate, …)의 private key rotate
3. Branch protection 검증 — `gh api repos/noah-study/homelab/branches/main/protection`
4. Audit log 분석 — `gh api /orgs/noah-study/audit-log`
5. SealedSecret 마스터키 + Cloudflare 토큰은 별도 (각각의 절차로)

## 2. Cloudflare 계정 침해 의심
1. CF Dashboard → My Profile → API Tokens → Roll All
2. cert-manager API 토큰 신규 발급 → SealedSecret 재봉인 + commit
3. cloudflared tunnel 토큰 신규 발급 → 동일
4. CT (Certificate Transparency) 모니터로 사기 발급 인증서 확인 — https://crt.sh/?q=noah.dev
5. 의심 발견 시 LE에 revoke 요청

## 3. SealedSecrets 마스터키 유출
1. **즉시** 회전: `./scripts/rotate-sealed-secrets.sh`
2. 모든 SealedSecret 재봉인 — 외부 시크릿 (DB pwd, API key, OAuth client secret) 전수 회전 후 다시 봉인
3. git history에 노출된 평문이 없는지 검증 — `git log --all -p | grep -iE 'password|secret|token'`
4. 1Password + USB의 옛 마스터키 즉시 폐기

## 4. Self-hosted runner pod 침해 의심
1. ARC pod 강제 종료: `kubectl delete pod -n actions-runner-system -l actions.github.com/scale-set-name=macmini-arm64 --force`
2. NetworkPolicy egress allowlist 위반 확인 — Cilium Hubble로 outbound 분석
3. GHCR 최근 push 검사 — 이상 image 발견 시 `gh api -X DELETE /user/packages/container/<pkg>/versions/<id>`
4. ARC GitHub App private key rotate

## 5. Cosign 서명 인프라 다운
- Sigstore 공식 status: https://status.sigstore.dev
- 빌드 실패 → 임시 우회: reusable workflow의 cosign 단계에 `continue-on-error: true` 토글 (커밋 + push)
- 단, ArgoCD에 Sigstore Policy Controller가 활성된 상태면 unsigned image 배포 안 됨 — 인지하고 수용

## 6. Mac mini 다운 (DR)
1. 새 머신 준비 (또는 OS 재설치)
2. OrbStack + Linux VM + k3s 재설치 (Phase 1-2 절차)
3. `infra` 레포 clone, `bootstrap/README.md` 따라 부트스트랩
4. SealedSecrets 마스터키 복원 (1Password에서 → kubectl apply)
5. 모든 ArgoCD Application이 Healthy 까지 대기
6. DNS 작업 0 (CF Tunnel은 토큰만 있으면 됨)

## 7. 비용 알림 ('Grafana Free 80% 도달' 등)
1. 카디널리티 SLO 위반 — `count by (__name__)({namespace=~".+"})` 상위 10 series 확인
2. Alloy config의 drop rule 보강 (regex 추가) → ArgoCD sync
3. 그래도 초과 임박이면 Grafana Pro 검토 (`docs/scaling-playbook.md`)
```
