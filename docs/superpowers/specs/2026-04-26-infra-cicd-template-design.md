# Infra CI/CD Template Design

**Date**: 2026-04-26
**Status**: Spec (post-review v2)
**Author**: noah
**Goal**: Mac mini 셀프호스팅 환경에서 GitHub Actions + ArgoCD 기반의 모든 인프라 공통 배포 파이프라인 템플릿. 새 서비스 추가 시 5~10분 안에 dev 자동 배포, prod는 명시적 promotion. **학습과 framework 소유**가 우선 순위 — 부트스트래퍼(argocd-autopilot, Kubefirst)를 사용하지 않고 직접 구축하되, argocd-autopilot의 검증된 디렉토리·명명·분리 원칙을 컨벤션으로 흡수.

---

## 1. 북극성 목표

> **새 서비스 = 코드 + Dockerfile + workflow 3줄 + 매니페스트 overlay 디렉토리만 있으면 5~10분 안에 dev 자동 배포. 운영 표준에서 벗어나지 않는 한 사람의 개입 0. Prod는 명시적 promotion으로 안전벨트.**

부수 목표:
- **소유**: 모든 정책·컨벤션을 본인이 결정하고 코드로 표현
- **학습**: 모든 컴포넌트의 동작 원리를 디버깅 가능한 수준으로 이해
- **확장 가능**: 1 → 30 서비스로 늘어도 같은 컨벤션 유지

---

## 2. 흡수한 외부 설계 원칙 (argocd-autopilot 정신)

직접 구축하되 다음 원칙은 그대로 따른다:

1. **Bootstrap ↔ Runtime 분리** — bootstrap 매니페스트는 1회 적용 후 거의 변경 없음. Runtime(apps)은 일상 변경.
2. **GitOps self-management** — bootstrap 후 ArgoCD는 자기 자신을 관리 (App-of-Apps 루트)
3. **Project-scoped Apps** — 모든 Application은 AppProject에 속해 RBAC 격리
4. **Atomic onboarding** — 새 서비스 추가는 단일 commit으로 namespace + Application + secrets 한꺼번에
5. **Convention over configuration** — 디렉토리 컨벤션이 곧 자동화 (사람이 ArgoCD UI나 yaml을 손으로 등록하지 않음)
6. **Cluster-first 디렉토리** — overlay에 cluster를 명시 → 미래 multi-cluster 호환
7. **Kustomize-first** — 자체 앱은 Kustomize, 외부 Helm chart는 Kustomize의 HelmChart generator로 wrapping (단일 템플릿 도구)
8. **Explicit references** — Application source(repoURL/path/targetRevision)는 ApplicationSet template에 명시, 디폴트로 숨기지 않음

---

## 3. 인프라 스택 (Day-1)

### 3.1 호스트
- Mac mini (Apple Silicon, **32GB 권장** — 16GB는 25~30 서비스 한계, 32GB는 50~60)
- OrbStack Linux VM 위에 모든 컴포넌트
- macOS 호스트 보안: Remote Login 비활성, FileVault 활성, OrbStack 자동 업데이트
- GitHub Actions Self-hosted Runner는 **별도 ephemeral 컨테이너** (다음 항목 4.1 참조)

### 3.2 클러스터 컴포넌트 (`platform/`, Helm via Kustomize HelmChart)
| 컴포넌트 | 역할 |
|---|---|
| k3s (single-node, sqlite 시작) | Kubernetes |
| Cilium | CNI — NetworkPolicy + Hubble (UI는 ingress 없이 port-forward만) |
| ingress-nginx | 단일 ingress class (`nginx`) |
| cert-manager | Let's Encrypt 와일드카드 (DNS01 via Cloudflare) |
| cloudflared | Cloudflare Tunnel (NAT 우회, DDoS/WAF) |
| sealed-secrets | git에 시크릿 안전 커밋 |
| argo-cd | GitOps reconciler (App-of-Apps 루트가 자기 자신 관리) |
| argocd-image-updater | dev 환경 이미지 자동 갱신 (path-restricted GitHub App, write-back: git-pr) |
| grafana-alloy | OTel 수집 → Grafana Cloud Free, redaction processor 포함 |
| trivy-operator | 이미지/매니페스트 취약점 스캔, finding → Alloy → Grafana alert |
| actions-runner-controller (ARC) | self-hosted runner를 k8s pod로 ephemeral 생성 |

### 3.3 외부 서비스
- **GitHub** — 코드, Actions, GHCR, Environments, GitHub App (Image Updater 토큰)
- **Cloudflare** — DNS, Tunnel, 외부 TLS 종료
  - 계정에 hardware 2FA (FIDO2) 강제
  - DNSSEC 활성화
  - CAA 레코드: `0 issue "letsencrypt.org"` + `0 iodef "mailto:..."`
  - cert-manager용 API 토큰: DNS edit 권한 + 특정 zone 한정
- **Grafana Cloud Free** — Logs (50GB/월/14d) / Metrics (10K series/14d) / Traces (50GB/월/14d) / Profiles (50GB/월/14d) / Synthetics 100K/월 / k6 500 VU-h/월 / 3 users
- **Sigstore (공개)** — Cosign keyless 서명 (Fulcio + Rekor 투명성 로그)
- **Renovate** (Mend 호스팅 GitHub App) — 의존성 자동 PR
- **1Password** — Sealed Secrets 마스터키 백업 (오프라인 암호화 USB 이중화)

### 3.4 보안·품질 표준 (Day-1)

#### 3.4.1 Self-hosted runner 격리
- **ARC (actions-runner-controller)** k8s mode — runner는 pod, job마다 ephemeral 생성
- 호스트 docker socket 마운트 금지, BuildKit rootless 사용
- 외부 contributor 받는 service repo는 cloud runner로 강제 (self-hosted는 trusted 레포 한정)
- runner pod에 outbound network policy (GHCR/Sigstore/registry.npmjs/pypi 등 allowlist만)

#### 3.4.2 Image 공급망
- **Cosign keyless 서명**: 모든 GHCR push에 자동 적용
  - OIDC subject pin 정책 (검증 시): `--certificate-identity 'https://github.com/noah-study/homelab/.github/workflows/reusable-build-push.yml@refs/heads/main'`
  - infra 레포 main 브랜치 보호 (직접 push 금지, PR만)
- **Cosign 검증 (Sigstore Policy Controller)**: deferred (~2026-06)
- **Day-1 interim**: Image Updater의 `update-strategy: digest` — kustomization에 tag 대신 digest 핀 → tag 교체 공격 차단
- GHCR retention 정책: cleanup script `scripts/ghcr-cleanup.sh` 매주 cron, `keep_latest: 10` + `sha-prod-*` 영구 보관
- 가능하면 service repo는 **public** (GHCR storage/transfer 무제한 무료)

#### 3.4.3 시크릿 (Sealed Secrets)
- namespace-scoped 봉인 (`--scope namespace-wide`)
- **마스터키 백업 이중화**: 1Password + 오프라인 암호화 USB
- **마스터키 회전**: 6개월마다 — 회전 + 재봉인 스크립트 `scripts/rotate-sealed-secrets.sh` Day-1 작성
- 1Password 계정에 hardware 2FA 강제

#### 3.4.4 ArgoCD Image Updater 권한 최소화
- **GitHub App** 사용 (PAT 금지) — 권한: `contents: write` + `pull_requests: write`
- write-back 모드: `git-pr` (commit이 아니라 PR 생성)
- branch protection: `apps/*/overlays/*/dev/kustomization.yaml`만 자동 머지 허용
- argocd-image-updater 컨테이너는 Renovate auto-merge 대상 제외 (`platform-critical` 라벨)

#### 3.4.5 Renovate 안전 정책
- Actions를 **commit SHA로 핀** (`pinDigests: true`)
- Auto-merge 대상은 official org allowlist: `actions/*`, `docker/*`, `sigstore/*`
- `vulnerabilityAlerts.enabled: true` — security advisory는 24시간 내 자동 머지
- ingress-nginx, cert-manager, argo-cd: `platform-critical` (수동 머지) + 보안 advisory만 자동

#### 3.4.6 PodSecurityStandards & NetworkPolicy
- 모든 namespace에 `pod-security.kubernetes.io/enforce: restricted`
- `_base/deployment.yaml`에 표준 securityContext 박아 자체 서비스 자동 준수
- exception PR은 본인 리뷰 + 이유 주석 필수
- Cilium NetworkPolicy: namespace 간 기본 차단, ingress 명시 허용
- Polaris 또는 kyverno로 cluster-wide audit (예외 추가 시 알림)

#### 3.4.7 옵저버빌리티 보안
- Alloy에 **redaction processor**: `Authorization`, `Cookie`, `apikey`, `password`, `token` 패턴 자동 제거
- gitleaks를 reusable workflow에 한 단계 추가 (시크릿 누출 방지)

#### 3.4.8 GitHub Environments `production` 게이트
- `infra` 레포에만 적용 — required reviewer (본인) + **wait timer 5분** + Discord webhook 알림

#### 3.4.9 trivy-operator finding 라우팅
- finding metric → Alloy → Grafana Cloud
- 알림 룰: Critical/High → Discord push, Medium/Low → 주간 다이제스트
- SLA: Critical 7일, High 30일 (위반 시 admission webhook 차단은 deferred)

### 3.5 Day-1 미적용 / 나중 도입 (interim mitigation 명시)

| 항목 | 도입 시점 | Interim mitigation |
|---|---|---|
| PR Preview 환경 (ApplicationSet PullRequest generator) | 동시 작업 충돌 체감 시 | hook(태그 컨벤션, overlay 컨벤션)은 미리 깔아둠 |
| 백업/DR (Velero, Backblaze B2) | 운영 시작 후 30일 내 재논의 | SealedSecrets 마스터키 1Password+USB. infra repo는 GitHub 자체가 백업 |
| Sigstore Policy Controller (서명 검증) | ~2026-06 | digest pin (위 3.4.2) |
| Falco 런타임 보안 | 보안 사고 발생 시 | Cilium NetworkPolicy로 lateral movement 제한 |
| Kargo promotion 자동화 | 환경 3개 이상 도달 시 | workflow_dispatch promotion |
| Mac mini 32GB → 64GB 또는 2nd node | 30 서비스 도달 시 | 단일 노드 모니터링 (Section 11) |
| ArgoCD controller sharding | 200 Application 도달 시 | reconcile 간격 조정 |
| ApplicationSet/ArgoCD webhook | 50 서비스 도달 시 | polling 간격 5분 |

---

## 4. 레포 구조 (autopilot 컨벤션 흡수)

```
noah-study/homelab/                              ← 모노레포 (단일 진실의 소스)
│
├── .github/
│   ├── workflows/
│   │   ├── reusable-build-push.yml      # 서비스 레포가 호출
│   │   ├── promote.yml                  # workflow_dispatch (prod PR 생성)
│   │   ├── ghcr-cleanup.yml             # cron weekly
│   │   ├── platform-deploy.yml          # platform/ 변경 알림
│   │   └── verify-renders.yml           # PR 시 kustomize render 검증
│   └── CODEOWNERS                        # platform/, argocd/ → 본인 강제 리뷰
│
├── renovate.json
│
├── bootstrap/                            ★ argocd-autopilot 컨벤션
│   ├── README.md                         # 1회 부트스트랩 절차
│   ├── argo-cd/                          # ArgoCD 자기 자신 관리
│   │   ├── kustomization.yaml
│   │   └── values.yaml                   # ArgoCD Helm values
│   ├── cluster-resources/                # 클러스터 와이드 (PSA, namespaces, CRDs)
│   │   ├── kustomization.yaml
│   │   ├── namespaces.yaml
│   │   ├── psa-restricted.yaml
│   │   └── caa-cluster-issuer.yaml
│   └── root.yaml                         # App-of-Apps 루트 (모든 ApplicationSet 트리거)
│
├── argocd/
│   ├── projects/                         # AppProject = RBAC 경계
│   │   ├── platform.yaml                 # platform 컴포넌트용
│   │   └── apps.yaml                     # 서비스 앱용
│   └── applicationsets/
│       ├── platform.yaml                 # platform/* 자동 등록
│       ├── apps-dev.yaml                 # apps/*/overlays/macmini/dev 자동, Image Updater 활성
│       └── apps-prod.yaml                # apps/*/overlays/macmini/prod 자동, Image Updater 비활성
│
├── platform/                             # 플랫폼 컴포넌트 (Kustomize + HelmChart wrapper)
│   ├── cilium/
│   ├── ingress-nginx/
│   ├── cert-manager/
│   │   ├── kustomization.yaml
│   │   ├── values.yaml
│   │   └── cluster-issuer.yaml           # LE + Cloudflare DNS01
│   ├── cloudflared/
│   │   ├── values.yaml
│   │   └── tunnel-token.sealed.yaml
│   ├── sealed-secrets/
│   ├── argocd-image-updater/
│   │   ├── values.yaml
│   │   └── github-app-secret.sealed.yaml
│   ├── grafana-alloy/
│   │   ├── values.yaml                   # OTel + redaction processor + metric drop 룰
│   │   └── grafana-cloud-token.sealed.yaml
│   ├── trivy-operator/
│   ├── actions-runner-controller/
│   └── kyverno/                          # PSA exception audit
│
├── apps/                                 ★ 서비스 매니페스트
│   ├── _base/                            # 전역 default (모든 서비스 상속)
│   │   ├── deployment.yaml               # OTel env, securityContext, probes 표준
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   ├── servicemonitor.yaml
│   │   └── kustomization.yaml
│   │
│   └── <service>/                        # 서비스마다 추가
│       ├── config.yaml                   ★ autopilot 정신: app metadata
│       │                                 # service, owner, source repo, project,
│       │                                 # branches policy (env→branch 매핑)
│       ├── base/                         # 서비스별 추가/override (옵션)
│       │   └── kustomization.yaml        # _base 참조 + 서비스 특수 리소스
│       └── overlays/
│           └── macmini/                  ★ cluster-first (미래 호환)
│               ├── dev/
│               │   ├── namespace.yaml
│               │   ├── kustomization.yaml
│               │   └── secrets.sealed.yaml
│               └── prod/
│                   ├── namespace.yaml
│                   ├── kustomization.yaml
│                   └── secrets.sealed.yaml
│
├── scripts/
│   ├── new-app.sh                        ★ atomic onboarding (autopilot CLI 정신)
│   ├── seal.sh                           # kubeseal wrapper
│   ├── rotate-sealed-secrets.sh          # 6개월 회전
│   ├── ghcr-cleanup.sh                   # GHCR retention
│   └── promote.sh                        # promote workflow의 코어 로직
│
└── docs/
    ├── onboarding.md                     # 새 서비스 추가 5분 가이드
    ├── runbook.md                        # 사고 대응 (계정 침해, 마스터키 유출, runner 침해)
    └── superpowers/
        ├── specs/2026-04-26-infra-cicd-template-design.md
        └── reviews/{security,scalability,free-economics}.md

noah-study/<service>/                           ← 서비스 코드 레포 (N개)
├── src/
├── Dockerfile
└── .github/workflows/ci.yml              # reusable workflow 3줄 호출
```

---

## 5. 핵심 컨벤션

### 5.1 이미지 태그 (3종 동시 push)
- `sha-<short>` 불변, 추적용
- `<branch-slug>-<short>` 불변, 환경 라우팅 기반 ★
- `<branch-slug>-latest` 가변, 디버깅용
- **Image Updater는 tag 대신 digest 사용** — kustomization에 `digest: sha256:...` 기록 (tag 교체 공격 차단)

### 5.2 환경 정책

| 환경 | 트리거 브랜치 | 배포 | namespace | 호스트 |
|---|---|---|---|---|
| dev | `develop`, `feat/**`, `fix/**` | Image Updater 자동 (regex `^(develop\|feat-.*\|fix-.*)-[0-9a-f]{7,}$`) | `<svc>-dev` | `<svc>.dev.noah.dev` |
| prod | `main` (빌드만) | 수동 promotion (workflow_dispatch → PR → 머지) | `<svc>-prod` | `<svc>.noah.dev` |

### 5.3 Promotion 게이트 (3단)
1. `workflow_dispatch` 실행 — `production` GH Environment 승인 + wait timer 5분
2. 자동 생성된 PR 리뷰 (Discord 알림으로 본인 재인지)
3. main 머지 (branch protection + required reviewer)

### 5.4 시크릿
- Sealed Secrets, namespace별 봉인
- 마스터키 백업: 1Password + 오프라인 USB
- 6개월 회전 + 재봉인 스크립트

### 5.5 옵저버빌리티
- 모든 컨테이너에 `_base`가 OTel 환경변수 자동 주입
- Alloy redaction processor (시크릿 헤더/패턴 제거)
- Alloy → Grafana Cloud Free
- **카디널리티 SLO**: 5 서비스 = ≤4K series, 10 = ≤7K, 20 = ≤9.5K
- **표준 metric drop 룰** (Day-1, Alloy values):
  - cAdvisor: `container_blkio_*`, `container_tasks_*`, `container_fs_inodes_*`, `container_memory_failures_*` drop
  - 라벨 `id`, `container_id`, `pod_uid` drop
  - histogram bucket 5개로 (le: 0.1, 0.5, 1, 5, 10)

### 5.6 TLS 전략 (하이브리드)
- 사용자 → CF: Cloudflare Universal SSL
- CF → Mac mini: Cloudflare Tunnel (origin TLS)
- 클러스터 내부: cert-manager + Let's Encrypt 와일드카드 (`*.noah.dev`, `*.dev.noah.dev`) — 모든 서비스 공유

### 5.7 GitHub Actions 컨벤션
- Self-hosted runner = ARC k8s mode + ephemeral pod
- PR (외부 contributor 가능)에는 cloud runner 강제
- 모든 action을 commit SHA로 핀 (Renovate가 자동 갱신 PR)

---

## 6. 새 서비스 onboarding (atomic)

`scripts/new-app.sh service-b` 한 명령으로:

1. `noah-study/service-b` 레포 생성 (CLI: `gh repo create`)
2. Dockerfile 템플릿 복사
3. `.github/workflows/ci.yml` 생성 (3줄짜리 reusable 호출)
4. `noah-study/homelab/apps/service-b/` 디렉토리 생성:
   - `config.yaml` (service metadata)
   - `overlays/macmini/dev/` (namespace, kustomization, empty secrets.sealed)
   - `overlays/macmini/prod/` (동일)
5. PR 생성 + 본인 리뷰 + 머지

머지 후 자동 진행:
```
서비스 push → ARC pod에서 빌드 → GHCR + Cosign 서명 (digest 추출)
       ↓
Image Updater (2분 polling) → infra repo PR 생성 (digest 갱신)
       ↓
Auto-merge (path-restricted dev kustomization만) → ArgoCD sync
       ↓
ApplicationSet apps-dev → 새 디렉토리 감지 → Application 자동 등록
       ↓
ArgoCD: namespace + pod + ingress 생성 → cert-manager 와일드카드 cert 공유
       ↓
Alloy: OTel/메트릭 자동 수집 (redaction 적용)
       ↓
https://service-b.dev.noah.dev 라이브 (~5-10분)

[검증 후]
workflow_dispatch promote.yml → 5분 wait → PR 생성 → 본인 리뷰 → 머지
       ↓
ArgoCD apps-prod sync → https://service-b.noah.dev 라이브
```

총 사용자 작업: 명령 1회 + PR 리뷰 1회 (~3분).

---

## 7. 의식적 트레이드오프

1. **단일 노드 SPOF** — Mac mini 다운 = 다운타임. 99.5% SLA 한계. 비핵심 워크로드만 적합.
2. **메트릭 카디널리티 통제 부담** — Free 10K series 안에서 운영하려면 Day-1 drop 룰 필수.
3. **dev 환경 충돌 가능** — feat/* 동시 push 시 마지막 빌드가 dev 덮음. PR Preview는 deferred.
4. **Image Updater 폴링 지연** (~2~5분) — 즉시 배포 아님.
5. **Coolify 같은 PaaS UX 포기** — k8s/GitOps 학습 + framework 소유를 위해 의식적 선택.
6. **macOS → Linux VM 4계층** — AMD64 이미지 필요 시 QEMU 에뮬레이션 비용. ARM64 우선 정책.
7. **백업 DR Day-1 미구현** — 운영 시작 후 30일 내 재논의. 그 사이 데이터 손실 위험 인정.
8. **부트스트래퍼(autopilot/Kubefirst) 미사용** — 소유권/학습 우선. 디렉토리 컨벤션은 흡수, 도구는 미사용.

---

## 8. Threat Model & Trust Boundaries

### 8.1 신뢰 경계
```
[GitHub (외부 SaaS, 신뢰)]
  ├─ infra repo (단일 진실 소스, branch protection)
  ├─ service repos (덜 신뢰 — collaborator 가능)
  └─ GHCR (이미지 저장)
       ↓ (digest pin + Cosign 서명)
[Cloudflare (외부 SaaS, 2FA + DNSSEC)]
  ├─ DNS, Tunnel, Universal SSL
       ↓ (cloudflared outbound)
[Mac mini (호스트, FileVault)]
  └─ OrbStack Linux VM (격리)
       └─ k3s 클러스터
            ├─ ArgoCD (App-of-Apps 자기관리)
            ├─ Image Updater (path-scoped GitHub App)
            ├─ ARC runner pods (ephemeral, outbound allowlist)
            └─ apps namespaces (PSA restricted, NetworkPolicy)
```

### 8.2 사고 대응 플레이북 (`docs/runbook.md`)
- **GitHub 계정 침해**: 토큰 회수, GitHub App 키 회전, branch protection 검증, audit log 분석
- **Cloudflare 계정 침해**: API 토큰 즉시 무효, DNSSEC 검증, CT 로그 모니터로 사기 발급 인증서 확인
- **SealedSecrets 마스터키 유출**: 즉시 회전 + 모든 SealedSecret 재봉인, 외부 시크릿(DB pwd, API key) 전수 회전
- **Self-hosted runner pod 침해**: ARC pod 강제 종료, 격리, allowlist 위반 outbound 차단 확인
- **Cosign 서명 인프라 다운**: workflow `continue-on-error: true` + Discord 알림, 임시로 unsigned 빌드 허용 (단, ArgoCD verify 활성 상태면 반려)

---

## 9. 성공 기준

### 9.1 기능
- [ ] 새 서비스 추가 시 사용자 작업 시간 ≤ 5분 (script + PR 리뷰)
- [ ] dev 환경 라이브까지 push로부터 ≤ 10분
- [ ] prod promotion 절차가 3단 게이트 모두 통과
- [ ] Renovate 주간 PR ≤ 10개 + auto-merge로 본인 리뷰 부담 최소

### 9.2 보안
- [ ] Service repo의 임의 PR이 GHCR push 토큰을 추출할 수 없다 (시뮬레이션 테스트)
- [ ] Image Updater 토큰이 platform/ 디렉토리를 수정할 수 없다 (path 제한 검증)
- [ ] CF 계정 2FA + DNSSEC + CAA 활성 (체크리스트)
- [ ] Cosign 서명 모든 GHCR 이미지에 부착 + Rekor 조회 검증 (자동 테스트)
- [ ] kustomization에 image digest pin 적용 (tag-only 금지 룰)

### 9.3 확장성
- [ ] Mac mini RAM 사용률 75% 이하
- [ ] GitHub API rate limit 사용량 60% 이하
- [ ] ArgoCD reconcile lag P99 < 60s
- [ ] Self-hosted runner 큐 대기시간 P95 < 5분
- [ ] sqlite WAL 크기 모니터링 (kine /metrics)

### 9.4 무료 운영
- [ ] Grafana Cloud series ≤ 8K (80%) at 5 서비스
- [ ] GHCR storage ≤ 500MB at 5 서비스 (private 가정)
- [ ] 모든 free tier 80% 도달 시 자동 알림 (Discord)
- [ ] 월 비용 보고서 자동화 (1회/월)

---

## 10. 규모별 Scaling Playbook

| 규모 | 트리거 신호 | 액션 | 예상 작업 |
|---|---|---|---|
| 1~10 서비스 | 본 설계 그대로 | — | — |
| 10~20 서비스 | Renovate PR 주 30개 초과 | grouping 강화, runner 2개로 | 30분 |
| 20~30 서비스 | Mac mini RAM 70% 도달 | 32GB 업그레이드 또는 ARC 자동 스케일 | 1일 (HW 교체) |
| 30~50 서비스 | sqlite latency P99 > 200ms / GitHub API 사용량 70% | k3s 백엔드 외부 Postgres, ApplicationSet/ArgoCD webhook 전환 | 반나절 |
| 50~100 서비스 | RAM 80% 또는 ArgoCD reconcile lag > 2분 | 2nd Mac mini node 추가, ArgoCD controller sharding, PR Preview 도입 | 1~2일 |
| 100+ | 단일 머신 한계 분명 | k3s 멀티노드 또는 vanilla k8s/managed로 마이그레이션 | 1주~ |

원칙: **분명히 깨질 때까지 스택을 바꾸지 말 것** (premature scaling 방지).

---

## 11. Cost Monitoring & Upgrade Triggers

### 11.1 무료 티어 모니터링 (자동)
- Grafana Cloud series count (Mimir self-stat) → Grafana 알림 룰 (8K = 80%)
- GHCR storage (GitHub API exporter, 매일 cron) → 400MB = 80%
- CF Workers req/day (CF API)
- GitHub API rate limit 잔량 (X-RateLimit-Remaining 헤더 메트릭화)

### 11.2 유료 전환 우선순위
1. **Grafana Cloud Pro** ($19/월) — 가장 빨리 깨짐 (15-20 서비스)
2. **Mac mini 32GB → 64GB** (1회 ~$300) — 메모리 천장
3. **1Password** ($3/월) — 사용 중 아니면
4. **GHCR Pro** ($5/월) 또는 public 전환 — 5+ 서비스 시 검토
5. **CF Pro** ($20/월) — 20+ 서비스 + WAF 룰 부족 시
6. **2nd Mac mini node** ($600+) — 50+ 서비스

### 11.3 Break-even 인식
- 1-3 서비스: 무료, 운영 시간 ≤ 1hr/월
- 5-10 서비스: 무료 가능, 카디널리티 통제 필수, ~2hr/월
- 15-20 서비스: Grafana Pro 시점 (시간 ROI)
- 30+ 서비스: paid stack ~$30-50/월 합리적
