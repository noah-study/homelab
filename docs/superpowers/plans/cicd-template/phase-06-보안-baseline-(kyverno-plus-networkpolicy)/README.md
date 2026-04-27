# Phase 6 — 보안 Baseline (Kyverno + NetworkPolicy)

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: PSA exception 감사 (Kyverno), Hubble UI 외부 노출 차단 (NetworkPolicy), namespace 간 기본 차단.

**참조**: 보안 리뷰 F12 (PSA erosion), F17 (Hubble UI 노출)


## Tasks

- [Task 6.1: Kyverno 설치 + PSA exception audit policy](./task-6.1.md)
- [Task 6.2: NetworkPolicy default-deny + Hubble UI 차단](./task-6.2.md)
