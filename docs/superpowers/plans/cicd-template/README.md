# CI/CD Template Implementation Plan — Index

원본 단일 plan: `../2026-04-26-infra-cicd-template-plan.md`
이 디렉토리는 task별 split 본 (homelab-builder가 1 task당 1 파일 Read).

## Phases

### Phase 1 — Foundation
- [Phase README](./phase-01-foundation/README.md)
  - [Task 1.1: Repo 초기화 + 디렉토리 골격](./phase-01-foundation/task-1.1.md)
  - [Task 1.2: Cloudflare 계정 보안 강화 (수동 UI)](./phase-01-foundation/task-1.2.md)
  - [Task 1.3: Mac mini 호스트 준비](./phase-01-foundation/task-1.3.md)

### Phase 2 — k3s + Cilium
- [Phase README](./phase-02-k3s-plus-cilium/README.md)
  - [Task 2.1: k3s 설치 (flannel/traefik 비활성화)](./phase-02-k3s-plus-cilium/task-2.1.md)
  - [Task 2.2: Cilium CNI 설치](./phase-02-k3s-plus-cilium/task-2.2.md)

### Phase 3 — ArgoCD Bootstrap (자기관리)
- [Phase README](./phase-03-argocd-bootstrap-(자기관리)/README.md)
  - [Task 3.1: ArgoCD 초기 설치 매니페스트 작성](./phase-03-argocd-bootstrap-(자기관리)/task-3.1.md)
  - [Task 3.2: ArgoCD 부트스트랩 적용](./phase-03-argocd-bootstrap-(자기관리)/task-3.2.md)
  - [Task 3.3: App-of-Apps root + cluster-resources](./phase-03-argocd-bootstrap-(자기관리)/task-3.3.md)

### Phase 4 — Sealed Secrets + 마스터키 백업
- [Phase README](./phase-04-sealed-secrets-plus-마스터키-백업/README.md)
  - [Task 4.1: sealed-secrets 컨트롤러 설치](./phase-04-sealed-secrets-plus-마스터키-백업/task-4.1.md)
  - [Task 4.2: 마스터키 백업 자동화 + 검증](./phase-04-sealed-secrets-plus-마스터키-백업/task-4.2.md)
  - [Task 4.3: seal.sh wrapper + bats 테스트](./phase-04-sealed-secrets-plus-마스터키-백업/task-4.3.md)

### Phase 5 — Cloudflare Tunnel + cert-manager + ingress-nginx
- [Phase README](./phase-05-cloudflare-tunnel-plus-cert-manager-plus-ingress-n/README.md)
  - [Task 5.1: cert-manager 설치 + Cloudflare DNS01 ClusterIssuer](./phase-05-cloudflare-tunnel-plus-cert-manager-plus-ingress-n/task-5.1.md)
  - [Task 5.2: ingress-nginx 설치](./phase-05-cloudflare-tunnel-plus-cert-manager-plus-ingress-n/task-5.2.md)
  - [Task 5.3: cloudflared (Tunnel) 배포](./phase-05-cloudflare-tunnel-plus-cert-manager-plus-ingress-n/task-5.3.md)
  - [Task 5.4: ArgoCD Ingress 추가 (첫 외부 노출)](./phase-05-cloudflare-tunnel-plus-cert-manager-plus-ingress-n/task-5.4.md)

### Phase 6 — 보안 Baseline (Kyverno + NetworkPolicy)
- [Phase README](./phase-06-보안-baseline-(kyverno-plus-networkpolicy)/README.md)
  - [Task 6.1: Kyverno 설치 + PSA exception audit policy](./phase-06-보안-baseline-(kyverno-plus-networkpolicy)/task-6.1.md)
  - [Task 6.2: NetworkPolicy default-deny + Hubble UI 차단](./phase-06-보안-baseline-(kyverno-plus-networkpolicy)/task-6.2.md)

### Phase 7 — AppProjects + ApplicationSets
- [Phase README](./phase-07-appprojects-plus-applicationsets/README.md)
  - [Task 7.1: AppProject 정의](./phase-07-appprojects-plus-applicationsets/task-7.1.md)
  - [Task 7.2: ApplicationSet for platform/](./phase-07-appprojects-plus-applicationsets/task-7.2.md)
  - [Task 7.3: ApplicationSet for apps/ (dev + prod)](./phase-07-appprojects-plus-applicationsets/task-7.3.md)

### Phase 8 — GitHub App + Image Updater (digest pin)
- [Phase README](./phase-08-github-app-plus-image-updater-(digest-pin)/README.md)
  - [Task 8.1: GitHub App 생성 (수동 UI)](./phase-08-github-app-plus-image-updater-(digest-pin)/task-8.1.md)
  - [Task 8.2: argocd-image-updater 설치](./phase-08-github-app-plus-image-updater-(digest-pin)/task-8.2.md)
  - [Task 8.3: write-back용 GitHub branch protection + auto-merge 설정](./phase-08-github-app-plus-image-updater-(digest-pin)/task-8.3.md)

### Phase 9 — ARC (Actions Runner Controller)
- [Phase README](./phase-09-arc-(actions-runner-controller)/README.md)
  - [Task 9.1: ARC 컨트롤러 + Runner Scale Set GitHub App](./phase-09-arc-(actions-runner-controller)/task-9.1.md)
  - [Task 9.2: Runner Scale Set 정의](./phase-09-arc-(actions-runner-controller)/task-9.2.md)

### Phase 10 — Reusable Build-Push + Cosign Keyless
- [Phase README](./phase-10-reusable-build-push-plus-cosign-keyless/README.md)
  - [Task 10.1: reusable workflow 작성](./phase-10-reusable-build-push-plus-cosign-keyless/task-10.1.md)

### Phase 11 — apps/_base + 새 앱 onboarding 스크립트
- [Phase README](./phase-11-apps-_base-plus-새-앱-onboarding-스크립트/README.md)
  - [Task 11.1: apps/_base 표준 매니페스트](./phase-11-apps-_base-plus-새-앱-onboarding-스크립트/task-11.1.md)
  - [Task 11.2: scripts/new-app.sh — atomic onboarding](./phase-11-apps-_base-plus-새-앱-onboarding-스크립트/task-11.2.md)

### Phase 12 — First Sample App Smoke Test
- [Phase README](./phase-12-first-sample-app-smoke-test/README.md)
  - [Task 12.1: sample-app 코드 레포 + 첫 배포](./phase-12-first-sample-app-smoke-test/task-12.1.md)

### Phase 13 — Promote Workflow + GHCR Cleanup + Verify-Renders
- [Phase README](./phase-13-promote-workflow-plus-ghcr-cleanup-plus-verify-ren/README.md)
  - [Task 13.1: promote workflow](./phase-13-promote-workflow-plus-ghcr-cleanup-plus-verify-ren/task-13.1.md)
  - [Task 13.2: GHCR cleanup workflow](./phase-13-promote-workflow-plus-ghcr-cleanup-plus-verify-ren/task-13.2.md)
  - [Task 13.3: verify-renders workflow](./phase-13-promote-workflow-plus-ghcr-cleanup-plus-verify-ren/task-13.3.md)

### Phase 14 — Trivy-Operator + Alert Routing
- [Phase README](./phase-14-trivy-operator-plus-alert-routing/README.md)
  - [Task 14.1: trivy-operator 설치](./phase-14-trivy-operator-plus-alert-routing/task-14.1.md)
  - [Task 14.2: Discord 알림 cron (간이)](./phase-14-trivy-operator-plus-alert-routing/task-14.2.md)

### Phase 15 — Grafana Cloud + Alloy + Redaction + Drop Rules
- [Phase README](./phase-15-grafana-cloud-plus-alloy-plus-redaction-plus-drop-/README.md)
  - [Task 15.1: Grafana Cloud 계정 + Stack + 토큰](./phase-15-grafana-cloud-plus-alloy-plus-redaction-plus-drop-/task-15.1.md)
  - [Task 15.2: Alloy 설치 + redaction + drop rules](./phase-15-grafana-cloud-plus-alloy-plus-redaction-plus-drop-/task-15.2.md)

### Phase 16 — Renovate + CODEOWNERS + Branch Protection
- [Phase README](./phase-16-renovate-plus-codeowners-plus-branch-protection/README.md)
  - [Task 16.1: Renovate 룰](./phase-16-renovate-plus-codeowners-plus-branch-protection/task-16.1.md)
  - [Task 16.2: CODEOWNERS + Branch Protection](./phase-16-renovate-plus-codeowners-plus-branch-protection/task-16.2.md)

### Phase 17 — Documentation
- [Phase README](./phase-17-documentation/README.md)
  - [Task 17.1: docs/onboarding.md](./phase-17-documentation/task-17.1.md)
  - [Task 17.2: docs/runbook.md](./phase-17-documentation/task-17.2.md)
  - [Task 17.3: docs/scaling-playbook.md](./phase-17-documentation/task-17.3.md)

### Phase 18 — Acceptance Tests
- [Phase README](./phase-18-acceptance-tests/README.md)
  - [Task 18.1: 자동 검증 스크립트](./phase-18-acceptance-tests/task-18.1.md)

