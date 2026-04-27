# Phase 3 — ArgoCD Bootstrap (자기관리)

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: ArgoCD 설치 후 root.yaml의 App-of-Apps로 ArgoCD가 자기 자신과 platform/, argocd/ 하위 모든 리소스를 관리하도록 한다.

**참조**: 흡수한 원칙 1, 2 (Bootstrap ↔ Runtime 분리, GitOps self-management)


## Tasks

- [Task 3.1: ArgoCD 초기 설치 매니페스트 작성](./task-3.1.md)
- [Task 3.2: ArgoCD 부트스트랩 적용](./task-3.2.md)
- [Task 3.3: App-of-Apps root + cluster-resources](./task-3.3.md)
