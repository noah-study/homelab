# Phase 8 — GitHub App + Image Updater (digest pin)

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: Image Updater가 path-restricted GitHub App으로 infra repo에 PR을 생성, dev kustomization을 digest로 갱신.

**참조**: 보안 리뷰 F2 (Image Updater 광역 write 권한), F3 (digest pin)


## Tasks

- [Task 8.1: GitHub App 생성 (수동 UI)](./task-8.1.md)
- [Task 8.2: argocd-image-updater 설치](./task-8.2.md)
- [Task 8.3: write-back용 GitHub branch protection + auto-merge 설정](./task-8.3.md)
