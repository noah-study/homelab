# Task 2.2: Cilium CNI 설치

**Phase**: 2 (k3s + Cilium)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 2.2' 섹션

---


- [ ] **Step 1: Cilium CLI 설치 (호스트)**

```bash
brew install cilium-cli
```

- [ ] **Step 2: Cilium 설치 (Hubble 활성)**

```bash
cilium install \
  --version 1.16.5 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=${VM_IP} \
  --set k8sServicePort=6443
```

- [ ] **Step 3: Cilium 설치 검증**

```bash
cilium status --wait
kubectl get nodes
```

기대: `cilium status` 모두 OK, 노드 `Ready`.

- [ ] **Step 4: Hubble UI는 ingress 없이 port-forward만 (보안 F17)**

`platform/cilium/hubble-access.md` 작성:
```markdown
# Hubble UI 접근 (외부 노출 금지)

```bash
cilium hubble ui
# 또는: kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

브라우저: http://localhost:12000

NetworkPolicy로 hubble-ui pod의 외부 노출 차단됨 (Phase 6).
```

- [ ] **Step 5: 검증 commit**

```bash
git add platform/cilium/hubble-access.md
echo "## Phase 2 acceptance" > tests/acceptance/phase-2-k3s-cilium.md
echo "- [x] k3s installed (no flannel, no traefik)" >> tests/acceptance/phase-2-k3s-cilium.md
echo "- [x] Cilium installed, node Ready" >> tests/acceptance/phase-2-k3s-cilium.md
echo "- [x] Hubble UI accessible via port-forward only" >> tests/acceptance/phase-2-k3s-cilium.md
git add tests/acceptance/phase-2-k3s-cilium.md
git commit -m "feat(platform): install k3s + Cilium CNI"
git push
```

---
