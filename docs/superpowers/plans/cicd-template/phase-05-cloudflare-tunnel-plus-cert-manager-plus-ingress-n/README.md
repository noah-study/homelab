# Phase 5 — Cloudflare Tunnel + cert-manager + ingress-nginx

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: 외부에서 `https://argocd.noah.dev` 같은 URL로 클러스터 서비스에 안전하게 도달 가능. NAT 우회 (Tunnel), 와일드카드 인증서 (cert-manager), 단일 ingress class (ingress-nginx).


## Tasks

- [Task 5.1: cert-manager 설치 + Cloudflare DNS01 ClusterIssuer](./task-5.1.md)
- [Task 5.2: ingress-nginx 설치](./task-5.2.md)
- [Task 5.3: cloudflared (Tunnel) 배포](./task-5.3.md)
- [Task 5.4: ArgoCD Ingress 추가 (첫 외부 노출)](./task-5.4.md)
