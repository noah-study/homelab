# Task 3.3: App-of-Apps root + cluster-resources

**Phase**: 3 (ArgoCD Bootstrap (자기관리))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 3.3' 섹션

---


**Files:**
- Create: `bootstrap/cluster-resources/kustomization.yaml`
- Create: `bootstrap/cluster-resources/namespaces.yaml`
- Create: `bootstrap/cluster-resources/psa-restricted.yaml`
- Create: `bootstrap/root.yaml`
- Create: `bootstrap/README.md`

- [ ] **Step 1: 클러스터 리소스 (namespace + PSA)**

`bootstrap/cluster-resources/namespaces.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: platform
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    pod-security.kubernetes.io/enforce: baseline   # cert-manager 일부 컴포넌트는 restricted 미준수
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    pod-security.kubernetes.io/enforce: baseline
---
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: actions-runner-system
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

`bootstrap/cluster-resources/psa-restricted.yaml`:
```yaml
# 디폴트 PSA: 새 namespace는 restricted (warn + audit)
# 실제 enforce는 namespace 라벨로
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-security-webhook
  namespace: kube-system
data:
  policy.yaml: |
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
```

`bootstrap/cluster-resources/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces.yaml
  - psa-restricted.yaml
```

- [ ] **Step 2: App-of-Apps root**

`bootstrap/root.yaml`:
```yaml
# 모든 후속 ArgoCD Application/ApplicationSet의 진입점
# 이 Application 1개만 수동 apply하면 그 안에서 platform/argocd 모두 자동 동기화
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/noah-study/homelab
    targetRevision: main
    path: argocd
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
---
# ArgoCD 자기관리 Application — argo-cd Helm chart를 ArgoCD가 추적
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-cd
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/noah-study/homelab
    targetRevision: main
    path: bootstrap/argo-cd
    plugin: {}   # Kustomize + helm 자동 인식
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false   # ArgoCD 자기 자신 prune 위험
      selfHeal: true
---
# Cluster resources도 ArgoCD가 관리
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cluster-resources
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/noah-study/homelab
    targetRevision: main
    path: bootstrap/cluster-resources
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- [ ] **Step 3: ArgoCD Kustomize+Helm 플러그인 활성**

ArgoCD는 디폴트로 Helm-inflated Kustomize를 지원하지 않으므로 활성 필요:

`bootstrap/argo-cd/values.yaml`에 추가:
```yaml
configs:
  cmp:
    create: true
  params:
    server.insecure: true
    "configmanagementplugins": |
      - name: kustomize-helm
        init:
          command: ["sh", "-c"]
          args: ["helm dependency build || true"]
        generate:
          command: ["sh", "-c"]
          args: ["kustomize build --enable-helm"]
```

또는 더 간단히 ArgoCD 디폴트의 `kustomize.buildOptions: --enable-helm`:
```yaml
configs:
  cm:
    kustomize.buildOptions: "--enable-helm"
```

후자를 채택. `bootstrap/argo-cd/values.yaml`에 한 줄 추가.

- [ ] **Step 4: bootstrap README**

`bootstrap/README.md`:
```markdown
# Bootstrap 절차

이 디렉토리는 1회 적용되며, 이후 ArgoCD가 자기 자신을 포함한 모든 것을 관리한다.

## 신규 클러스터 부트스트랩

전제: k3s + Cilium 설치 완료, kubectl 컨텍스트 활성.

```bash
# 1. ArgoCD 설치
kustomize build --enable-helm bootstrap/argo-cd/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=300s -n argocd deployment/argocd-server

# 2. App-of-Apps 루트 적용 (이후 모든 sync는 ArgoCD가 자동)
kubectl apply -f bootstrap/root.yaml
```

## 검증

```bash
kubectl -n argocd get applications
# argo-cd, cluster-resources, root 가 보여야 함
# 잠시 후 platform-* (ApplicationSet 생성), apps-* 가 추가됨
```

## 재해 복구

Mac mini 폭발 시 위 절차 그대로 새 머신에서 실행 → 모든 상태 git에서 복원.
시크릿은 SealedSecrets 마스터키 (1Password) 복원 후 자동 풀림.
```

- [ ] **Step 5: 적용 + sync 검증**

```bash
kubectl apply -f bootstrap/root.yaml
sleep 30
argocd app list
argocd app sync root cluster-resources
```

기대: `root`, `argo-cd`, `cluster-resources` Application 모두 Healthy + Synced.

- [ ] **Step 6: 검증 commit**

```bash
echo "## Phase 3 acceptance" > tests/acceptance/phase-3-argocd.md
echo "- [x] argocd Application self-managed (Healthy)" >> tests/acceptance/phase-3-argocd.md
echo "- [x] cluster-resources Application syncs namespaces + PSA" >> tests/acceptance/phase-3-argocd.md
echo "- [x] root.yaml is the only manual apply for re-bootstrap" >> tests/acceptance/phase-3-argocd.md
git add bootstrap/ tests/acceptance/phase-3-argocd.md
git commit -m "feat(bootstrap): ArgoCD App-of-Apps self-managed"
git push
```

---
