# Review: Security (Adversarial)

**Reviewer perspective**: Red-team the design as if you intend to compromise it.
**Target**: `2026-04-26-infra-cicd-template-design.md`
**Date**: 2026-04-26

## TL;DR

설계는 모던한 GitOps 베이스라인으로 출발은 좋지만 **3개의 Critical 결함이 있다**: (1) self-hosted runner가 N개 서비스 레포의 빌드를 받으면서 GHCR/OIDC 자격증명을 보유 — 1개 레포 침해로 전체 파이프라인 침해 가능. (2) Image Updater가 infra 레포에 직접 commit할 수 있는 광범위 토큰을 가짐 — Image Updater pod 침해 시 platform/까지 임의 변경. (3) Cosign 서명 검증을 1~2개월 미루면서 그 사이 ArgoCD가 tag 기반 pull → tag 교체 공격에 노출. 추가로 Day-1에 잡지 않으면 회복 비용이 큰 결함 다수 존재.

## Findings

### F1. [Critical] Self-hosted runner 침해 → 전체 파이프라인 장악
- **Scenario**: noah-study/homelab의 reusable workflow는 N개 service 레포가 호출. 각 호출은 self-hosted runner (Mac mini)에서 실행. 한 service 레포가 침해되면 (collaborator 추가, PAT 유출, 외부 의존성 supply chain 등) 공격자는 Dockerfile/workflow에 임의 코드를 심을 수 있고 → runner에서 실행됨 → GHCR push 토큰, Sigstore OIDC 토큰, runner의 `_work` 디렉토리에 캐시된 모든 자격증명 추출 가능. 동일 머신에 도는 k3s에 kubeconfig가 있으면 클러스터 자체도 장악.
- **Mitigation**:
  - Runner를 `--ephemeral` 플래그로 1-job-1-container로 격리
  - actions-runner-controller (ARC) 도입해 k8s pod에 격리 (podman/sysbox로 runner 권한 최소화)
  - Runner가 도는 컨테이너에는 절대 호스트 docker socket 마운트 금지 → BuildKit rootless 사용
  - Service 레포에 외부 contributor 받는 경우 cloud runner로 강제 (self-hosted는 trusted 레포 한정)
- **Day-1 vs deferred**: **Day-1 필수**. 후순위로 두면 이후 재구성 비용 큼.

### F2. [Critical] Image Updater 토큰 = infra repo 광역 write
- **Scenario**: ArgoCD Image Updater는 dev kustomization에 image tag를 commit해야 하므로 infra repo write 토큰 필요. 일반적으로 PAT 또는 GitHub App. 토큰이 path-scoped가 아니면 Image Updater pod 침해 시 (image-updater 자체의 supply chain — Renovate가 해당 차트를 auto-merge한다면 즉시 트리거됨) `platform/`, `argocd/`, RBAC 매니페스트까지 임의 변경 가능 → 다음 ArgoCD sync에서 클러스터 전체 장악.
- **Mitigation**:
  - GitHub App을 만들어 권한 최소화: `contents: write` + `pull_requests: write`만, **path filter는 GitHub App이 지원 안 하므로** branch protection으로 보완
  - Image Updater의 `write-back-method: git` 대신 `git-pr` 모드 + 별도 봇 브랜치 → Renovate-style auto-merge 룰을 path filter (`apps/*/overlays/dev/kustomization.yaml`)로 강제
  - argocd-image-updater 컨테이너 이미지를 Renovate auto-merge 대상에서 **제외** (`platform-critical` 라벨)
  - `kustomization.yaml`만 수정 가능하도록 path-restricted bot 사용 (예: cli-pr-bot 패턴)
- **Day-1 vs deferred**: **Day-1**. 토큰 정책은 처음에 잘못 박으면 모든 서비스 onboarding이 그 위에 쌓여 변경 비용 큼.

### F3. [Critical] Cosign 서명 검증 1~2개월 지연 = tag 교체 공격 창
- **Scenario**: Day-1에는 서명만, 검증은 1~2개월 후. 그 사이 ArgoCD/Image Updater가 image를 **tag로** 참조하면 (현재 설계의 `kustomization.yaml`은 `newTag: sha-abc123` 형식) GHCR push 권한이 있는 누구든 같은 태그를 다른 이미지로 교체 가능 → ArgoCD가 다음 sync에서 새 digest를 가져와 배포. Cosign 서명 자체는 digest에 부착되어있지만, 아무도 검증하지 않으니 의미 없음.
- **Mitigation**:
  - **Day-1부터 digest 핀** — Image Updater의 `update-strategy: digest` 모드 사용. kustomization에 `newTag` 대신 `digest: sha256:...` 작성 (Kustomize 1.27+ 지원). 또는 reusable workflow에서 digest를 출력해 Image Updater가 SHA가 아닌 digest로 commit하도록.
  - GHCR 이미지에 immutable tag 정책 적용: `sha-*` 패턴은 push 후 변경 금지 (GHCR 자체 기능 제한적이라 어려움 → digest 방식이 더 확실)
  - 검증 정책(Sigstore Policy Controller)은 이후 도입해도, 그 전까지 digest 핀이 절반의 방어
- **Day-1 vs deferred**: **Day-1**. "검증 지연"의 진짜 위험은 tag 교체이며, digest 핀으로 즉시 막을 수 있다.

### F4. [High] Sealed Secrets 마스터키 — single point of total compromise
- **Scenario**: 마스터키는 git의 모든 SealedSecret을 복호화 가능. 1Password 백업 = 1Password 계정 침해 = 평생 모든 시크릿 노출 (git 히스토리는 영구). 키 회전 정책 미명시. 회전 시 기존 SealedSecret 재봉인 절차 미정의.
- **Mitigation**:
  - 백업을 **두 곳**: 1Password + 오프라인 암호화 USB (또는 Yubikey age key로 암호화)
  - 1Password 계정에 하드웨어 2FA 키 강제
  - 6개월마다 키 rotation: `kubectl delete secret -n kube-system sealed-secrets-key*` → 컨트롤러 재생성 → 모든 SealedSecret 재봉인 (스크립트 day-1에 작성)
  - SealedSecret을 namespace-scoped로 봉인 (`--scope namespace-wide`) — namespace 분리만으로도 한 시크릿이 다른 namespace에서 못 풀림
- **Day-1 vs deferred**: **Day-1**. 마스터키 손실 = 복구 불가. 백업·회전 절차는 처음부터.

### F5. [High] Renovate auto-merge 공급망 공격 (tj-actions/changed-files 패턴)
- **Scenario**: 2025년 3월 tj-actions/changed-files가 mutated tag 공격으로 침해 → 모든 사용자의 secrets 유출. 본 설계는 `github-actions: minor/patch auto-merge`. 공격자가 인기 action repo의 maintainer 토큰을 침해해 v3.5.1 → v3.5.2 patch에 악성 코드 심으면 → 다음 Renovate run에 자동 PR + auto-merge → reusable workflow가 호출되는 모든 서비스의 self-hosted runner에서 실행 → F1과 결합해 클러스터 전체 침해.
- **Mitigation**:
  - Actions를 **commit SHA로 핀** (Renovate `pinDigests: true`). Renovate는 SHA 업데이트 PR을 만들지만 maintainer가 SHA 위조하기는 어려움
  - Auto-merge 대상을 official org allowlist로 제한 (`actions/`, `docker/`, `sigstore/`)
  - 모든 외부 action에 대해 OpenSSF Scorecard 점수 7+ 요구
  - Self-hosted runner에 outbound network policy 적용 — 알려진 GHCR/Sigstore 외 egress 차단
- **Day-1 vs deferred**: **Day-1**. 핀과 allowlist는 즉시 무료.

### F6. [High] ingress-nginx CVE 패턴 (CVE-2025-1974 IngressNightmare)
- **Scenario**: 2025년 3월 CVE-2025-1974 — 임의 Ingress 리소스로 컨트롤러 RCE. 본 설계는 ingress-nginx를 Renovate `platform-critical` 라벨로 auto-merge 비활성. → 보안 패치도 수동 머지까지 대기. 패치 발표부터 사용자 머지까지 평균 7~14일 = 알려진 CVE 노출 기간.
- **Mitigation**:
  - Renovate 룰 분리: `platform-critical: minor/major 수동, security advisory는 자동 머지`. Renovate `vulnerabilityAlerts.enabled: true` + 별도 alert 룰로 24시간 내 머지
  - GitHub Security Advisory 구독 (이메일/Slack)
  - 대안 검토: Cilium Gateway API (ingress-nginx 대비 코드 베이스 작음)
  - PSA `restricted` + NetworkPolicy로 Ingress 컨트롤러의 폭발 반경 제한 (host network 차단 등)
- **Day-1 vs deferred**: **Day-1**: vulnerability auto-merge 룰 + 구독.

### F7. [High] Cosign keyless OIDC subject 정책 미정의
- **Scenario**: Section 2.4 "Cosign keyless 서명". 검증 시점에 어떤 OIDC subject를 trusted로 인정할지 미명시. 흔한 실수: `--certificate-identity-regexp 'https://github.com/noah-study/.*'` → noah org의 어떤 워크플로우든 서명 가능. 공격자가 noah org의 어떤 레포(예: 공개 저장소)에 reusable workflow를 추가해 임의 image에 서명 → "유효한 서명"으로 통과.
- **Mitigation**:
  - 검증 정책을 **특정 reusable workflow 경로 + ref**로 핀: `--certificate-identity 'https://github.com/noah-study/homelab/.github/workflows/reusable-build-push.yml@refs/heads/main'`
  - infra 레포의 main 브랜치 보호 (직접 push 금지, PR만)
  - reusable workflow 자체를 SHA로 호출하면 더 강함 (단, 운영 부담 ↑)
- **Day-1 vs deferred**: 검증 자체는 deferred지만 **정책 결정은 Day-1**. 나중에 verify를 켰을 때 즉시 적용 가능한 형태로 설계.

### F8. [High] Self-hosted runner 컨테이너 _work 디렉토리 persistence
- **Scenario**: 설계는 "동일 머신, 별도 컨테이너"라고만 명시. 컨테이너가 long-lived (재시작 없이 N job)면 `_work/` 디렉토리에 이전 빌드 잔재 (node_modules, .cargo, BuildKit cache, .docker/config.json) 남음. 악성 빌드가 `~/.cargo/bin/cargo` 또는 `node_modules/.bin/`을 변조 → 다음 빌드가 변조된 바이너리 실행 → secret 추출.
- **Mitigation**:
  - `--ephemeral` 플래그 (1 job per runner lifecycle)
  - 또는 ARC + Kubernetes mode (job마다 새 pod)
  - BuildKit cache는 외부 (예: GHCR cache export) 또는 read-only 마운트
- **Day-1 vs deferred**: **Day-1**. 한 번 자리 잡으면 전환 비용 큼.

### F9. [High] DNS 하이재킹 / DNSSEC 부재 / CAA 미설정
- **Scenario**: noah.dev는 Cloudflare. CF 계정 침해 시 (피싱, 비밀번호 재사용) 공격자가 DNS 변경 → DNS01 challenge로 LE 인증서 직접 발급 → 자기 origin으로 트래픽 우회 → 모든 TLS 무력화. CAA 레코드가 없으면 어떤 CA든 발급 가능 → DV 인증서 사기 발급 위험.
- **Mitigation**:
  - Cloudflare 계정에 hardware 2FA 키 (FIDO2) 강제
  - DNSSEC 활성화 (CF 한 클릭)
  - CAA 레코드: `0 issue "letsencrypt.org"` + `0 iodef "mailto:..."`
  - cert-manager의 Cloudflare API token은 **DNS edit 권한만, 특정 zone 한정** (CF token templates)
  - LE 발급 알림: Certificate Transparency 모니터 (예: Cert Spotter, crt.sh 알림)
- **Day-1 vs deferred**: **Day-1**. CF 계정 보안은 셋업 직후 즉시.

### F10. [Medium] OTel/Loki를 통한 시크릿 누출
- **Scenario**: `_base`가 모든 컨테이너에 OTel 환경변수 주입 (`OTEL_EXPORTER_OTLP_ENDPOINT` 등). 자동 instrumentation은 흔히 HTTP 요청 헤더 (Authorization 포함), 쿼리 파라미터 (apikey=), 에러 스택 트레이스의 환경변수 등을 캡처. → Grafana Cloud로 평문 송신 → 3 user free tier 누구나 조회 가능. 검색 가능한 영구 로그가 됨.
- **Mitigation**:
  - Alloy/OTel Collector에 redaction processor 적용: `Authorization`, `Cookie`, `apikey`, `password`, `token` 패턴 자동 제거
  - Loki retention을 14일로 강제 (Free tier 디폴트)
  - app 코드 컨벤션: 시크릿은 절대 로그하지 않기. linter 룰로 검출 (예: gitleaks를 빌드 단계에서)
  - Grafana Cloud user audit log 활성, 의심 쿼리 알림
- **Day-1 vs deferred**: **Day-1**. redaction processor는 Alloy 설치 시 한 번에 박는 게 자연스러움.

### F11. [Medium] Cloudflare Tunnel 토큰 유출 = origin 강탈
- **Scenario**: cloudflared 토큰은 SealedSecret. 마스터키 유출 시 (F4) 함께 풀림. 공격자가 자기 cloudflared 클라이언트를 노아의 tunnel에 attach → CF는 단순 round-robin → 일부 트래픽이 공격자에게 라우팅 → 자격증명 수집/MITM. CF 대시보드에서는 정상 connection으로 보임.
- **Mitigation**:
  - cloudflared service token + Account-level access policy (특정 IP/ASN만 허용)
  - CF Zero Trust dashboard에서 unexpected tunnel connection 알림
  - 토큰 정기 rotation (분기마다)
  - mTLS origin authentication 추가 (CF에서 클라이언트 인증서 발급, origin은 그것만 신뢰)
- **Day-1 vs deferred**: 모니터링과 service token은 **Day-1**, mTLS는 deferred.

### F12. [Medium] PodSecurityStandards `restricted` 도입의 erosion 위험
- **Scenario**: `restricted` 모드는 까다로움 (runAsNonRoot, drop ALL caps, no privilege escalation, seccomp, read-only root fs). 많은 외부 helm chart가 default로 안 맞음. 사람들이 단순히 namespace에 exception label (`pod-security.kubernetes.io/enforce: baseline`) 붙여 우회 → 표준이 점진 erosion.
- **Mitigation**:
  - `_base/deployment.yaml`에 표준 securityContext 박아 자체 서비스는 자동 준수
  - exception 요청은 PR로만, CODEOWNERS에 본인 강제 리뷰
  - Polaris 또는 kyverno로 cluster-wide audit 활성: exception 추가 시 알림
  - 외부 chart는 sub-chart wrapping으로 securityContext 강제
- **Day-1 vs deferred**: **Day-1**. erosion은 시작이 가장 어려움 — 처음부터 빡빡하게.

### F13. [Medium] Mac mini host (macOS) + OrbStack VM escape
- **Scenario**: Linux VM이 OrbStack 위. OrbStack는 빠르지만 빈번한 vulnerability — 2024년 file system sharing 권한 우회 사례. VM 침해 시 공격자가 host macOS 파일 접근 (mount된 ~/Documents 등). macOS의 SSH가 활성이면 VM에서 host로 lateral movement.
- **Mitigation**:
  - macOS Remote Login (SSH) 비활성
  - OrbStack의 file sharing을 최소화 (필요 디렉토리만, read-only)
  - OrbStack 자동 업데이트 활성 (Renovate 적용 안 됨 — 별도 모니터)
  - macOS FileVault 활성 (디스크 암호화)
  - host에 민감 데이터 없도록 VM 외부 저장소 사용
- **Day-1 vs deferred**: **Day-1**.

### F14. [Medium] Trivy-operator alert handling 미정의
- **Scenario**: Section 2.4에 trivy-operator 명시되지만 finding 처리 절차 없음. Vulnerability finding이 CRD에 쌓이기만 하고 아무도 안 봄 → 보안 부채가 보이지 않게 됨. 1년 후 "이미 알려진 CVE 200건 noted but unaddressed" 상태.
- **Mitigation**:
  - trivy-operator → Alloy → Grafana Cloud로 finding metric export
  - Grafana alert rule: severity Critical/High → 즉시 push notification (Discord webhook)
  - Medium/Low → 주간 다이제스트 이메일
  - 모든 finding에 SLA: Critical 7일, High 30일 — 위반 시 ArgoCD sync 차단 (admission webhook)
- **Day-1 vs deferred**: alert 라우팅은 **Day-1**, SLA enforcement는 deferred.

### F15. [Low] Sigstore Rekor 공개 로그 = 빌드 메타데이터 공개
- **Scenario**: 모든 cosign sign이 Rekor에 entry 생성 → 공격자가 rekor-cli로 noah org의 모든 레포/워크플로우 정찰 가능. 비공개 인프라/서비스 명세가 사실상 공개됨.
- **Mitigation**:
  - 수용. 가치 (서명) > 비용 (메타데이터 노출)
  - 또는 self-hosted Rekor + Fulcio (운영 부담 큼, 1인 환경에서 비현실적)
- **Day-1 vs deferred**: **수용**. 인지하고 진행.

### F16. [Low] GitHub Environments `production` reviewer = self → theatre
- **Scenario**: 1인 운영에서 본인이 본인 PR을 승인 → 실질적 second-eye 부재. 잘못된 promotion이 그대로 통과.
- **Mitigation**:
  - Wait timer 5분 추가 (취소 여유)
  - Slack/Discord webhook으로 promotion 알림 → 자기 자신에게도 "방금 뭘 했는지" 재인지
  - 향후 협업자 추가 시 mandatory reviewer로 전환
- **Day-1 vs deferred**: **Day-1**: wait timer + 알림.

### F17. [Low] Cilium Hubble UI 노출 시 정보 누설
- **Scenario**: Hubble UI 활성하면 클러스터 내 트래픽 흐름이 시각화됨. 외부 노출 시 (실수로 ingress 추가) 공격자에게 토폴로지 지도 제공.
- **Mitigation**:
  - Hubble UI는 ingress 없이 `kubectl port-forward`로만 접근
  - NetworkPolicy로 hubble-ui pod의 외부 노출 명시 차단
- **Day-1 vs deferred**: **Day-1** (설치 시).

## Cross-cutting concerns

1. **모든 자격증명이 1Password에 집중되는 단일 신뢰점** — 1Password 계정 침해 = 마스터키, CF API 토큰, GHCR PAT 일거 노출. 상이한 백업 매체로 분산 필요.
2. **공격 표면의 시간 차원 부재** — Cosign 검증 deferred, Sigstore Policy Controller deferred, Falco deferred 모두 "1~2개월 후"인데 그 사이 무방비 윈도우 명시 안 됨. 각 deferred 항목에 명시적 deadline + interim mitigation 정의 필요.
3. **Single operator 모델의 한계** — F16 외에도, 1인이 모든 권한 보유 = compromised account = total compromise. 적어도 break-glass account (offline 보관) 분리 권장.
4. **로그/메트릭 평면이 보안 평면을 덮는 위험** — OTel pipeline (F10)이 secret을 흘리면 Grafana Cloud (외부 SaaS)에 영구 잔존. 보안 사고 후 데이터 회수 불가.
5. **Day-1 미적용 항목들이 누적적으로 보안 부채를 만든다** — 백업/DR 부재 → 공격받았을 때 복구 어려움. 이는 보안 사고의 영향도(impact)를 키운다.

## Recommendation (design doc 변경 제안)

### Section 2.2 — `argocd-image-updater`
- "**write-back-method를 `git-pr`로, GitHub App + path-restricted bot 사용**" 명시 추가 → F2

### Section 2.3 — 외부 서비스
- Cloudflare 항목에 "**계정 hardware 2FA, DNSSEC, CAA 레코드**" 추가 → F9

### Section 2.4 — 보안·품질 표준
- "**Cosign 검증 deferred지만 digest pin은 Day-1**" 명시 추가 → F3
- "**Cosign keyless OIDC subject 정책: reusable workflow 경로+ref 핀**" 정의 → F7
- "**Self-hosted runner는 `--ephemeral` 또는 ARC k8s mode**" 명시 → F1, F8
- "**Sealed Secrets 마스터키 백업: 1Password + 오프라인 USB 이중화 + 6개월 회전 스크립트**" → F4
- "**OTel redaction processor (Alloy values)**" 추가 → F10
- "**Cloudflare service token + zero-trust monitoring**" → F11
- "**`_base`에 securityContext 표준 박기, exception PR 절차 명시**" → F12
- "**trivy-operator finding → Alloy → Grafana alert 라우팅**" → F14
- "**GitHub Environments `production`에 wait-timer 5분 + Discord webhook**" → F16

### Section 2.5 — Day-1 미적용
- 각 deferred 항목에 **interim mitigation + deadline** 추가:
  - Cosign 검증 (~ 2026-06): interim = digest pin
  - 백업/DR: interim = SealedSecrets 마스터키 1Password+USB, infra repo는 GitHub 자체가 백업
  - Sigstore Policy Controller: 검증 켤 때 OIDC subject regex 명시

### Section 4.5 — 옵저버빌리티
- "**Alloy redaction processor 컨벤션**" 명시 → F10

### 신설 Section 8 — Threat Model & Boundaries
- 신뢰 경계 다이어그램 추가 (GitHub ↔ runner ↔ cluster ↔ CF ↔ user)
- Day-1 미적용 항목들의 위험 인정 표
- 사고 대응 플레이북 (계정 침해 시, 마스터키 유출 시, runner 침해 시)

### 신설 Section 9 — Security Acceptance Tests
성공 기준에 추가:
- [ ] Service repo의 임의 PR이 GHCR push 토큰을 추출할 수 없다 (시뮬레이션 테스트)
- [ ] Image Updater 토큰이 platform/ 디렉토리를 수정할 수 없다 (path 제한 검증)
- [ ] CF 계정 2FA + DNSSEC + CAA 활성 (체크리스트)
- [ ] Cosign 서명이 모든 GHCR 이미지에 부착 + Rekor 조회 검증 (자동 테스트)
