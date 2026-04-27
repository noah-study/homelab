# Ralph Loop — Custom Agents Design Review

대상 design doc: `/Users/noah/Development/homelab/docs/superpowers/plans/2026-04-28-agents-design.md`
참고 자료:
- spec: `/Users/noah/Development/homelab/docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md`
- 구현 plan: `/Users/noah/Development/homelab/docs/superpowers/plans/2026-04-26-infra-cicd-template-plan.md`
- 보안/확장성/무료 리뷰: `/Users/noah/Development/homelab/docs/superpowers/reviews/`

3개 관점에서 agent design을 비판적으로 리뷰하라. 각 관점별로 별도 파일을 생성.

## Review 1 — `review-effectiveness.md` (효과·완결성)

질문: **5개 agent 세트가 50 task plan을 정말 가속/안전화 하는가?**

다룰 항목:
- 각 agent의 책임 경계가 실제 task 분포와 매칭되는가? (Section 2의 카테고리 vs 5 agents)
- 커버리지 갭: plan의 어떤 task가 어느 agent로도 잘 안 풀리는가?
- 책임 중복: 두 agent가 같은 일에 호출 가능한 모호 지점은?
- dispatch 빈도 추정이 합리적인가? (50+, 18, 18, ad-hoc, ad-hoc)
- "수동 UI" task 8건의 처리 — agent가 완료 후 사용자 작업을 받는 핸드오프 패턴이 정의되어 있는가?
- "장애 시 builder 실패 → debug-detective 진단 → builder 재시도" 사이클이 매끄러운가? 진단 결과를 builder에 어떻게 전달?
- security-auditor의 검증 매트릭스가 spec 3.4와 review F1~F17을 빠짐없이 매핑하는가? 빠진 컨트롤?
- acceptance-tester가 phase별 manual checklist의 모든 항목을 자동/반자동으로 실행 가능한가?
- 5개로 충분한가, 더 분할/통합 필요?
- 명시적 제외 후보 7개의 제외 판단이 정확한가?

각 finding 형식: severity (Critical/High/Medium/Low) + 구체 시나리오 + 권고 변경.

## Review 2 — `review-safety.md` (안전·권한)

질문: **agent 권한 모델이 실수/사고/침해 시 폭발 반경을 충분히 제한하는가?**

다룰 항목:
- WRITE/READ 분리가 실제로 강제되는가? (tool 권한이 아니라 system prompt만으로? — 진짜 강제 가능한가?)
- builder agent의 blast radius: Bash 권한이 있어 `rm -rf`, `kubectl delete`, `git push --force` 다 가능. 어떻게 제약?
- inspector agent가 실수로 변경 명령을 실행하지 않도록 system prompt가 충분한가? Bash tool은 본질적으로 write 가능.
- security-auditor가 Sigstore 외부 호출 시 의심스러운 endpoint로 데이터 누출 가능?
- agent dispatch 중 secret 노출 위험: SealedSecrets 마스터키, Cloudflare API 토큰, GitHub App private key를 agent가 접근 가능한 위치에 두는 시나리오?
- log/output에 시크릿이 echo되는 패턴 (예: `echo $TOKEN`)을 차단하는 강제 룰?
- builder가 plan에 없는 작업을 추가로 하는 것을 어떻게 방지? "plan 외 작업 금지"가 system prompt에 있어도 모델이 무시 가능.
- debug-detective가 진단 중 실수로 변경(예: `kubectl rollout restart` 같은 "복구" 시도)할 위험?
- Bash 권한이 있는 agent의 ALLOW/DENY 리스트를 .claude/settings.json hook으로 추가 강제 필요?
- agent 정의 자체가 git repo에 있어 PR로 변경 가능. 악성 PR이 agent system prompt를 변조하면? CODEOWNERS로 보호?

각 finding 형식: severity + 구체 공격/사고 시나리오 + Day-1 mitigation vs deferred.

## Review 3 — `review-dx.md` (개발자 경험·운영성)

질문: **사용자(1인 운영자)가 50회+ dispatch를 부담 없이 진행 가능한가?**

다룰 항목:
- dispatch 프롬프트가 진짜로 1줄로 가능한가? ("Phase N Task N.M") — 모호 케이스에서는?
- 출력 포맷이 빠르게 review 가능한가? 한 task당 사용자 review 시간 목표 1-2분 — 실측 추정?
- 실패 모드 친절성: BLOCKED 시 사용자가 다음에 무엇을 해야 할지 명확한가?
- 5개 agent의 호출 결정이 사용자에게 직관적인가? "이건 builder인가 inspector인가?" 헷갈림 케이스?
- 컨텍스트 한도: 각 agent의 system prompt + plan 발췌 + tool output → 실제 컨텍스트 점유? Opus 4.7 (1M)이라 여유 크지만 효율성 측면.
- security-auditor 응답이 너무 길어 review가 부담스러운 위험?
- debug-detective의 진단이 실행 가능한 권고로 이어지는가, 아니면 "더 조사하라"의 무한 루프?
- agent 간 상태 공유 부재 — builder가 실패한 task를 detective가 모를 가능성. 사용자가 매번 컨텍스트 paste?
- 사용자가 plan에 없는 즉흥 작업 (예: "잠깐 ingress 재시작해줘")을 요청할 때 어느 agent? — case 누락?
- agent 진화: spec/plan 변경 시 5개 agent system prompt 동기화 부담?
- "/agents" 명령으로 본 agent들이 잘 보이는지, description이 사용자가 호출 결정에 도움 되는지?

각 finding 형식: severity + 구체 사용자 마찰 시나리오 + 권고.

---

## Review 문서 구조 (3개 모두 동일)

```
# Review: <Perspective>

## TL;DR (3-5 lines)

## Findings (numbered, severity-tagged)

### F1. [Severity] Title
- **Scenario**: ...
- **Mitigation**: ...
- **Day-1 vs deferred**: ...

(N개 반복)

## Cross-cutting concerns

## Recommendation
- design doc Section 별로 변경 액션 항목
```

## 완성 조건 (`<promise>AGENT REVIEWS COMPLETE</promise>` 출력 가능 시점)

ALL 3 reviews:
- (a) ≥ 8 distinct findings each
- (b) ≥ 3 Critical/High findings each
- (c) 모든 finding은 구체 시나리오/숫자 (일반 원칙 X)
- (d) Recommendation은 design doc의 특정 section/agent를 명시 변경 제안
- (e) Cross-cutting concerns 비어있지 않음

## 반복 전략

매 iteration:
1. 세 review 파일 모두 읽기 (있다면)
2. 가장 약한 것을 찾기 (짧음 / Critical-High 부족 / 추상적)
3. 그것을 deepen (finding 추가 + 구체화)
4. spec/plan에 ambiguity 있으면 'Design ambiguity: ...' 표기 (추측 금지)
