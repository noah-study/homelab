# Review: Scalability (Stress Test)

**Reviewer perspective**: 5 / 20 / 50 / 100 서비스 규모에서 무엇이 어디서 깨지는가.
**Target**: `2026-04-26-infra-cicd-template-design.md`
**Date**: 2026-04-26

## TL;DR

설계는 **5~10 서비스에서 견고하고, 20~30에서 튜닝이 필수, 50에서 한계 도달, 100은 비현실적**이다. 가장 먼저 깨지는 것은 **Mac mini RAM**과 **Grafana Cloud Free 10K series** (이 둘은 거의 동시). k3s 자체는 100+ 서비스도 가능하지만 단일 노드 SPOF + sqlite 백엔드가 운영 마진을 잠식. ArgoCD 컨트롤러는 200 Application 부근에서 reconcile 지연이 체감. Image Updater + ApplicationSet의 GitHub API polling이 50 서비스에서 rate limit 압박. 마이그레이션 경로(2nd node, 외부 etcd, ARC, webhook 전환)는 명확하지만 설계 doc에 단계별 전환 트리거가 명시 안 됨.

## Findings

### F1. [High] Mac mini 16GB RAM 천장 — 30 서비스에서 헤드룸 소진
- **Scenario**: Linux VM ~2GB + k3s control plane ~512MB + Cilium agent ~300MB + ArgoCD (server+repo+controller+image-updater+applicationset) ~800MB + ingress-nginx ~200MB + cert-manager ~150MB + sealed-secrets ~50MB + Alloy DaemonSet ~250MB + trivy-operator ~300MB + cloudflared ~80MB + Hubble UI ~150MB + kube-system overhead ~500MB ≈ **5.3GB platform baseline**. 서비스 1개당 평균 (Pod replicas:1, OTel SDK 포함): app ~80MB, sidecar 0 = ~80MB.
- 5 services dev+prod = 800MB → total 6.1GB → 여유 OK (10GB 잔여)
- 20 services × 2 envs × 80MB = 3.2GB → total 8.5GB → 여유 (7.5GB 잔여)
- 30 services × 2 envs = 4.8GB → total 10.1GB → 50% 헤드룸 (6GB 잔여)
- 50 services × 2 envs = 8GB → total 13.3GB → **헤드룸 소진** (2.7GB 잔여, OOM 위험)
- 100 services = 19.3GB → **16GB 머신에서 불가능**
- **추가 변수**: Postgres/Redis 같은 stateful은 256MB+/instance → 5개만 있어도 1.3GB
- **Mitigation**:
  - 32GB Mac mini로 즉시 업그레이드 시 50 서비스까지 안정 (~$200 1회 비용 vs 클라우드 월 비용 비교 압도적 우위)
  - 또는 2번째 Mac mini를 worker node로 추가 (k3s `--token` 조인) — 약 1시간 셋업
  - 모든 stateful은 Mac mini 외부 (Neon, Supabase, Upstash 등 free tier) — 메모리 압력 회피
- **Threshold**: 16GB로는 25-30 서비스. 32GB로는 50-60. 그 이상은 노드 추가.

### F2. [High] k3s sqlite (kine) 쓰기 처리량 — 50+ 서비스에서 latency 증가
- **Scenario**: k3s 디폴트는 etcd 대신 sqlite (kine)로 단순화. sqlite는 단일 writer + WAL mode로 **300-500 writes/sec까지 안정**, 그 이상에서 latency P99 급증. 본 설계의 write 부하:
  - ArgoCD reconcile: 200 Application × 3-min poll = ~1.1 events/sec (정상)
  - Image Updater: 50 services × 2-min poll → API 호출 + 변경 시 commit (read 위주)
  - ApplicationSet 재계산: 디렉토리 변경 시 generator 트리거 (낮은 빈도)
  - Cilium identity 갱신, network policy 평가 (지속적, 약 5-20/sec at 50 services)
  - kube-state-metrics scrape (15s interval, 다수 metric → ETCD watch)
  - **합산 추정**: 50 서비스에서 50-150 writes/sec, 100 서비스에서 150-400/sec
- **Threshold**: 50 서비스 OK, 100 서비스 압박, 200 서비스 한계
- **Mitigation**:
  - k3s를 embedded etcd 모드로 전환 (`--cluster-init` + 3 노드면 HA, 1 노드도 가능): 1k+ writes/sec 안정
  - 또는 외부 Postgres 백엔드 (`--datastore-endpoint=postgres://...`): 운영 단순, Neon free tier 사용 가능
  - sqlite 유지 시: VACUUM cron 추가, WAL 체크포인트 튜닝
- **신호**: `kubectl get --raw /metrics | grep apiserver_request_duration` P99 > 500ms = 임박

### F3. [High] ArgoCD application-controller — 200 App에서 reconcile 지연 체감
- **Scenario**: ArgoCD 디폴트 reconcile 간격 3분. controller는 단일 replica. 50 서비스 × 2 환경 = 100 Application. 각 reconcile = git fetch + manifest gen (Kustomize/Helm) + diff vs cluster + (필요시) sync. 평균 200ms-2s/Application. 100 App × 1s avg = 100s 직렬 처리 → 3분 budget 안에서 OK. 200 App × 1s = 200s → **budget 초과**, lag 누적.
- **Threshold**: ArgoCD 공식 가이드는 200+ Application부터 controller sharding 권장
- **Mitigation**:
  - `controller.replicas: 2-3` + sharding (단, 메모리 ↑)
  - Webhook receiver 활성: GitHub → ArgoCD push notification → polling 의존 감소
  - Reconcile 간격 늘리기 (5-10분) — drift detection이 늦어지지만 자동 sync니까 큰 문제 아님
  - Application별 syncPolicy 튜닝: `selfHeal: true`만 켜고 자동 polling은 webhook 의존
- **단일 노드 제약**: sharding은 메모리/CPU 헤드룸 필요 → F1과 충돌

### F4. [High] Image Updater + ApplicationSet GitHub API rate limit
- **Scenario**: GitHub API authenticated rate limit = **5000 req/hr**. 본 설계의 호출 주체:
  - Image Updater: 디폴트 2분 폴링 = 30/hr. 서비스당 image 1개 가정. 50 services = 1500/hr (단, 같은 PAT/App 토큰 공유)
  - ApplicationSet git generator: 3-min 폴링 = 20/hr × 3 ApplicationSets = 60/hr
  - ArgoCD 자체 git polling per Application: 3-min = 20/hr × 100 App = **2000/hr**
  - Renovate (주 1회 batch): peak 시 200~500 calls/hr
  - GitHub Actions 자체 (workflow runs API)
- **합산 (50 services)**: ~3700/hr → 80% 사용량, 공격적 폴링 시 한도 도달
- **합산 (100 services)**: ~7000/hr → **초과**
- **Mitigation**:
  - GitHub App 사용 (rate limit이 installation 단위, 5000 +bonus)
  - Webhook 전환: ArgoCD `argocd-server` 에 webhook receiver 활성, Image Updater 0.13+ webhook
  - Image Updater 폴링 5분 이상으로 (실시간성 일부 포기)
  - ArgoCD reconcile 간격 5분으로
- **신호**: GitHub API response header `X-RateLimit-Remaining` 모니터링 (Alloy로 메트릭화)

### F5. [High] Self-hosted runner 단일 동시성 = PR storm 시 큐 폭증
- **Scenario**: 설계는 self-hosted runner 1개 ("동일 머신, 별도 컨테이너"). 빌드 평균:
  - Node.js (캐시 hit): ~3분
  - Go: ~2분
  - Rust: ~5-15분
  - 멀티아키 빌드 (ARM64+AMD64 with QEMU): 2-3배 가중
- 시나리오: Renovate 월요일 9시 batch → 50 service repos × 평균 3 PR each = 150 jobs 큐. 각 3분 가정 = **7.5시간 직렬 처리**. 그 사이 사람의 코드 push가 큐 뒤로 밀림.
- **Mitigation**:
  - Runner 2-4개 병렬 (Mac mini RAM 허용 한도 내) → Cilium runner pod 4 × ~1GB working set = 4GB 추가
  - **ARC (Actions Runner Controller)**: pod 단위 동적 생성, idle 시 0개 → 효율적
  - Renovate 룰: `prHourlyLimit: 4` + schedule 분산 (평일 새벽)
  - Concurrency group으로 같은 서비스 동시 빌드 방지
  - QEMU 에뮬레이션 회피: ARM64 전용 또는 매트릭스로 분리하여 cloud runner 사용
- **Threshold**: 10 서비스 + 일반 패치 빈도면 단일 runner OK. 30+ 서비스면 ARC 필수.

### F6. [Medium] Mac mini 디스크 I/O — Loki 로컬 운영 시 빠르게 saturation
- **Scenario**: 본 설계는 Loki 외부(Grafana Cloud) → 로컬 디스크 부담 적음. 하지만 다음 시나리오는 위험:
  - Free tier 초과해서 Loki 셀프호스팅 전환 시: 50 services × 100MB logs/day = 5GB/day → 150GB/month
  - PV 사용하는 stateful (Postgres) 5개 = ~50GB, write IOPS 지속
  - k3s sqlite + ArgoCD repo 캐시 + GHCR pull 캐시 + BuildKit 캐시 = ~50GB
- M2 Mac mini 256GB SSD 시 "data" 파티션 ~150GB만 사용 가능 → **6개월 안에 fill**
- **Mitigation**:
  - Mac mini 1TB SSD 옵션 (구매 시 +$200)
  - 외부 NVMe USB-C (~$80 1TB)
  - 모든 stateful 외부화 유지 (B2/Neon/Upstash)
  - BuildKit cache TTL 7일로 제한
- **Threshold**: 256GB로는 20 서비스 안전. 1TB로는 100+.

### F7. [Medium] 네트워크 대역폭 — 주거용 ISP upload 한계
- **Scenario**: 한국 가정 인터넷 1Gbps 대칭 일반적이지만 미국/유럽은 50-100Mbps upload 흔함. CF Tunnel은 모든 응답이 upload. 50 services × 평균 5 req/s × 50KB = 12.5MB/s = 100Mbps → ISP upload 캡 도달. CF의 edge cache 활용해도 동적 API는 직접 응답.
- **Mitigation**:
  - CF의 캐시 룰 적극 활용 (cache-control 헤더)
  - 정적 자산은 R2/S3 분리 (CF Workers로 라우팅)
  - 트래픽 많은 서비스는 클라우드 (Fly.io edge)로 분리
- **Threshold**: 한국 1Gbps면 100+ 서비스 가능, 미국 50Mbps면 10-15 서비스에서 압박

### F8. [Medium] dev 환경 동시 작업 충돌 — feature 브랜치 stacking
- **Scenario**: 설계에서 인정한 트레이드오프 (Section 6.3). 50 services 운영 시 동시 작업 가지가 5-10개 일상적. `feat-login-abc`이 dev에 배포된 직후 `feat-billing-def` push → Image Updater가 dev kustomization 덮어쓰기 → login PR 검증자가 다른 코드를 보게 됨.
- **수치**: 50 services × 평균 동시 작업 가지 1.5개 = 75 활성 브랜치 → 일주일 단위 dev 환경 평균 수명 분 단위
- **Mitigation**:
  - PR Preview 환경 도입 (설계의 deferred 항목) — ApplicationSet PullRequest generator
  - Namespace per branch (dev-feat-login, dev-feat-billing) — 운영 부담 큼
  - 또는 1인 운영의 자제력에 의존 (실용적이지만 fragile)
- **Threshold**: 5-10 서비스 + 1인이면 견딤. 20+ 서비스면 PR Preview 필수.

### F9. [Medium] Grafana Cloud Free 10K series 초과 — 5-10 서비스에서 도달
- (자세한 분석은 review-free-economics.md F1)
- **Scenario**: kube-state-metrics ~1500 + node-exporter ~700 + cAdvisor 7500 (50 pods × 150) + 앱 메트릭 기본 = **5 서비스에서 이미 10K 도달**. 공격적 relabel 적용 시 50 서비스까지도 가능하지만 운영 부담.
- **Threshold**: 무수정 5 서비스, 표준 relabel 20 서비스, 공격적 drop 50 서비스, 그 이상 paid

### F10. [Medium] Renovate PR 폭증 — 50 레포에서 주 1회 batch 시 100+ PR
- **Scenario**: 설계는 `prHourlyLimit: 4` + schedule. 50 service repos + infra repo = 51 repos. 각 1-5 PR/주 평균 3 = 153 PR/주. Auto-merge 80% = 30개 수동 리뷰. 1인이 주당 30 PR 리뷰 = 1시간/주 (개당 2분 가정).
- **추가 부담**: 모든 PR이 self-hosted runner 큐 → F5와 결합해 큐 폭증
- **Mitigation**:
  - Renovate `groupName` 적극 활용 (예: 모든 actions 묶어 1 PR)
  - 도메인별 schedule (월요일 deps, 수요일 platform)
  - `automerge: true` 비율 늘리기 (Cosign이 깔리면 더 안전)
- **Threshold**: 20 repos까지 1인이 견딤. 50+에서는 grouping 필수.

### F11. [Medium] cert-manager DNS01 propagation lag — 신규 도메인 다발 추가 시
- **Scenario**: 와일드카드 인증서 (`*.noah.dev`, `*.dev.noah.dev`) 1개 공유 → 새 서비스 추가에도 인증서 작업 0. **단**, 새로운 base 도메인 추가 시 (예: `*.api.noah.dev`) DNS01 challenge → CF API → propagation 30s-2min × 첫 발급. cert-manager가 동시 다발 발급 시 CF API rate limit (1200/5min) 위협.
- **Mitigation**:
  - 와일드카드 1개로 모든 서비스 커버하는 컨벤션 강제 (서브 도메인 추가 자제)
  - DNS01 대신 Cloudflare Origin Cert (영구 인증서) 검토
- **Threshold**: 와일드카드 컨벤션 유지 시 100+ 서비스 무문제.

### F12. [Low] Cosign 서명 — 빌드당 5-10s 추가
- **Scenario**: keyless 서명 = OIDC token 획득 + Fulcio 인증서 발급 + Rekor 등록. 평균 5-10초/빌드. 50 builds/day × 7s = 6분/일 누적. 미미.
- **Sigstore 무료 인프라 의존**: Fulcio/Rekor가 다운되면 빌드 fail. 본 설계는 fail-open 정책 미명시.
- **Mitigation**: workflow에 `continue-on-error` for cosign step + 알림. Sigstore 무료 인프라는 99.9% SLO 발표.
- **Threshold**: scaling 우려 없음.

### F13. [Low] Cilium identity 한계 — 65535 identities
- **Scenario**: Cilium은 endpoint별 identity 할당 (label set 조합). 65535 identity까지. 한 서비스 = ~3-5 identity. 100 서비스 = 500 identity. 매우 여유.
- **Mitigation**: 우려 없음.

### F14. [Low] trivy-operator 스캔 동시성 — 50+ 서비스에서 scanner pod 누적
- **Scenario**: trivy-operator는 새 image 발견 시 vulnerability scan job 띄움. 디폴트 동시성 ~10 jobs. PR 폭주 시 scan job 큐. 각 job ~30s-2min, ephemeral.
- **Mitigation**: scanner concurrency 튜닝, scan interval 늘리기
- **Threshold**: 100 서비스에서도 운영 가능 (ephemeral pod의 cpu/mem 부하만 주의).

## Cross-cutting concerns

1. **단일 노드 SPOF의 누적 효과** — 설계가 인정한 "99.5% SLA". 실제로는 macOS 자동 업데이트 재부팅 + OrbStack 자동 업데이트 + ISP outage가 모두 누적되어 99% 이하 가능.
2. **확장 마이그레이션 경로의 명시적 트리거 부재** — 설계 doc에 "30 서비스 도달 시 32GB 업그레이드", "50 서비스 도달 시 ARC 도입", "100 서비스에서 2nd node" 같은 단계적 트리거가 없음. 운영자가 임계치 모니터링 안 하면 고장 후 발견.
3. **GitHub API rate limit이 다수 컴포넌트 공유 자원** — Image Updater, ApplicationSet, ArgoCD, Renovate, Actions가 모두 같은 PAT/App 토큰의 5000/hr 풀을 나눠 씀. 한 컴포넌트가 폭주하면 전체 영향.
4. **Mac mini 1대로 모든 컨트롤 + 데이터 + 빌드** — 빌드 폭주가 ArgoCD reconcile, 클러스터 sqlite write 등 모든 것에 영향. 빌드 머신 분리가 자연스러운 진화.
5. **단일 운영자의 인지 부하 한계** — 30+ 서비스가 도달하면 사람이 한 명으로 모든 PR 리뷰 + 알림 처리 + 사고 대응 어려움. 자동화 정도가 운영 한계가 되며, 본 설계의 "수동 promotion"이 병목.

## Recommendation (design doc 변경 제안)

### Section 6 — 의식적 트레이드오프
다음 항목 추가:
- "**규모별 명시 트리거**" 섹션 신설:
  - 10 서비스: 본 설계 그대로
  - 20 서비스: Renovate grouping 강화, runner 2개로
  - 30 서비스: Mac mini 32GB 업그레이드 또는 ARC 도입
  - 50 서비스: PR Preview, ApplicationSet webhook 전환, k3s 백엔드 외부 Postgres
  - 100 서비스: 2nd node, ArgoCD controller sharding, Loki 셀프호스팅 검토

### Section 4 — 핵심 컨벤션
- "**GitHub Actions runner = `--ephemeral` + ARC**" 명시 (보안 F1과 함께)
- "**Webhook 우선, polling은 fallback**" — ArgoCD/Image Updater 모두 webhook receiver 활성

### Section 2.2 — 클러스터 컴포넌트
- ArgoCD에 "controller.replicas 사항: 200 App 도달 시 sharding 활성"

### Section 4.5 — 옵저버빌리티
- "**Metric drop 룰을 _base에 표준화**, 카디널리티 SLO 5K series at 5 서비스"

### Section 7 — 성공 기준
- 추가:
  - [ ] GitHub API rate limit 사용량 60% 이하
  - [ ] ArgoCD reconcile lag P99 < 60s
  - [ ] Self-hosted runner 큐 대기시간 P95 < 5분
  - [ ] Mac mini RAM 사용률 75% 이하
  - [ ] sqlite WAL 크기 모니터링 (kine /metrics)

### 신설 Section 10 — Scaling Playbook
- 단계별 트리거 + 액션 + 예상 작업시간 매트릭스
- "분명히 깨질 때까지 스택을 바꾸지 말 것" 원칙 (premature scaling 방지)
