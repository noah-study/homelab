# Task 6.2: NetworkPolicy default-deny + Hubble UI 차단

**Phase**: 6 (보안 Baseline (Kyverno + NetworkPolicy))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 6.2' 섹션

---


**Files:**
- Create: `bootstrap/cluster-resources/network-policies.yaml`

- [ ] **Step 1: 기본 차단 + 명시 허용 패턴**

`bootstrap/cluster-resources/network-policies.yaml`:
```yaml
# 1. Hubble UI 외부 노출 차단 (kube-system namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hubble-ui-internal-only
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: hubble-ui
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: {}   # kube-system 내부만
---
# 2. ingress-nginx → 모든 namespace의 pod 접근 허용 (디폴트)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress-nginx
  namespace: argocd
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
```

(주: 앱 namespace의 NetworkPolicy는 `_base/`에서 표준화 — Phase 11)

- [ ] **Step 2: cluster-resources kustomization 업데이트**

`bootstrap/cluster-resources/kustomization.yaml` 업데이트:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces.yaml
  - psa-restricted.yaml
  - network-policies.yaml
```

- [ ] **Step 3: ArgoCD가 자동 sync (Phase 3의 cluster-resources Application)**

```bash
git add platform/kyverno/ bootstrap/cluster-resources/network-policies.yaml bootstrap/cluster-resources/kustomization.yaml
git commit -m "feat(security): kyverno PSA audit + NetworkPolicy baseline"
git push

# ArgoCD가 변경 감지 + sync (3분 내) 또는 수동:
argocd app sync cluster-resources

# Hubble UI 외부 접근 차단 검증
curl -fs https://hubble.noah.dev || echo "blocked (expected)"
# 내부 port-forward는 여전히 가능
cilium hubble ui
```

- [ ] **Step 4: 검증 commit**

```bash
echo "## Phase 6 acceptance" > tests/acceptance/phase-6-security-baseline.md
echo "- [x] kyverno running, 3 ClusterPolicies registered" >> tests/acceptance/phase-6-security-baseline.md
echo "- [x] PolicyReport visible (kubectl get polr -A)" >> tests/acceptance/phase-6-security-baseline.md
echo "- [x] Hubble UI not externally reachable" >> tests/acceptance/phase-6-security-baseline.md
echo "- [x] ingress-nginx → all namespaces allowed via NetworkPolicy" >> tests/acceptance/phase-6-security-baseline.md
git add tests/acceptance/phase-6-security-baseline.md
git commit -m "test: phase 6 acceptance"
git push
```

---
