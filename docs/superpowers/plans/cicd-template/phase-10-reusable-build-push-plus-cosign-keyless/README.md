# Phase 10 — Reusable Build-Push + Cosign Keyless

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: 서비스 레포에서 `uses: noah-study/homelab/.github/workflows/reusable-build-push.yml@main` 3줄 호출만으로 테스트 + 멀티태그 빌드 + GHCR push + Cosign 서명 + digest output.

**참조**: 보안 리뷰 F3 (digest 출력), F5 (action SHA pin), F7 (OIDC subject)


## Tasks

- [Task 10.1: reusable workflow 작성](./task-10.1.md)
