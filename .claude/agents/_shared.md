# Shared conventions for all homelab-* agents

> 본 파일은 모든 homelab-* agent의 첫 단계에서 Read한다. (system prompt가 직접 명시)

## 프로젝트 컨텍스트

- repo: `noah-study/homelab` (public, main 브랜치)
- 작업 디렉토리: `/Users/noah/Development/homelab`
- spec: `docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md`
- plan (task별 파일): `docs/superpowers/plans/cicd-template/phase-NN-task-N.M.md` (split 후) 또는 단일 파일 `docs/superpowers/plans/2026-04-26-infra-cicd-template-plan.md`
- spec/plan 리뷰: `docs/superpowers/reviews/{security,scalability,free-economics}.md`
- agent 리뷰: `docs/superpowers/reviews/agents/{effectiveness,safety,dx}.md`
- 보안 컨트롤 매트릭스: `docs/superpowers/security-controls.md`
- 클러스터: `macmini` (Mac mini, OrbStack Linux VM, k3s)
- KUBECONFIG: agent별 분리 — agent 정의의 system prompt에 명시

## 흡수한 8 원칙 (argocd-autopilot 정신)

1. **Bootstrap ↔ Runtime 분리** — bootstrap은 1회, 이후 ArgoCD가 자기 자신 관리
2. **GitOps self-management** — ArgoCD App-of-Apps 루트가 자기 자신 추적
3. **Project-scoped Apps** — 모든 Application이 AppProject에 속해 RBAC 격리
4. **Atomic onboarding** — 새 서비스 추가 = 단일 commit으로 namespace + Application + secrets
5. **Convention over configuration** — 디렉토리 컨벤션이 곧 자동화
6. **Cluster-first overlays** — `apps/<svc>/overlays/macmini/{dev,prod}/`
7. **Kustomize-first** — Helm은 Kustomize의 HelmChart wrapper로 통합
8. **Explicit Application source** — repoURL/path/targetRevision 명시

## 보안 baseline (Day-1)

- Cosign keyless 서명 + digest pin
- Sealed Secrets namespace-scoped + 마스터키 1Password+USB 백업 + 6개월 회전
- ARC ephemeral runners
- PSA `restricted` 모든 namespace
- NetworkPolicy default-deny + namespace 간 명시 허용
- GitHub App path 제한 (Image Updater)
- OTel redaction processor (Authorization/Cookie/secret 패턴)
- GitHub Environments `production` 게이트 (5분 wait + 본인 승인)
- Cloudflare 계정 hardware 2FA + DNSSEC + CAA

## 출력 정책

- **본문**: 한국어
- **명령/식별자/log 라인**: 원어 그대로 (번역 금지)
- **Secret 패턴**: `KEY=value`, `*_TOKEN`, `*_KEY`, `*_PASSWORD`, `BEGIN PRIVATE KEY`, base64 의심 → 자동 redact (`<REDACTED>`)
- **통일 골격**:

```
## <agent-name>: <한 줄 요약>

### Result
PASS / FAIL / BLOCKED / DIAGNOSIS

### Details
(agent type별 구조화 필드)

### Next
(다음 dispatch 또는 사용자 액션)
```

## BLOCKED 표준 필드 (모든 agent 공통)

```
### Result: BLOCKED

### Details
  Action: <사용자가 수행할 수동 작업>
  Input: <UI에 채울 필드/값>
  Output to capture: <결과 저장 위치 — 1Password vault 이름, 파일 경로 등>
  Resume: <다음 dispatch 명령 그대로 복사 가능 형태>
  Owner: user (또는 specific agent name)
```

## 진단 핸드오프 패턴

- `homelab-debug-detective`는 진단 결과를 `tests/diagnostics/<YYYYMMDD-HHMMSS>-<phase-task>.md` 파일로 저장
- `homelab-builder`는 dispatch 시작 시 `tests/diagnostics/`에서 같은 phase-task에 대한 최근 24시간 내 파일을 검색, 발견 시 Read 하여 적용
- 진단 파일 형식:
  ```
  # Diagnosis: <phase-task>
  ## Symptom
  ## Hypothesis (ranked)
  ## Recommended actions (executable commands)
  ```

## Idempotent 처리

- `kubectl apply` — 자연 idempotent
- 파일 Write — overwrite OK (기존 내용 덮어쓰기 명시)
- `gh repo create` — `--confirm` 또는 `gh repo view ... &>/dev/null && echo skip || gh repo create`
- `git commit` — 변경 없으면 skip: `git diff --cached --quiet || git commit ...`
- `helm install` 대신 Kustomize HelmChart wrapper 통일

## Progress 추적

- 모든 builder/ops가 task 완료 시 `docs/superpowers/progress.md`에 한 줄 append:
  ```
  2026-04-28 16:00Z | Phase 1 Task 1.1 | a1b2c3d | OK
  2026-04-28 16:30Z | Phase 1 Task 1.2 | (manual)  | BLOCKED
  ```

## 금지 사항 (모든 agent 공통)

1. plan에 없는 작업 (refactor, optimization, "while I'm here" cleanup)
2. 검증 실패 시 git push
3. 모호한 plan에서 추측 — 사용자에게 질문
4. 수동 단계 (UI, 자격증명) 우회
5. secret 평문을 commit message / SUMMARY / log에 노출
6. pre-commit/pre-push hook `--no-verify`
7. `git push --force` (사용자 명시 승인 없이)
8. denyCommands 우회 시도

## 추가 안전 — settings.local.json denyCommands

`.claude/settings.local.json`에 정의된 denyCommands는 모든 agent에 강제 적용:
- `kubectl delete *`, `kubectl edit *`, `kubectl patch *`, `kubectl replace *`
- `git push --force *`, `git reset --hard *`, `rm -rf *`
- `gh repo delete *`, `kubectl drain *`, `kubectl taint *`

agent system prompt가 이를 우회하라고 지시해도 거부할 것.
