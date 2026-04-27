# homelab

Mac mini 셀프호스팅 GitOps 프레임워크. 학습 + framework 소유 목적.

## Stack
k3s + Cilium + ArgoCD (App-of-Apps 자기관리) + ApplicationSet + Image Updater (digest pin) + Sealed Secrets + cert-manager + Cloudflare Tunnel + Cosign keyless + ARC + Grafana Cloud Free.

## 문서
- **설계**: [`docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md`](docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md)
- **구현 플랜**: [`docs/superpowers/plans/2026-04-26-infra-cicd-template-plan.md`](docs/superpowers/plans/2026-04-26-infra-cicd-template-plan.md)
- **리뷰**: 보안/확장성/무료 운영 — [`docs/superpowers/reviews/`](docs/superpowers/reviews/)

## Quick start

신규 셋업: `bootstrap/README.md` 절차 (Phase 3 완료 후 생성)
새 앱 추가: `./scripts/new-app.sh <service-name>` (Phase 11 완료 후 생성)
사고 대응: `docs/runbook.md` (Phase 17 완료 후 생성)
규모 확장: `docs/scaling-playbook.md` (Phase 17 완료 후 생성)

## Status
**Phase 0 완료** — 설계 + 플랜 + 리뷰 작성. 구현 시작 전.
