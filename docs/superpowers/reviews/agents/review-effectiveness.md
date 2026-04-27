# Review: Effectiveness & Completeness

**Perspective**: 5개 agent 세트가 50 task plan을 정말 가속·안전화하는가?
**Date**: 2026-04-28

## TL;DR

5개 agent 세트는 **80% 시나리오에 잘 맞지만 진단↔수정 사이클의 매끄러움, 수동 UI 핸드오프, security-auditor 검증 매트릭스 완결성에서 갭이 있다**. 가장 중요한 결함: (1) debug-detective 진단 결과를 builder 재dispatch에 전달하는 명시적 패턴 부재 → 사용자 수작업 중계, (2) acceptance-tester가 phase별 manual checklist를 자동 검증할 수 있다고 약속하지만 plan의 acceptance md들은 대부분 `[x] 상태` 마크일 뿐 실행 가능 명령이 명시 안 됨 — 진짜 검증은 어떻게? (3) plan-task-extractor를 명시 제외했는데, builder가 매 dispatch마다 4046줄 plan을 grep으로 파싱하면 컨텍스트 비효율 + 잘못된 task 매칭 위험.

## Findings

### F1. [Critical] debug-detective ↔ builder 진단-수정 사이클 패턴 미정의
- **Scenario**: Phase 8 Task 8.2 dispatch → builder가 image-updater pod CrashLoop 검증 실패 → 사용자가 `homelab-debug-detective "image-updater pod CrashLoop"` dispatch → 진단 리포트 받음 ("SealedSecret namespace mismatch") → 사용자가 `homelab-builder Phase 8 Task 8.2 재시도`. **문제**: 새 builder dispatch는 fresh context — 이전 진단을 모름. 사용자가 진단 내용을 builder 프롬프트에 일일이 복사 붙여넣어야 함.
- **Mitigation**:
  - design doc에 "진단 결과 핸드오프 프로토콜" 정의: detective가 출력을 `tests/diagnostics/<timestamp>-<phase-task>.md`에 파일로 저장
  - builder dispatch 시 "최근 진단 파일 확인 후 적용" 룰 추가
  - 또는 dispatch 시 detective 진단 path를 builder 프롬프트에 명시 ("Phase 8 Task 8.2 재시도. 진단: tests/diagnostics/20260428-08.02.md 참조")
- **Day-1 vs deferred**: **Day-1**. 패턴 없으면 첫 장애에서 부닥침.

### F2. [Critical] acceptance-tester가 자동 검증 가능하다고 약속, 실제론 manual md만
- **Scenario**: agent design Section 5.4는 acceptance-tester가 "manual checklist를 자동/반자동으로 실행"한다고 약속. 하지만 plan의 `tests/acceptance/phase-N-*.md`는 다음 형식:
  ```
  - [x] sealed-secrets controller running
  - [x] master key backed up to 1Password + USB
  ```
  → "controller running" 검증은 `kubectl get deployment` 가능, "1Password에 백업"은 사람만 검증 가능. 실행 가능 명령이 manual md에 없음.
- **Design ambiguity**: design doc은 "자동/반자동"이 정확히 무엇을 의미하는지 안 정함.
- **Mitigation**:
  - acceptance-tester에 "각 체크리스트 항목을 시도 가능한 명령으로 매핑하고, 매핑 불가능한 것은 사용자에게 수동 확인 요청" 룰 추가
  - 더 강력: plan의 manual checklist 형식을 다음으로 통일:
    ```
    - [ ] {description} | {verify-command-or-MANUAL}
    ```
    예: `- [ ] sealed-secrets controller running | kubectl wait --for=condition=available deployment/sealed-secrets -n kube-system --timeout=10s`
  - acceptance-tester는 verify-command 부분 자동 실행, MANUAL은 사용자 확인 요청
- **Day-1 vs deferred**: **Day-1**. 안 하면 acceptance-tester는 단순 "체크리스트 보여주기" agent로 전락.

### F3. [Critical] plan-task-extractor 제외 결정의 컨텍스트 비효율
- **Scenario**: builder는 매 dispatch마다 plan에서 "Phase N Task N.M" section을 찾아야 함. 4046줄 plan을 통째로 Read하면 ~80K 토큰. 50회 dispatch = 4M 토큰 누적 단순 반복 (Opus 4.7 1M 컨텍스트 안에서도 비싸고 캐시 무효 위험).
- **추가 위험**: grep으로 추출 시 task boundary가 모호 — Task 3.2가 `### Task 3.2:`부터 다음 `### Task` 직전까지인데, sub-step 구분자 `- [ ] **Step N:**`을 task 경계로 오해 가능.
- **Mitigation**:
  - design doc Section 4 명시 제외 결정 재고: plan-task-extractor는 1번 호출에 ~200줄만 반환 → 컨텍스트 절약
  - 또는 builder가 grep -A 200 로 정확한 추출 패턴 박아두기 (system prompt에)
  - 가장 좋은 해결: plan을 task별 파일로 split (`docs/superpowers/plans/2026-04-26-cicd-template/{phase-N-task-N.M.md}`) → builder가 정확히 1 파일 Read
- **Day-1 vs deferred**: **Day-1** (split 또는 명시적 추출 패턴 박기).

### F4. [High] 수동 UI 핸드오프 패턴이 design doc에 없음
- **Scenario**: plan의 8 task가 수동 UI (CF Dashboard, GitHub App 생성, 1Password 백업). builder는 BLOCKED 리포트만 함. 사용자는 그 후:
  - UI 작업이 정확히 어떤 input/output을 요구하는지 다시 plan 확인
  - 결과 (예: GitHub App ID, Installation ID, Private Key 위치)를 다음 builder dispatch에 전달
- **갭**: 핸드오프 input/output 데이터 형식 미정. 사용자가 매번 임시 결정.
- **Mitigation**:
  - design doc에 "BLOCKED 리포트 표준 필드" 추가:
    ```
    Manual pending:
      Action: <what user must do>
      Input: <what UI fields to fill>
      Output to capture: <what to write back to where>
      Resume command: <next dispatch prompt template>
    ```
- **Day-1**.

### F5. [High] security-auditor 검증 매트릭스 — spec 3.4와 review F1~F17 완전 매핑 결여
- **Scenario**: design doc Section 5.3은 "spec Section 3.4 9개 + review F1~F17 17개 = 26개 컨트롤"이라 했지만 검증 명령 예시는 3개만 (digest pin, Cosign verify, PSA). 나머지 23개 컨트롤의 자동 검증 명령은 미정의 → auditor agent가 매번 즉흥적으로 짜야 함.
- **Mitigation**:
  - design doc에 "보안 컨트롤 매트릭스" 표 추가 — 각 행: 컨트롤명 + spec/review 출처 + 자동 검증 명령 + 실패 시 액션
  - 매트릭스가 길어지면 별도 파일 `docs/superpowers/security-controls.md`로 분리, auditor system prompt가 Read
- **Day-1**.

### F6. [High] "수정 + 부분 phase 재실행" 시나리오 미정의
- **Scenario**: Phase 5에서 cert-manager 적용 후 며칠 뒤 review F11(Cloudflare Tunnel 토큰 회전 권고) 적용하려면 phase 5 일부만 재실행 필요. 5개 agent 중 어느 것도 "phase의 특정 Task만 idempotent 재실행" 패턴 정의 없음.
- **Mitigation**:
  - builder system prompt에 "기존 리소스 존재 시 idempotent 처리" 명시 (apply는 자연 idempotent, write는 overwrite 명시)
  - 또는 별도 `homelab-task-rerun` agent 정의 (state 검증 후 차이만 적용) — 추가 agent로 카운트 6개 됨
- **Day-1 vs deferred**: **Day-1** (system prompt 보강), 별도 agent는 deferred.

### F7. [High] "plan 외 즉흥 작업"의 지정 agent 부재
- **Scenario**: 운영 중 사용자가 "잠깐 ingress-nginx pod 재시작해줘" 또는 "sample-app dev 로그 마지막 100줄" 요청. 어느 agent? builder는 plan task만, inspector는 read-only, debug-detective는 장애 상황. 즉흥 운영 task의 home이 없음.
- **Mitigation**:
  - 옵션 A: inspector를 "read-only + 명시 안전 write 일부 (rollout restart, scale)" 까지 확장 — 단, 안전성 흐려짐
  - 옵션 B: 별도 `homelab-ops` agent 정의 (운영 task 전용)
  - 옵션 C: 사용자가 직접 kubectl 사용 (가장 간단, agent 무용)
- **Day-1 vs deferred**: **Day-1 결정**. 옵션 B 추천 — 운영 task 빈도 높을 것.

### F8. [Medium] inspector ↔ debug-detective 책임 모호
- **Scenario**: inspector도 logs/events를 읽을 수 있고, debug-detective도 같은 일을 함. 차이는? design doc은 inspector를 "상태 보고", detective를 "장애 진단"이라 구분하지만 사용자 입장에서 "왜 안 되지?"는 두 agent 모두에 호출 가능.
- **Mitigation**:
  - 명시 구분: inspector = "단일 객체/질문, raw 상태 보고", detective = "다층 cross-cutting 분석, 가설 ranking"
  - description에 가이드라인 추가
- **Day-1**.

### F9. [Medium] dispatch 빈도 추정에 ad-hoc 비중 누락
- **Scenario**: design doc은 50+/18/18/ad-hoc/ad-hoc로 추정. ad-hoc은 보통 phase당 1-3회 = 18-54회 추가 = 총 dispatch ~120-150회 / 2-3주. 1회 dispatch에 5-10분 소요(agent 작업 + 사용자 review) → 하루 ~3시간 agent 작업, 2-3주 완료 일관됨. 하지만 디스플레이 안 된 추정.
- **Mitigation**: design doc Section 7 사용 시나리오에 일평균 dispatch 횟수 + 시간 추정 추가 (운영 사이클 가시화).
- **Day-1**.

### F10. [Medium] security-auditor가 phase 종료 시 "1회"만 호출되면 erosion 위험
- **Scenario**: Phase 5 종료 시 보안 PASS, 하지만 Phase 7에서 Phase 5 컴포넌트(ingress-nginx)에 patch 작업 → 보안 컨트롤(예: PSA exception) 의도치 않게 무력화. 다음 호출은 Phase 7 종료 시 → 그 사이 violations 누적 가능.
- **Mitigation**:
  - "phase 종료 + 모든 cross-phase modify 후" 호출 룰
  - 또는 GitHub Actions workflow로 nightly auditor 실행 자동화 (단, agent를 CI에서 부르려면 별도 메커니즘 필요 — 현 plan 범위 밖)
  - 단순 해결: design doc에 "보안 컨트롤 영향 가능한 변경 후 auditor 호출" 룰 명시
- **Day-1**.

### F11. [Medium] Korean/English 출력 일관성 미정의
- **Scenario**: 사용자는 한국어로 대화. spec/plan은 한국어. agent 출력은 한국어인가 영어인가? design doc 출력 포맷 예시는 영어 (`Files Created`, `Verification`). 혼용 시 review 마찰.
- **Mitigation**: design doc에 명시: "system prompt는 한국어, 출력은 한국어. 단 명령/식별자/log 라인은 원어 그대로."
- **Day-1**.

### F12. [Medium] 명시 제외 후보 중 plan-tracker 제외 판단 재고
- **Scenario**: 50+ task 진행 중 "지금 어느 phase, 어느 task까지 완료?"를 매번 git log + tests/acceptance/ 디렉토리 둘러봐야 파악. TaskList tool은 단일 세션 한정 — 다음 세션에서 사라짐. 진행 가시화 약함.
- **Mitigation**:
  - 별도 agent 안 만들고, builder가 task 완료 시 `docs/superpowers/progress.md`에 한 줄 append (date, phase, task, commit) 룰 추가
- **Day-1**.

## Cross-cutting concerns

1. **agent 간 상태 전달이 파일 기반 또는 사용자 중계만 가능** — 명시적 IPC 없음. fresh context 모델의 본질적 한계.
2. **plan 자체의 가공이 필요** — 4046줄 단일 파일은 builder dispatch 단위로 비효율. task별 split 또는 추출 도구 도입.
3. **"이 작업을 할 때 어느 agent?"의 의사결정 트리 부재** — design doc Section 7 시나리오는 정상 흐름만, 분기점 없음.
4. **agent 5개 집합이 evolution 친화적이지 않음** — plan 변경 시 system prompt 5개 동기 업데이트, drift 위험.

## Recommendation (design doc 변경 제안)

### Section 3 (Agent set)
- **F7**: 6번째 agent `homelab-ops` 추가 — 즉흥 운영 task 전용 (Bash write 일부, kubectl rollout restart/scale 등 명시 안전 명령)
- **F12**: builder 시스템 프롬프트에 "task 완료 시 progress.md 갱신" 룰 추가 (별도 agent 없이)

### Section 4 (제외 후보 재검토)
- **F3**: plan-task-extractor 제외 결정 reverse — 또는 plan을 task별 파일로 split (더 깔끔)
- **F12**: plan-tracker 제외 유지하되 builder의 progress 갱신 룰로 대체

### Section 5 (Agent 상세 설계)
- **F1**: design doc Section 5.5 (debug-detective)에 "진단 결과를 `tests/diagnostics/<ts>-<phase-task>.md`에 저장" 룰 추가
- **F2**: design doc Section 5.4 (acceptance-tester)에 "manual checklist 항목 형식 표준화 + verify-command 매핑" 룰 추가 + plan의 acceptance md 형식 update task 추가
- **F4**: design doc Section 5.1 (builder)의 출력 포맷에 "Manual pending" 표준 필드 5개 (Action/Input/Output/Resume/Owner) 추가
- **F5**: design doc Section 5.3 (security-auditor)에 "26개 컨트롤 매트릭스" 별도 파일 (`docs/superpowers/security-controls.md`)로 외부화, auditor가 Read
- **F6**: design doc Section 5.1 (builder)에 "idempotent 처리" 규칙 명시 (kubectl apply, kustomize edit, 파일 overwrite, gh repo create --skip-existing)
- **F8**: inspector vs detective 사용 가이드라인을 description field에 명시
- **F11**: 모든 5개 agent의 출력 언어 정책 (한국어 본문 + 원어 식별자) 한 곳에 명시

### Section 7 (사용 시나리오)
- **F1, F4**: 정상 흐름 + 장애 흐름 + 즉흥 운영 흐름 3종 시나리오 추가
- **F8**: "이 작업은 어느 agent?" 의사결정 트리 (3-4단계 questionnaire) 추가
- **F9**: 일평균 dispatch + 시간 추정 추가
- **F10**: "보안 컨트롤 영향 가능한 변경 후 auditor 호출" 룰 명시

### 신설 Section 10 — Operational Patterns
- 진단↔수정 핸드오프 프로토콜 (F1)
- BLOCKED 핸드오프 표준 (F4)
- idempotent 재실행 패턴 (F6)
- agent ↔ agent 파일 기반 상태 전달 (cross-cutting #1)
