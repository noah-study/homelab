# Phase 9 — ARC (Actions Runner Controller)

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: GitHub Actions self-hosted runner를 k8s pod로 ephemeral 생성. job마다 새 pod, 끝나면 폐기.

**참조**: 보안 리뷰 F1 (runner 침해), F8 (_work persistence)


## Tasks

- [Task 9.1: ARC 컨트롤러 + Runner Scale Set GitHub App](./task-9.1.md)
- [Task 9.2: Runner Scale Set 정의](./task-9.2.md)
