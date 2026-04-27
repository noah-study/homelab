# Task 15.1: Grafana Cloud 계정 + Stack + 토큰

**Phase**: 15 (Grafana Cloud + Alloy + Redaction + Drop Rules)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 15.1' 섹션

---


- [ ] **Step 1: Grafana Cloud 가입 (UI)**

https://grafana.com/auth/sign-up — 무료, 신용카드 불필요. Stack 이름: `noah`.

- [ ] **Step 2: API Access Policy + Token 생성**

Grafana Cloud Portal → Stack noah → Access Policies → Create:
- Name: `alloy-write`
- Realms: stack-noah
- Scopes: `metrics:write`, `logs:write`, `traces:write`, `profiles:write`
- Create Token → 1Password 저장 (`grafana-cloud-alloy-token`)

엔드포인트도 기록:
- Prometheus remote write URL
- Loki push URL
- Tempo push URL
- Pyroscope push URL
- 사용자 ID (instance-id)
