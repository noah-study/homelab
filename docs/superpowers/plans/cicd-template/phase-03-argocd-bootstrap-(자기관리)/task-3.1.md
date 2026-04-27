# Task 3.1: ArgoCD 초기 설치 매니페스트 작성

**Phase**: 3 (ArgoCD Bootstrap (자기관리))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 3.1' 섹션

---


**Files:**
- Create: `bootstrap/argo-cd/namespace.yaml`
- Create: `bootstrap/argo-cd/values.yaml`
- Create: `bootstrap/argo-cd/kustomization.yaml`

- [ ] **Step 1: namespace 매니페스트**

`bootstrap/argo-cd/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

- [ ] **Step 2: ArgoCD Helm values**

`bootstrap/argo-cd/values.yaml` (chart 7.x 기준 기본 값 + 핵심 override만):
```yaml
global:
  domain: argocd.noah.dev
configs:
  params:
    server.insecure: true   # ingress가 TLS 종료 (Phase 5)
  cm:
    timeout.reconciliation: 180s
  rbac:
    policy.default: role:readonly
server:
  ingress:
    enabled: false   # Phase 5에서 별도 ingress 추가
controller:
  replicas: 1   # 200 App 도달 시 sharding (scaling playbook)
applicationSet:
  replicas: 1
```

- [ ] **Step 3: Kustomize HelmChart wrapper**

`bootstrap/argo-cd/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
helmCharts:
  - name: argo-cd
    repo: https://argoproj.github.io/argo-helm
    version: 7.7.13
    releaseName: argocd
    namespace: argocd
    valuesFile: values.yaml
```
