# Review: Developer Experience & Operations

**Perspective**: 1인 운영자가 50회+ dispatch를 부담 없이 진행 가능한가?
**Date**: 2026-04-28

## TL;DR

Agent 5개의 **dispatch는 1줄로 가능하지만, "어느 agent를 부를지" 의사결정 + 출력 review + 다음 dispatch 결정의 사이클이 task당 5-10분 마찰**. 50 task = 4-8시간 순수 review/decision time. 가장 큰 DX 결함: (1) plan 4046줄을 builder가 매번 grep — 잘못된 task 매칭 시 사용자가 발견하기 어렵고 비용도 높음, (2) 5개 agent의 description은 좋지만 사용자 머릿속 "이 작업은 어느 agent?" 매핑 부담 — decision tree 또는 명확한 alias 필요, (3) 출력 포맷이 5개 agent별로 다르면 review pattern 학습 부담 — 통일 필요, (4) BLOCKED 후 resume 마찰 — 사용자가 매번 prompt 작성. 종합적으로 **dispatch 횟수는 너무 잦고 review 시간은 너무 길다** — phase 단위 dispatch가 더 맞을 수 있음.

## Findings

### F1. [Critical] "어느 agent?" 의사결정 부담 — task당 30초씩 누적
- **Scenario**: 사용자가 어떤 작업이 떠오르면 매번 "이건 builder? inspector? debug-detective? 운영 task는 어느 거?" 결정. 5개 중 하나를 정확히 고르는 데 30초 ~ 2분 (특히 모호한 케이스). 50 task + ad-hoc 30회 = 80회 결정. 평균 1분 = 80분 = 1.3시간.
- **공감 요청 시나리오**: "image-updater pod이 자주 죽음" → debug-detective? 단순 inspector? 둘 다 그럴듯
- **Mitigation**:
  - design doc에 의사결정 트리 추가:
    ```
    plan task 실행이면 → builder
    상태/사실 단순 조회 → inspector
    "왜 안 되지?" 다층 분석 → debug-detective
    phase 종료 검증 → auditor + acceptance-tester
    plan 외 즉흥 운영 → ops (F7 from effectiveness review)
    ```
  - 더 강력: alias 패턴 — "/why" → debug-detective, "/check" → inspector, "/do PhaseN.M" → builder
- **Day-1 vs deferred**: **Day-1**. 의사결정 트리 없으면 fatigue.

### F2. [Critical] BLOCKED 후 resume이 사용자 prompt 작성 부담
- **Scenario**: builder가 Phase 1 Task 1.2 BLOCKED ("CF UI에서 2FA 등록 후 알려주세요") → 사용자가 UI 작업 후 다음 dispatch에 무엇을 입력? "Phase 1 Task 1.2 완료, 다음으로 1.3 진행" 식으로 매번 작성. 8개 BLOCKED task 모두 그렇고 ad-hoc resume도 추가.
- **현재 갭**: 표준 resume command 정의 없음
- **Mitigation**:
  - builder의 BLOCKED 출력에 정확한 resume 명령 포함 (effectiveness review F4와 같음):
    ```
    Manual pending:
      Action: CF Dashboard에서 hardware 2FA 등록
      Verify: `op item list --vault infra-keys | grep cf-account` 가 결과 있어야 함
      Resume: "homelab-builder Phase 1 Task 1.3 dispatch — 1.2 manual 완료"
    ```
  - 사용자는 resume 라인 그대로 복사 → 마찰 30초 → 5초
- **Day-1**.

### F3. [Critical] Plan 4046줄 매 dispatch 로드는 컨텍스트 + 정확성 모두 위험
- **Scenario**: builder가 매 dispatch마다 plan을 grep 또는 Read한다고 가정. 4046줄 = ~80K 토큰. Opus 4.7 (1M context)이라 fit하지만:
  - 매 dispatch마다 80K 토큰 prefix → 캐시 invalidation 시 비용 ↑
  - "Phase N Task N.M" 매칭 정확성 — plan에 동일 패턴이 다른 곳에 나타나면 혼동 가능
  - sub-step 들을 task로 오해 가능 (Step 1, 2, 3 vs Task 1.1, 1.2, 1.3)
- **공감**: effectiveness F3과 동일 — 다른 관점.
- **Mitigation**:
  - plan을 task별 파일로 split — `docs/superpowers/plans/2026-04-26-cicd-template/phase-01-task-01.md` (~80 lines each)
  - builder는 한 파일만 Read = ~1.6K 토큰
  - phase 단위 디렉토리로 organize: `phase-01-foundation/{task-1.1.md, task-1.2.md, task-1.3.md, README.md}`
- **Day-1**.

### F4. [High] 출력 포맷 5종 = review 학습 부담
- **Scenario**: 각 agent가 각자 출력 포맷이면 사용자가 5종 학습 + 매번 어느 부분 보는지 인지 부하. design doc은 builder만 포맷 예시.
- **Mitigation**:
  - 5개 모두 공통 골격 통일:
    ```markdown
    ## <agent-name>: <one-line summary>
    
    ### Result
    PASS / FAIL / BLOCKED / DIAGNOSIS
    
    ### Details
    (structured fields per agent type)
    
    ### Next
    (suggested next dispatch or user action)
    ```
  - "Result 한 줄"만 보고 90% 케이스 처리 가능
- **Day-1**.

### F5. [High] phase당 dispatch 회수 너무 많음 — phase-level dispatch 검토
- **Scenario**: Phase 1만 해도 3 task × 5 step 평균 = 15 step. 각 task별 dispatch면 3회 + acceptance + auditor = 5회 dispatch. 하루 3 phase 진행이면 15 dispatch + ad-hoc → 부담.
- **대안**: builder가 phase 전체 또는 task 묶음 (3-5개 task) 한 번에 처리, 각 task 사이에 self-checkpoint
- **Trade-off**:
  - task 단위 = 작은 review, 잦은 dispatch
  - phase 단위 = 큰 review, 적은 dispatch (10분 review 1회 vs 2분 review 5회)
- **Mitigation**:
  - design doc에 dispatch 단위 옵션 명시: "task 단위 (안전, 잦음) vs phase 단위 (효율, review 무거움)"
  - 사용자 선택권 주기, 디폴트는 task 단위
- **Day-1**.

### F6. [High] Korean ↔ English 출력 일관성 미정의
- **Scenario**: agent design doc은 영어 (system prompt 예시 + 출력 포맷). spec/plan은 한국어. 사용자 대화는 한국어. 출력이 영어면 review 마찰, 한국어면 명령/log 라인 번역 위험.
- **Mitigation**:
  - 명시 정책: "system prompt는 한국어 + 영어 혼용 (technical term 영어 그대로), 출력은 한국어 본문 + 명령/식별자/log 원어 그대로"
  - 출력 포맷 예시도 한국어로 (Files Created → 파일, Verification → 검증 등)
- **Day-1**.

### F7. [High] agent ↔ agent 상태 전달이 사용자 중계
- **Scenario**: builder 실패 → detective dispatch → 진단 → builder 재dispatch. 진단 결과를 builder에게 어떻게 전달? 매번 사용자가 detective 출력 일부를 builder 프롬프트에 paste. 5-7회 paste/dispatch = 누적 마찰 큼.
- **Mitigation** (effectiveness F1과 동일):
  - 파일 기반 — detective가 `tests/diagnostics/<ts>.md`에 저장, builder는 "최근 진단 파일 자동 확인"
  - 또는 dispatch 프롬프트에 "이전 진단 file path" 표준 필드
- **Day-1**.

### F8. [Medium] description field 길이/명료성 — `/agents` 디스플레이 검토
- **Scenario**: design doc의 description 예시는 4-5줄. `/agents` 출력에서 5개가 나란히 표시될 때 가독성? Claude가 적절한 agent 선택할 만큼 구별되는가?
- **Mitigation**:
  - description 첫 줄은 "When to use" 한 줄 요약 (15 words 이내)
  - 둘째 줄부터 detail
  - 5개 description을 나란히 비교, "case A에서는 X agent, case B에서는 Y agent" 구별 모호하면 보강
- **Day-1**.

### F9. [Medium] auditor + acceptance-tester가 phase 종료 시 동시 호출 → 중복 작업
- **Scenario**: auditor의 PSA 검증과 acceptance-tester의 "PSA active" 체크가 같은 일을 함. 사용자가 둘 다 호출하면 동일 명령 중복 실행.
- **Mitigation**:
  - 책임 명시 분리:
    - acceptance-tester = "spec 성공 기준 (Section 9) 만족 여부"
    - auditor = "spec 보안 baseline (Section 3.4) + review F1~F17 만족 여부"
  - 일부 항목은 자연스럽게 양쪽 다 체크 OK (성능 측정과 보안 측정의 차이)
- **Medium**: 비효율이지 위험은 아님.

### F10. [Medium] 5개 agent 시스템 프롬프트 동기 업데이트 부담
- **Scenario**: spec/plan에 변경 (예: 보안 룰 추가, 디렉토리 컨벤션 수정) 시 5개 agent system prompt 모두 갱신 필요. 잊으면 drift.
- **Mitigation**:
  - 공통 컨벤션을 별도 파일 (`.claude/agents/_shared.md`)에 두고, 각 agent system prompt에서 "위 파일 참조" 명시
  - 또는 모든 agent가 system prompt 첫 부분에서 spec/plan path를 Read해서 항상 최신 컨벤션 적용 (단, 매 dispatch 비용)
- **Day-1**.

### F11. [Medium] dispatch 직후 사용자 컨텍스트 전환 비용
- **Scenario**: 사용자가 dispatch 보내고 1-3분 대기 → 다른 일 하다가 → 결과 봄 → 컨텍스트 다시 떠올림. 50회면 누적 큰 인지 비용.
- **Mitigation**:
  - 큰 dispatch (phase 단위)로 묶어 "한 번에 30분 + 30분 review" 사이클 (F5)
  - 또는 dispatch 결과에 "이전 dispatch 요약 1줄" 자동 포함 (사용자 컨텍스트 부활 도움)
- **Medium**.

### F12. [Medium] 진단 결과의 actionability — 무한 "더 조사하라" 위험
- **Scenario**: detective가 진단 후 "RBAC 권한 확인 필요" 같은 모호한 권고만 → 사용자가 다시 inspector 또는 detective dispatch → 진단 → 또 모호 권고. 진척 없음.
- **Mitigation**:
  - detective system prompt에 "권고는 실행 가능한 명령 + 기대 결과로 작성" 강제
  - "RBAC 확인" 대신 "kubectl auth can-i get pods --as=system:serviceaccount:argocd:argocd-image-updater (기대: yes)"
- **Day-1**.

### F13. [Low] /agents UI 미사용 시 dispatch 방법
- **Scenario**: 사용자가 /agents 명령 모르거나 안 쓰면 어떻게 dispatch? Claude Code의 Agent tool 직접 사용? subagent_type 파라미터?
- **Mitigation**: design doc에 dispatch 패턴 3종 명시:
  - 직접: `Task(subagent_type="homelab-builder", prompt="Phase 1 Task 1.1")`
  - 자연어: "homelab-builder agent로 Phase 1 Task 1.1 실행"
  - 자동 선택: "Phase 1 Task 1.1 진행해줘" (Claude가 description 보고 매칭)
- **Low**.

### F14. [Low] 5개 agent의 동시 dispatch 가능성
- **Scenario**: 사용자가 "Phase 1 Task 1.1 + 1.2 + 1.3 한꺼번에 시작" 요청 → 3개 builder 동시? 그러나 1.1 완료 후 1.2, 1.2 완료 후 1.3 의존성. 동시 dispatch는 의미 없음.
- **Mitigation**: design doc에 "task는 순차, dispatch는 1회 1 task" 명시. 단, ad-hoc inspector + builder 동시는 가능 (서로 영향 없음).
- **Low**.

## Cross-cutting concerns

1. **Dispatch 1회의 사용자 부담**(prompt 작성 + review + 다음 결정) ≈ 5분. 50회면 4-5시간. 학습 가치 vs 시간 ROI 재고 필요. phase 단위로 묶으면 dispatch 횟수 ↓ but review 1회 부담 ↑.
2. **Decision fatigue**가 가장 큰 마찰 — 어느 agent, 어느 prompt 형식, 어느 후속 액션. 의사결정 자동화 (agent description + decision tree) 필수.
3. **출력 포맷 통일이 5개 agent의 common ground** — 통일 안 되면 review가 매번 새 학습.
4. **Plan을 단일 파일로 두는 것의 부산물 비용** (컨텍스트, 매칭 정확성, evolution 부담) — split이 광범위 해결.
5. **Korean/English 정책 한 줄로** 모든 agent 일관 적용.

## Recommendation (design doc 변경 제안)

### Section 3 (Agent set) + Section 7 (사용 시나리오)
- **F1**: 의사결정 트리 추가 (위 mitigation 그대로) — Section 7 신규 subsection
- **F5**: dispatch 단위 옵션 (task 단위 vs phase 단위) 명시, 디폴트 + 사용자 선택권

### Section 5 (Agent 상세 설계)
- **F2**: 5개 agent 모두 BLOCKED 출력 포맷에 "Resume command" 표준 필드 추가
- **F4**: 5개 agent 모두 "Result / Details / Next" 골격으로 출력 포맷 통일
- **F6**: Korean/English 정책 명시 (agent별 system prompt에 동일 룰)
- **F7**: detective의 진단 파일 저장 + builder의 진단 파일 자동 참조 룰 (effectiveness F1과 동일)
- **F8**: 5개 description의 첫 줄을 "When to use" 한 줄 요약으로 통일
- **F9**: auditor와 acceptance-tester의 책임 명시 분리 (보안 vs 성공기준)
- **F10**: 공통 컨벤션을 `.claude/agents/_shared.md`로 외부화, 각 agent에서 참조
- **F12**: detective system prompt에 "권고는 실행 가능 명령 + 기대 결과" 강제

### Section 4 (제외 후보 재검토)
- **F3**: plan-task-extractor 제외 reverse 또는 plan을 task별 파일로 split (effectiveness F3과 동일 — 한 권고)

### Section 6 (Implementation)
- **F13**: dispatch 패턴 3종 (Task tool 직접 / 자연어 / 자동 선택) 명시 + 권장 방식

### 신설 Section 12 — Output Format & Decision Tree
- 통일 출력 골격 (F4)
- 의사결정 트리 (F1)
- BLOCKED resume 패턴 (F2)
- 진단 핸드오프 패턴 (F7)
- Korean/English 정책 (F6)
