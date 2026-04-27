# Phase 4 — Sealed Secrets + 마스터키 백업

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: git에 시크릿을 안전하게 커밋할 수 있도록 sealed-secrets 컨트롤러 설치, 마스터키를 1Password + 오프라인 USB에 이중화.

**참조**: 보안 리뷰 F4


## Tasks

- [Task 4.1: sealed-secrets 컨트롤러 설치](./task-4.1.md)
- [Task 4.2: 마스터키 백업 자동화 + 검증](./task-4.2.md)
- [Task 4.3: seal.sh wrapper + bats 테스트](./task-4.3.md)
