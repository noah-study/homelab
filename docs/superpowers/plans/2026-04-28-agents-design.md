# Custom Agents — Design & Implementation Plan (v2)

**Date**: 2026-04-28
**Version**: v2 (post-3-perspective-review)
**Goal**: homelab plan(50 task / 18 phase)을 효율·안전하게 dispatch하기 위한 custom agent 세트 정의 및 생성.
**Scope**: project-scoped (`.claude/agents/`) — 본 repo 작업에만 사용.
**Reviews informing this**: `docs/superpowers/reviews/agents/{effectiveness,safety,dx}.md`

---

## 0. v2에서 변경된 것 (v1 대비)

3관점 ralph-loop 리뷰의 권고 반영:

| 변경 | 출처 |
|---|---|
| **6번째 agent `homelab-ops` 추가** (즉흥 운영 task 전용) | Effectiveness F7 |
| **Plan을 task별 파일로 split** — `docs/superpowers/plans/cicd-template/phase-NN-task-N.M.md` | Effectiveness F3, DX F3 |
| **보안 컨트롤 매트릭스 외부화** — `docs/superpowers/security-controls.md` | Effectiveness F5 |
| **공통 컨벤션 외부화** — `.claude/agents/_shared.md` | DX F10 |
| **통일 출력 포맷** — `Result / Details / Next` 골격 | DX F4 |
| **BLOCKED resume 표준 필드** — Action/Verify/Resume/Owner | Effectiveness F4, DX F2 |
| **의사결정 트리** (Section 8) | DX F1 |
| **Korean output 정책** | DX F6, Effectiveness F11 |
| **denyCommands + WebFetch allowlist** in settings.local.json | Safety F1, F8 |
| **ServiceAccount RBAC 분리** (agent별 KUBECONFIG) | Safety F5 |
| **`.claude/agents/` 변경 보호** — CODEOWNERS strict + Renovate exclude + Discord 알림 | Safety F3, F9 |
| **debug-detective Bash + 엄격한 denyCommands** | Safety F7 |
| **secret pattern redaction** in output | Safety F6 |
| **builder progress.md 갱신** 룰 (별도 agent 없이) | Effectiveness F12 |
| **진단 파일 핸드오프** 패턴 | Effectiveness F1, DX F7 |
| **idempotent 처리** rule | Effectiveness F6 |
| **acceptance checklist 포맷 표준화** — `- [ ] desc | verify-cmd` | Effectiveness F2 |

---

## 1. Why Custom Agents

50 task dispatch에 매번 컨텍스트(spec/plan 위치, 8 원칙, 디렉토리 컨벤션, 보안 baseline, 출력 포맷, …) 반복 → 컨텍스트 폭증 + 일관성 균열.

**Custom agent ROI**:
- dispatch 프롬프트 1줄 ("Phase 1 Task 1.1 실행")
- 컨벤션 위반 자동 차단 (system prompt 박힘)
- 검증 누락 방지
- WRITE/READ 분리로 안전성 + denyCommands로 기술적 강제
- 출력 구조 통일로 review 시간 ↓

---

## 2. Research: 작업 카테고리 (v1 그대로)

| 카테고리 | 빈도 | 예시 task |
|---|---|---|
| manifest 작성 + apply | 30+ | platform Helm, app overlays, ApplicationSet |
| shell script 작성 + bats | 5 | seal/new-app/ghcr-cleanup |
| GitHub workflow 작성 | 5 | reusable-build-push, promote, verify-renders |
| 수동 UI | 8 | CF, GitHub App, 1Password |
| 검증 (read-only) | 모든 task 끝 | kubectl wait, curl, dig |
| bats 테스트 실행 | 18 | tests/acceptance/checks.bats |
| 장애 진단 | ad-hoc | "왜 sync 안 되지?" |
| 보안 감사 | phase별 | digest pin? PSA? OIDC subject? |
| 문서 갱신 | 5 | runbook, scaling-playbook |
| **즉흥 운영** | ad-hoc | "ingress 재시작", "마지막 로그 100줄" |

---

## 3. Agent Set (6개로 확장)

### 1️⃣ `homelab-builder` — plan task 실행 (workhorse)
- **권한**: WRITE — Bash + Read/Write/Edit/Grep/Glob
- **빈도**: 50+
- **Stop**: 수동 UI / 검증 실패 / plan 모호

### 2️⃣ `homelab-inspector` — read-only 상태 보고
- **권한**: READ-ONLY (Bash 제한된 명령만, no Write/Edit)
- **빈도**: ad-hoc (~30회)
- **Use**: "platform-cilium 상태?", "image-updater 마지막 PR 5개"

### 3️⃣ `homelab-security-auditor` — phase별 보안 감사
- **권한**: READ-ONLY + WebFetch (allowlist)
- **빈도**: phase 종료 시 18회
- **Use**: "Phase 5 보안 감사" → 매트릭스 PASS/FAIL

### 4️⃣ `homelab-acceptance-tester` — phase별 성공 기준 검증
- **권한**: READ-ONLY (Bash bats/kubectl/curl/dig)
- **빈도**: phase 종료 시 18회
- **Use**: "Phase 12 acceptance" → 자동/수동 체크리스트

### 5️⃣ `homelab-debug-detective` — 장애 진단
- **권한**: READ-ONLY + WebFetch (Bash 엄격 denyCommands)
- **빈도**: ad-hoc (~10회)
- **Use**: 장애 시 다층 분석, 가설 ranking, **실행 가능 권고만**
- **출력**: `tests/diagnostics/<ts>-<phase-task>.md`로 자동 저장 (builder가 다음 dispatch에서 참조)

### 6️⃣ `homelab-ops` — 즉흥 운영 task (NEW)
- **권한**: WRITE 일부 — 명시 안전 명령 allowlist (`rollout restart`, `scale`, `logs`, `port-forward`, `cordon`/`uncordon`)
- **빈도**: ad-hoc (~30회)
- **Use**: "ingress-nginx 재시작", "sample-app dev 로그 100줄", "alloy daemonset scale"
- **금지**: namespace/deployment 생성/삭제, RBAC 변경, secret 생성/수정 (그건 builder)

---

## 4. 명시적으로 제외한 후보들

여전히 제외 (v1과 동일):
- `manifest-author`, `script-author`, `cost-monitor`, `runbook-writer`, `plan-tracker`, `github-state` 전용
- `plan-task-extractor`도 제외 — **단, plan을 task별 파일로 split하는 보강 채택** (Section 6 Step 1.5)

미래 검토 후보:
- `homelab-cost-monitor` — 협업자 늘면
- `homelab-runbook-writer` — incident 패턴 누적 시
- `homelab-pr-reviewer` — public repo이므로 외부 PR 받을 가능성 시

---

## 5. Agent 상세 설계

### 5.0 모든 agent 공통 (`.claude/agents/_shared.md`로 외부화)

```markdown
# 공통 컨벤션 (모든 homelab-* agent가 따름)

## 프로젝트 컨텍스트
- repo: noah-study/homelab (public, main)
- 작업 디렉토리: /Users/noah/Development/homelab
- spec: docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md
- plan (task별 파일): docs/superpowers/plans/cicd-template/phase-NN-task-N.M.md
- reviews: docs/superpowers/reviews/{security,scalability,free-economics}.md
- agent 리뷰: docs/superpowers/reviews/agents/{effectiveness,safety,dx}.md
- 보안 컨트롤 매트릭스: docs/superpowers/security-controls.md
- 클러스터: macmini (Mac mini, OrbStack Linux VM, k3s)
- KUBECONFIG: agent별 분리 (Section 11)

## 흡수한 8 원칙 (autopilot)
1. Bootstrap ↔ Runtime 분리
2. GitOps self-management
3. Project-scoped Apps
4. Atomic onboarding
5. Convention over configuration
6. Cluster-first overlays (overlays/macmini/{dev,prod}/)
7. Kustomize-first (Helm은 HelmChart wrapper)
8. Explicit Application source

## 출력 정책
- 본문: 한국어
- 명령/식별자/log 라인: 원어 그대로 (번역 금지)
- secret 패턴 (KEY=value, *_TOKEN, *_KEY, *_PASSWORD, BEGIN PRIVATE KEY) 발견 시 redact
- output 통일 골격:
  ```
  ## <agent>: <one-line summary>

  ### Result
  PASS / FAIL / BLOCKED / DIAGNOSIS

  ### Details
  (agent type별 구조화 필드)

  ### Next
  (다음 dispatch 또는 사용자 액션)
  ```

## BLOCKED 표준 필드
```
### Result: BLOCKED
### Details
  Action: <사용자가 수행할 수동 작업>
  Input: <UI에 채울 필드/값>
  Output to capture: <결과를 어디에 저장할지 — 1Password vault, 파일 등>
  Resume: <다음 dispatch 명령 그대로 복사 가능 형태>
  Owner: user (또는 specific agent)
```

## 진단 핸드오프
- detective는 진단 결과를 `tests/diagnostics/<YYYYMMDD-HHMMSS>-<phase-task>.md`로 저장
- builder dispatch 시 같은 phase-task에 대한 최근 진단 파일이 있으면 자동 Read
- builder system prompt: "tests/diagnostics/ 검색 후 최근 24시간 파일 발견 시 Read하여 적용"

## Idempotent 처리
- kubectl apply는 자연 idempotent
- 파일 Write는 overwrite OK
- gh repo create에 --confirm + 존재 시 skip
- git commit은 변경 없으면 skip (`git diff --cached --quiet || git commit ...`)

## 금지 사항 (모두 적용)
- plan에 없는 작업 (refactor, optimization)
- 검증 실패 시 push
- 추측 (모호한 plan은 사용자에게 질문)
- 수동 단계 우회
- secret 평문을 commit message/SUMMARY/log에 노출
- pre-commit/pre-push hook --no-verify
```

### 5.1 `homelab-builder`

```yaml
---
name: homelab-builder
description: |
  When to use: implement a specific plan task (input: 'Phase N Task N.M').
  Reads plan task file, executes all steps with tests, commits, reports.
  Stops on manual UI / verification failure / plan ambiguity.
tools: Bash, Read, Write, Edit, Grep, Glob
---
```

System prompt 핵심:
1. `_shared.md` 컨벤션 (Read 후 적용)
2. 5단계 절차 (Read task file → 각 step 실행 → 검증 → idempotent commit → progress.md 갱신 → 리포트)
3. progress.md 갱신: `docs/superpowers/progress.md`에 한 줄 append: `2026-04-28 16:00Z | Phase 1 Task 1.1 | a1b2c3d | OK`
4. tests/diagnostics/ 자동 확인
5. Resume 명령 BLOCKED 시 표준 형식

### 5.2 `homelab-inspector`

```yaml
---
name: homelab-inspector
description: |
  When to use: read-only inspection of cluster, repo, GHCR, DNS state.
  Does NOT modify anything. For diagnosing failures, pre-checking phase
  prerequisites, spot-checking after deploys.
tools: Bash, Read, Grep, Glob
---
```

System prompt 핵심:
1. READ-ONLY 룰 (denyCommands는 settings.local.json에서 별도 강제)
2. 자주 쓰는 패턴: kubectl get/describe/logs/events, argocd app get, gh pr list, dig, helm get values
3. 응답: raw output 덤프 X, **요약 + 핵심 fields + anomalies**
4. 모호한 질문 = 명확화 요청

### 5.3 `homelab-security-auditor`

```yaml
---
name: homelab-security-auditor
description: |
  When to use: after a phase completes (or major change) to verify security
  baseline compliance. Runs control matrix from docs/superpowers/security-controls.md.
  Input: phase number or 'all'. Returns PASS/FAIL per control + violations.
tools: Bash, Read, Grep, Glob, WebFetch
---
```

System prompt 핵심:
1. 첫 단계: `docs/superpowers/security-controls.md` Read
2. 각 컨트롤별 자동 검증 명령 실행
3. WebFetch는 allowlist (rekor.sigstore.dev, kubernetes.io 등)
4. 출력: 매트릭스 + violation 상세 + 우선순위 권고

### 5.4 `homelab-acceptance-tester`

```yaml
---
name: homelab-acceptance-tester
description: |
  When to use: validate spec Section 9 success criteria for a completed phase.
  Runs bats tests + executes verify-commands from acceptance md files.
  Manual items reported separately for user check.
tools: Bash, Read, Grep
---
```

System prompt 핵심:
1. acceptance md 형식: `- [ ] description | verify-command-or-MANUAL`
2. verify-command 자동 실행 후 PASS/FAIL
3. MANUAL은 사용자 확인 요청
4. 책임 명시: "spec Section 9 성공 기준" (보안은 auditor 책임)

### 5.5 `homelab-debug-detective`

```yaml
---
name: homelab-debug-detective
description: |
  When to use: complex incidents where cause is non-obvious. Multi-layer
  analysis (logs/events/network/git/GHCR) → ranked hypotheses → actionable
  recommendations. Saves diagnosis file for builder pickup.
tools: Bash, Read, WebFetch
---
```

System prompt 핵심:
1. Bash 권한 있지만 strict denyCommands (settings.local.json) — 변경 명령 0
2. 표준 sweep: events → logs → app status → network policy → recent commits/PRs
3. 흔한 패턴 리스트 (image pull error, RBAC, NetworkPolicy 차단, SealedSecret namespace mismatch, cert pending)
4. **권고는 실행 가능 명령 + 기대 결과**:
   ```
   권고 1: kubectl auth can-i get secrets --as=system:serviceaccount:argocd:argocd-image-updater -n argocd
   기대: yes
   현재: no → RBAC 권한 누락 가능
   ```
5. 진단 결과 파일 저장: `tests/diagnostics/<ts>-<phase-task>.md`

### 5.6 `homelab-ops` (NEW)

```yaml
---
name: homelab-ops
description: |
  When to use: ad-hoc operational tasks NOT in the plan (rollout restart,
  scale, logs, port-forward, cordon/uncordon). Limited write permissions
  via command allowlist. Plan tasks → use homelab-builder instead.
tools: Bash, Read, Grep
---
```

System prompt 핵심:
1. 명시 ALLOW 명령 (write 측면): `kubectl rollout restart`, `kubectl scale`, `kubectl logs`, `kubectl port-forward`, `kubectl cordon/uncordon`, `argocd app sync` (수동), `helm rollback`
2. 명시 DENY: namespace/deployment 생성/삭제, RBAC 변경, secret 생성/수정 (그건 builder)
3. 모든 변경 명령 실행 전 사용자에게 confirm
4. 변경 후 결과 검증 + 리포트

---

## 6. Implementation Plan

### Step 1: 보호 인프라 (Day-0)

#### 1.1 `.claude/settings.local.json` — denyCommands + WebFetch allowlist

```json
{
  "permissions": {
    "additionalDirectories": ["/Users/noah/Development/homelab"],
    "denyCommands": [
      "kubectl delete *",
      "kubectl edit *",
      "kubectl patch *",
      "kubectl replace *",
      "git push --force *",
      "git reset --hard *",
      "rm -rf *",
      "gh repo delete *",
      "kubectl drain *",
      "kubectl taint *"
    ],
    "webFetchAllowlist": [
      "rekor.sigstore.dev",
      "kubernetes.io",
      "k3s.io",
      "argo-cd.readthedocs.io",
      "cert-manager.io",
      "cilium.io",
      "grafana.com",
      "github.com"
    ]
  }
}
```

(주의: 위 키 이름은 예시 — 실제 Claude Code settings.json 스키마 확인 필요. settings는 update-config 스킬로 갱신.)

#### 1.2 ServiceAccount + ClusterRole 분리 (Phase 2 k3s install 시)

`bootstrap/cluster-resources/agent-rbac.yaml` (Phase 2에 추가):
```yaml
# read-only ClusterRole + 4 SA + 4 ClusterRoleBinding
# 1) homelab-admin (cluster-admin) — 사용자 직접 사용 (default)
# 2) homelab-inspector (read-only across cluster)
# 3) homelab-builder-ns (특정 namespace에만 write — 시작은 default+observability+platform)
# 4) homelab-auditor (read + WebFetch는 cluster 외 영역)
# kubeconfig per SA: ~/.kube/macmini-<sa-name>.kubeconfig
```

각 agent system prompt에 KUBECONFIG 환경변수 명시.

#### 1.3 1Password Service Account 생성

- 1Password Business 또는 Family이 있으면 Service Account 생성 (`infra-keys` vault만 권한)
- 토큰을 macOS Keychain에 저장
- builder system prompt: `OP_SERVICE_ACCOUNT_TOKEN=$(security find-generic-password -s op-svc-infra)` 패턴

#### 1.4 `.claude/agents/` 보호

- `.github/CODEOWNERS`에 추가:
  ```
  /.claude/agents/*.md   @noah
  /.claude/agents/_shared.md @noah
  ```
- `renovate.json`에 추가:
  ```json
  "packageRules": [
    {
      "matchPaths": [".claude/agents/**"],
      "enabled": false
    }
  ]
  ```
- Discord webhook으로 .claude/agents/** 변경 PR 알림 (별도 workflow `.github/workflows/agent-change-alert.yml`)

#### 1.5 Plan을 task별 파일로 split

```
docs/superpowers/plans/cicd-template/
├── README.md                   # 18 phase + task 인덱스
├── phase-01-foundation/
│   ├── README.md               # phase 개요
│   ├── task-1.1.md             # Repo 초기화
│   ├── task-1.2.md             # Cloudflare 보안
│   └── task-1.3.md             # Mac mini 호스트
├── phase-02-k3s-cilium/
│   ├── task-2.1.md
│   └── task-2.2.md
... (총 50개 task 파일)
```

기존 4046줄 단일 plan은 보존 (linked from README), task 파일은 plan에서 추출.

### Step 2: 외부화 자료 작성

#### 2.1 `.claude/agents/_shared.md` — 공통 컨벤션 (Section 5.0 그대로)

#### 2.2 `docs/superpowers/security-controls.md` — 26개 컨트롤 매트릭스

```markdown
# Security Controls Matrix

| ID | Control | Source | Verify Command | Failure Action |
|---|---|---|---|---|
| SC-01 | Image digest pin (no tag-only) | spec 3.4.2, Sec F3 | `find apps -name kustomization.yaml ! -path '_base/*' | xargs grep -L 'digest:' | wc -l` (expect 0) | builder/Phase 8 |
| SC-02 | Cosign signature on all GHCR images | spec 3.4.2, Sec F-misc | (loop on GHCR images) cosign verify --certificate-identity-regexp=... | builder/Phase 10 |
| SC-03 | PSA restricted on all app namespaces | spec 3.4.6 | kubectl get ns -o json | jq ... | builder + Kyverno warn |
| SC-04 | NetworkPolicy default-deny per namespace | spec 3.4.6 | for ns in apps; kubectl get netpol -n $ns | grep default-deny | builder/Phase 6+11 |
| SC-05 | GitHub App path restriction (Image Updater) | spec 3.4.4, Sec F2 | gh pr list --search 'author:noah-image-updater[bot]' --json files | path 검사 | builder/Phase 8 |
| SC-06 | Sealed Secrets master key backed up | spec 3.4.3, Sec F4 | op item get sealed-secrets-master --vault infra-keys | manual |
| SC-07 | Cloudflare DNSSEC + CAA | Sec F9 | dig noah.dev DNSKEY/CAA | manual |
| SC-08 | OTel redaction processor active | spec 3.4.7, Sec F10 | kubectl get configmap alloy-config -o yaml | grep redact | builder/Phase 15 |
| SC-09 | ARC ephemeral runners (no _work persistence) | spec 3.4.1, Sec F1, F8 | kubectl get pod -n actions-runner-system -o yaml | grep emptyDir | builder/Phase 9 |
| SC-10 | Renovate actions pinned to SHA | spec 3.4.5, Sec F5 | grep -rE 'uses: actions/[^@]+@v?\d+\.' .github/workflows | (expect 0) | builder/Phase 16 |
... (SC-11 ~ SC-26)
```

(전체 매트릭스는 ~26 행, 별도 파일에)

### Step 3: 6개 agent 파일 생성

| 파일 | 라인 |
|---|---|
| `.claude/agents/_shared.md` | ~80 |
| `.claude/agents/homelab-builder.md` | ~120 |
| `.claude/agents/homelab-inspector.md` | ~70 |
| `.claude/agents/homelab-security-auditor.md` | ~90 |
| `.claude/agents/homelab-acceptance-tester.md` | ~70 |
| `.claude/agents/homelab-debug-detective.md` | ~100 |
| `.claude/agents/homelab-ops.md` | ~90 |

### Step 4: 검증
- 6개 파일 frontmatter YAML valid
- `/agents` 명령으로 6개 모두 표시
- description "When to use" 첫 줄 일관

### Step 5: Smoke test
- inspector: "현재 git 상태 보고" — 안전한 read-only 호출
- builder: "Phase 1 Task 1.1 실행" — dry-run 옵션 또는 실제 시작

---

## 7. 사용 시나리오

### 7.1 정상 흐름 (Phase 진행)
```
Sa: "homelab-builder, Phase 1 Task 1.1 실행"
B: [task 1.1 실행 + commit + progress.md 갱신 + 리포트]
Sa: review 후 "Task 1.2"
B: → BLOCKED 리포트 (CF UI Resume 명령 포함)
Sa: 본인이 CF UI 작업 → resume 명령 그대로 복사 → 다음 dispatch
B: → "Task 1.3 진행..."

[Phase 1 종료]
Sa: "homelab-acceptance-tester, Phase 1"
A: → 자동 verify-command 실행 + manual 항목 확인 요청
Sa: 본인이 manual 항목 확인 → "homelab-security-auditor, Phase 1"
SA: → 매트릭스 PASS

Sa: "Phase 2 시작"
```

### 7.2 장애 흐름
```
Sa: "homelab-builder, Phase 8 Task 8.2"
B: → BLOCKED (검증 실패: image-updater pod CrashLoop)
Sa: "homelab-debug-detective, image-updater pod CrashLoop 원인"
D: [진단] → tests/diagnostics/20260428-160000-08.02.md 저장
D: → 권고: "kubectl get secret -n argocd | grep github-app, 누락 시 builder/Task 8.2 재실행"
Sa: "homelab-builder Phase 8 Task 8.2 재시도"
B: → tests/diagnostics/ 자동 Read → 진단 적용 → 정상 완료
```

### 7.3 즉흥 운영 흐름
```
Sa: "homelab-ops, sample-app dev pod 재시작"
O: → confirm 요청 ("kubectl rollout restart deployment/sample-app -n sample-app-dev 실행할까요?")
Sa: "yes"
O: [실행 + 검증 + 리포트]
```

### 7.4 의사결정 트리

새 dispatch 결정 시:

```
어떤 작업?
├── plan에 정의된 task 실행 → builder
├── 단일 객체 상태 조회 → inspector
├── "왜 안 되지?" 다층 분석 → debug-detective
├── phase 종료 검증
│   ├── spec Section 9 성공 기준 → acceptance-tester
│   └── 보안 baseline (Section 3.4) → security-auditor
├── plan 외 즉흥 운영 (재시작/스케일/로그) → ops
└── 위 어디에도 안 맞음 → 사용자 직접 (kubectl, gh)
```

### 7.5 dispatch 단위
- **디폴트**: task 단위 (작은 review, 잦은 dispatch)
- **옵션**: phase 단위 (큰 review, 적은 dispatch) — `homelab-builder, Phase 1 전체` 호출
- 사용자 선호로 선택

---

## 8. Acceptance criteria (이 agent set 자체)

- [ ] 6개 agent 파일 + _shared.md + security-controls.md 모두 존재 | `ls .claude/agents/*.md docs/superpowers/security-controls.md`
- [ ] 각 frontmatter valid YAML, name unique | yq 검증
- [ ] settings.local.json denyCommands 적용 | settings 확인
- [ ] CODEOWNERS에 .claude/agents/ 보호 | grep
- [ ] inspector smoke test 동작 (git 상태 보고) | dispatch
- [ ] builder Phase 1 Task 1.1 정상 완료 | dispatch + verify
- [ ] 출력 통일 골격 (Result/Details/Next) | 첫 dispatch 결과 review

---

## 9. Threat Model & 신뢰 경계 (Safety review F1~F13 종합)

```
[사용자] (cluster-admin, 1Password personal session, gh PAT)
    │
    ↓ 의도된 권한 분리
    │
[homelab-builder] (Bash + denyCommands, namespace-scoped SA)
    │ ↓ commits, applies (limited)
    ↓
[git repo] ← [homelab-inspector] (read-only SA)
[k3s cluster]                       │ ↓ reads only
[GHCR]                              ↓
[1Password infra-keys] ← [homelab-auditor] (SA + WebFetch allowlist)
                                    │ ↓ reads + Sigstore
                                    ↓
[homelab-debug-detective] (Bash + strict deny, no write)
[homelab-ops] (Bash allowlist 일부 write)
[homelab-acceptance-tester] (Bash bats only)
```

- denyCommands가 1차 방어
- ServiceAccount RBAC가 2차 방어
- Output redaction이 3차 방어

---

## 10. 미래 확장 후보 (지금 X)

- `homelab-cost-monitor` — 협업자 늘면
- `homelab-runbook-writer` — incident 패턴 누적 시
- `homelab-pr-reviewer` — public repo + 외부 PR
- `homelab-renovate-reviewer` — Renovate PR 안전성 평가
