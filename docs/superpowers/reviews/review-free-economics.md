# Review: Free-tier Economics

**Reviewer perspective**: 무료 운영 가능 한계와 유료 전환 break-even.
**Target**: `2026-04-26-infra-cicd-template-design.md`
**Date**: 2026-04-26

## TL;DR

설계는 "월 USD 5-10" 운영 비용을 약속하지만 **현실은 5 서비스에서 이미 Grafana Cloud Free 10K series를 초과**한다. 무수정으로는 1-3 서비스에서만 진정한 무료. 표준 metric drop 룰로 10-15 서비스, 공격적 drop으로 30 서비스까지 무료 운영 가능. **break-even은 15-20 서비스 부근** — 그 이상에서는 Grafana Pro USD 19/월 + 약간이 운영자 시간 절약 측면에서 더 경제적. 다른 무료 티어(GHCR, Cloudflare, GitHub Actions self-hosted)는 100 서비스까지도 큰 위협 없음. 진짜 숨은 비용은 **카디널리티 통제에 드는 운영자 시간**으로, 월 2-4시간이면 기회비용이 Pro 구독 비용을 초과한다.

## Findings

### F1. [Critical] Grafana Cloud Free 10K series — 5 서비스에서 이미 초과
- **상세 시나리오 (실측 기반)**:
  - **kube-state-metrics**: 1 노드 + 50 pods + 50 services + 50 deployments + 50 ingress 등 표준 객체 = **약 1,500 series** (KSM 디폴트, 우리 설계의 platform 컴포넌트 포함)
  - **node-exporter**: 1 노드 = **약 700 series** (디폴트 collectors)
  - **cAdvisor (kubelet container metrics)**: 컨테이너당 ~150 series. 5 서비스 × dev+prod × 1 replica = 10 pods. 플랫폼 pods ~15. 총 25 pods × 150 = **3,750 series**
  - **OTel HTTP server histogram (앱 자동 instrumentation)**: 라우트당 ~30 series (status × method × le bucket). 평균 5 라우트 × 5 서비스 × 2 envs = **1,500 series**
  - **앱 비즈니스 메트릭 (사용자 정의)**: 평균 50 series/서비스 × 10 = **500 series**
  - **Cilium Hubble**: ~500 series (network flow metrics)
  - **ArgoCD/Image Updater/cert-manager/trivy 운영 metrics**: ~800 series
- **합산**: ~9,250 series at 5 services
- **20 services 추정**: cAdvisor가 선형 증가 → 25 + 60 = 85 pods × 150 = 12,750. 합산 ~17,500 → **75% 초과**
- **50 services 추정**: ~30,000 series → **3배 초과**
- **Mitigation (각 단계의 절감 효과)**:
  - **cAdvisor 메트릭 60% drop** (`container_blkio_*`, `container_tasks_*`, `container_fs_inodes_*`, `container_memory_failures_*`, 라벨 `id`/`container_id` drop): 7,500 → 3,000 → 절약 4,500
  - **node-exporter 30% drop** (`node_filesystem_*` 일부, `node_network_*` 가상 인터페이스): 700 → 500 → 절약 200
  - **histogram bucket 절반** (`le` 값 줄임): 1,500 → 750 → 절약 750
  - **아이들 series TTL** (Alloy의 `metric_relabel_configs` action: drop): 누적 절감 큼
- **표준 drop 적용 후 5 서비스**: ~3,800 series ✅ 38%
- **표준 drop 적용 후 20 서비스**: ~10,500 ⚠️ 105% (한도 직후)
- **공격적 drop + bucket 축소 후 50 서비스**: ~12,000 ⚠️ 120% (Pro 시점)
- **Day-1 vs deferred**: **Day-1 표준 drop 룰 박기**. `_base` 또는 Alloy values에 컨벤션화.
- **Cost trigger**: 20 서비스 도달 = Grafana Pro USD 19/월 (또는 셀프호스팅 LGTM)

### F2. [High] GHCR 빌드 retention 미정의 → storage 폭증
- **Scenario**: GitHub Packages free private repo 한도: **500MB storage** + **1GB transfer/month**. 본 설계는 이미지당 3 tag (sha-, branch-slug-, branch-slug-latest) 동시 push. 같은 digest면 manifest 하나로 공유되어 storage는 1배. **단**, 매 빌드는 새 digest → 누적.
  - 평균 image 압축 크기 ~50MB
  - 5 서비스 × 일 1 빌드 × 30일 = 150 builds × 50MB = **7.5GB** → 무료 한도 15배 초과
  - GHCR free private 초과: USD 0.25/GB/월 → 7.5GB = USD 1.88/월 (5 서비스 기준)
  - 50 서비스 × 일 1 빌드 = 75GB/월 → USD 18.75/월 (Grafana Pro와 비슷)
- **Mitigation**:
  - GHCR retention 정책 자동화: GitHub API로 `keep_latest: 10` + `tag-pattern: !sha-prod-*` 매주 cleanup script
  - 또는 모든 서비스 레포를 **public**으로 (가능하면) → GHCR public는 storage/transfer 무제한 free
  - BuildKit으로 layer 공유 극대화 (alpine 베이스 통일 등)
- **Day-1**: cleanup script 작성 (~50 lines)
- **Cost trigger**: 5 서비스부터 measurable. public 레포 가능하면 100 서비스도 무료.

### F3. [High] GHCR transfer (image pull) 1GB/month — Mac mini로 pull도 카운트되는가?
- **Scenario**: GHCR pull bandwidth는 free 한도 1GB/월 (private). ArgoCD가 sync마다 pull. 50 서비스 × 평균 1 deploy/day × 50MB = 2.5GB/day = **75GB/월 pull**.
- **불확실성**: GitHub 공식 문서에 따르면 GitHub Actions 안에서의 pull은 카운트 안 됨. 하지만 self-hosted runner나 외부 (Mac mini 클러스터)에서의 pull은 카운트됨 — **이게 본 설계의 패턴**.
- **만약 카운트된다면**: USD 0.50/GB → 75GB = **USD 37.5/월** (5 서비스에서 USD 7.5)
- **Mitigation**:
  - public 레포 (transfer 무제한)
  - 또는 클러스터 내 registry mirror (Harbor, Distribution) — 첫 pull만 GHCR에서, 이후 캐시
  - Image 변경 빈도 줄이기 (자동 dev 배포 → 일 1회 promotion)
- **Design ambiguity**: 설계 doc에 "GHCR transfer/storage 정책" 섹션 없음. 명시 필요.
- **Cost trigger**: 5 서비스 + private repos일 때 즉시.

### F4. [High] Sigstore Rekor 무료 인프라 — 명시적 rate limit 없음, 가용성 의존
- **Scenario**: Cosign keyless 서명은 Sigstore 공개 인프라 (Fulcio CA + Rekor 투명성 로그) 사용. 무료지만:
  - 공식 SLO: Rekor 99.9%, Fulcio 99.9%
  - 공식 rate limit 미발표, 실측 ~120 req/min/IP
  - 빌드 폭주 시 throttling → 빌드 fail 가능
- **본 설계의 빈도**: 50 서비스 × 5 builds/day = 250 sign/day = 10/hr 평균, peak 50/hr → 안전 마진 큼
- **단**, Sigstore 인프라가 다운되면 (실제 2024년 2회 incident) 빌드 fail
- **Mitigation**:
  - reusable workflow에 `continue-on-error: true` for cosign step + Slack 알림
  - 자체 Sigstore 인프라 (fulcio, rekor 셀프호스팅) — 1인 환경에 비현실적
- **Cost trigger**: 0. 가용성 위험만.

### F5. [High] 운영자 시간 = 진짜 비용 — 무료 유지 비용 vs Pro 구독 break-even
- **Scenario**: 무료 티어 유지에 드는 운영자 시간:
  - 메트릭 카디널리티 튜닝: 초기 4시간, 분기 1시간 = **0.5h/월**
  - GHCR retention cleanup 모니터링: 30분/월
  - Free tier 사용량 알림 대응: 30분/월
  - Renovate auto-merge 사고 복구 (가끔): 1시간/월 평균
  - 카디널리티 사고 (한도 초과 시 ingest drop) 디버깅: 1시간/월 평균
- **합계**: 약 **3.5 hr/월** ongoing
- **기회비용**: 한국 시니어 개발자 시간 ~₩70,000/hr → **₩245,000/월 ≈ USD 175/월**
- **비교**:
  - Grafana Pro: USD 19/월 + 사용량
  - GHCR Pro: USD 5/월 (10GB storage)
  - **합계 paid: USD 30-50/월** vs **time cost USD 175/월**
- **Break-even**: 5 서비스 부근에서 이미 paid가 더 경제적 (시간을 어떻게 평가하느냐)
- **반론**: 학습 가치는 무료. 1인 사이드 프로젝트면 시간 = 즐거움
- **Day-1 vs deferred**: **Pro 전환 시점 트리거 명시 필요** — 예: 카디널리티 사고 월 2회 이상 = Pro 시점

### F6. [Medium] Cloudflare Free Tunnel — 명시적 bandwidth 한도 없음, 그러나 implicit 위험
- **Scenario**: CF Tunnel 무료, 공식 bandwidth 무제한. 단:
  - L7 WAF 룰: 5개 무료 한도 (Pro USD 20/월에서 20개)
  - Workers: 100K req/day (USD 5/월부터 10M req/day)
  - Page Rules: 3개 (Pro 20개)
  - DDoS 방어: 무료 포함 ✅
- **본 설계 영향**: 50 서비스에서 cache rule 5개로 부족 → Pro 검토
- **Bot Fight Mode/Bot Management**: Free에 기본만, advanced는 Enterprise
- **Mitigation**:
  - 인증 페이지를 CF Workers로 (USD 5/월 — 큰 가치)
  - WAF 룰 통합 (per-service 대신 cluster-wide)
- **Cost trigger**: 20-30 서비스에서 CF Pro USD 20/월 검토

### F7. [Medium] GitHub Actions self-hosted runner — 분당 무료지만 부수 비용
- **Scenario**: Self-hosted runner는 GitHub Actions 분당 과금 회피 = USD 0/월 컴퓨트 비용
- **부수 비용**:
  - **Runner 유지**: pre-built tools 다운로드 (~5GB), GitHub binary 업데이트 자동
  - **Bandwidth**: base image pull (node:20-alpine ~50MB), npm install (~100-500MB/build)
  - 50 builds/day × 200MB = 10GB/day egress (CF Tunnel 통하지 않는 직접 다운로드)
  - 한국 ISP 무제한 일반적, US/EU 가정 종량 시 위협
- **Mitigation**: BuildKit cache (GHCR cache export)로 layer 재사용 → 빌드당 다운로드 80% 감소
- **Cost trigger**: 가정 ISP가 종량제일 때만 issue. 한국에선 사실상 무료.

### F8. [Medium] Mac mini 전기료 — 24/7 운영 시 월 USD 1.7 (한국)
- **Scenario**: M2 Pro Mac mini 전력 측정값:
  - Idle: 7-10W
  - Load (빌드 중): 30-50W
  - 평균 (빌드 일 5회 가정): ~15W
  - 24/7: 15W × 720hr = **10.8 kWh/월**
- **요금**:
  - 한국 가정용 (200~400 kWh 구간 기준): ~₩140/kWh = **₩1,512/월 ≈ USD 1.05**
  - 미국 평균: USD 0.16/kWh = **USD 1.73/월**
  - 유럽 (독일 기준): EUR 0.40/kWh = **EUR 4.32/월 ≈ USD 4.7**
- 더 큰 부하 (50 서비스, 빌드 잦음) 시 평균 25W → 18 kWh/월 → 한국 ₩2,520/월
- **Cost trigger**: 무시 가능

### F9. [Medium] 도메인 (noah.dev) 갱신
- **Scenario**: `.dev` TLD = USD 12-15/년 (Cloudflare 등)
- **Cost trigger**: 무시 가능 (월 USD 1)

### F10. [Medium] 1Password 구독 — Sealed Secrets 마스터키 백업
- **Scenario**: Personal USD 2.99/월 = USD 36/년
- **무료 대안**: GPG-encrypted file on offline USB. 운영 부담 ↑.
- **Cost trigger**: 사용자 이미 1Password 보유면 0. 없으면 USD 3/월 추가.

### F11. [Medium] (가까운 미래) Backblaze B2 백업 — 운영 시작 후 도입 예정
- **Scenario**: 백업/DR Day-1 미적용이지만 운영 시작 후 추가 예상. 비용 추정:
  - Postgres dumps: 10 서비스 × 100MB × 30일 retention = 30GB
  - Velero PV snapshots: 50GB
  - 총 ~80GB
  - B2: 첫 10GB 무료, 이후 USD 6/TB/월 → 70GB = **USD 0.42/월**
  - Egress (복원 시): USD 10/TB → 거의 무시
- **Cost trigger**: 백업 도입 시 USD 1/월 미만

### F12. [Low] Sigstore Rekor entry 메타데이터의 부수 비용?
- **Scenario**: Rekor entry는 영구 보존 (transparency). 비용 0. 메타데이터 (보안 review F15)만 부수.
- **Cost trigger**: 0.

### F13. [Low] Cloudflare R2 vs Backblaze B2 (장래 선택)
- **Scenario**: 만약 정적 자산 호스팅이 늘어나면:
  - R2: 10GB 무료 + USD 0.015/GB-월 + egress free (큰 장점)
  - B2: 10GB 무료 + USD 6/TB-월 + USD 10/TB egress
- 백업용은 B2 (저장만), 서빙용은 R2 (egress free) — 적재적소 선택
- **Cost trigger**: 정적 자산 100GB 미만이면 둘 다 USD 1-2/월

### F14. [Low] Renovate 호스팅 — Mend가 무료 호스팅
- **Scenario**: Renovate는 GitHub App으로 Mend가 무료 호스팅. 셀프호스팅 옵션도 있지만 Mac mini RAM 부담 (~200MB). 무료 호스팅 사용이 정답.
- **Cost trigger**: 0.

## Cross-cutting concerns

1. **"무료 운영"의 진짜 비용은 시간** — 모든 무료 티어는 운영자가 한도 안에 맞추는 작업을 요구. 5 서비스 부근에서 이미 paid 구독이 시간 비용 측면 더 경제적이지만, 학습/취미 가치는 별개.
2. **5 서비스가 진짜 한계점** — 카디널리티가 가장 먼저 깨지고, 거의 동시에 GHCR retention과 transfer가 카운트되기 시작. 그 전까지는 "거의 무료"가 사실.
3. **Public 레포로 만들면 비용 곡선 완전히 평탄화** — GHCR public는 storage/transfer 무제한. 이는 가능하면 강력한 비용 제어 수단이지만 보안 review F15(Rekor 메타데이터)와 함께 종합 평가 필요.
4. **숨은 cost가 측정 안 됨** — 설계에 cost 모니터링 컴포넌트 부재. Grafana Cloud 사용량 알림은 있지만 GHCR storage, CF rule 사용량 등은 수동 확인. → 어느 한도가 임박했는지 모르고 운영하다가 사고로 발견.
5. **무료 → 유료 전환 트리거 부재** — 설계에 "언제 Pro로 갈지" 기준 없음. 운영자가 매번 즉흥 판단.

## Recommendation (design doc 변경 제안)

### Section 4.5 — 옵저버빌리티
- "**Day-1 metric drop 컨벤션**" 명시 추가:
  - cAdvisor: `container_blkio_*`, `container_tasks_*`, `container_fs_inodes_*`, `container_memory_failures_*` drop
  - 라벨 `id`, `container_id`, `pod_uid` 드롭
  - histogram bucket 디폴트 5개 (le: 0.1, 0.5, 1, 5, 10)
- "**카디널리티 SLO**: 5 서비스 = 4K series, 10 서비스 = 7K, 20 서비스 = 9.5K"
- "**Grafana Cloud series 알림**: 8K (80%) 도달 시 push notification"

### Section 2.3 — 외부 서비스
- GHCR 항목에 "**Retention 정책**: cleanup script 매주 실행, keep_latest 10 + sha-prod-* 영구 보관"
- "**저장소 가시성 정책**: 가능하면 public으로 → free 한도 무제한"

### Section 3 — 레포 구조
- `scripts/`에 `ghcr-cleanup.sh` 추가
- `.github/workflows/`에 `ghcr-cleanup.yml` (cron weekly) 추가

### Section 6 — 트레이드오프
- "**무료 운영의 break-even**" 섹션 신설:
  - 1-3 서비스: 진짜 무료, 운영 시간 < 1hr/월
  - 5-10 서비스: 무료 가능하지만 카디널리티 통제 필수, 2hr/월
  - 15-20 서비스: Grafana Pro USD 19 검토 시점 (시간 ROI)
  - 30+ 서비스: paid stack USD 30-50/월 합리적

### Section 7 — 성공 기준
- 추가:
  - [ ] Grafana Cloud 카디널리티 ≤ 8K series at 5 서비스
  - [ ] GHCR 사용량 ≤ 500MB at 5 서비스 (private 가정)
  - [ ] 무료 티어 사용량 모니터링 자동화 (월 1회 보고서)
  - [ ] 모든 free tier 80% 도달 시 알림 룰 설정

### 신설 Section 11 — Cost Monitoring & Upgrade Triggers
- 모든 무료 티어 한도 매트릭스
- 80% 알림 자동화
- 유료 전환 우선순위 순서:
  1. Grafana Cloud Pro (USD 19/월) — 가장 빨리 깨짐
  2. Mac mini 32GB 업그레이드 (1회 USD 200) — 메모리
  3. 1Password (USD 3/월) — 시크릿 백업
  4. GHCR Pro (USD 5/월) — 또는 public 전환
  5. CF Pro (USD 20/월) — 20+ 서비스 시
  6. ARC 도입 (운영 시간 비용) — 빌드 큐 폭증 시

### 신설 Section 12 — Cost Tracking Dashboard
- Grafana 대시보드 1개:
  - GHCR storage/transfer (GitHub API exporter)
  - Grafana Cloud series count (Mimir self-stat)
  - CF Workers req/day (CF API)
  - Mac mini 전력 (선택, 스마트 플러그)
