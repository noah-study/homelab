# Phase 2 — k3s + Cilium

**원본**: `../../2026-04-26-infra-cicd-template-plan.md` (이 phase 섹션)


**목표**: Mac mini Linux VM에 k3s 단일 노드 설치 + 기본 flannel CNI를 Cilium으로 교체 (NetworkPolicy + Hubble).

**참조**: 확장성 리뷰 F2 (sqlite 한계 인식), 보안 리뷰 F12 (PSA), F17 (Hubble UI)


## Tasks

- [Task 2.1: k3s 설치 (flannel/traefik 비활성화)](./task-2.1.md)
- [Task 2.2: Cilium CNI 설치](./task-2.2.md)
