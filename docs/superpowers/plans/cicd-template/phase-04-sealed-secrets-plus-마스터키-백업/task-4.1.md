# Task 4.1: sealed-secrets 컨트롤러 설치

**Phase**: 4 (Sealed Secrets + 마스터키 백업)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 4.1' 섹션

---


**Files:**
- Create: `platform/sealed-secrets/kustomization.yaml`
- Create: `platform/sealed-secrets/values.yaml`

- [ ] **Step 1: Helm wrapper 작성**

`platform/sealed-secrets/values.yaml`:
```yaml
fullnameOverride: sealed-secrets
namespace: kube-system   # 마스터키는 kube-system에 둠 (관례)
metrics:
  serviceMonitor:
    enabled: false   # Phase 15에서 별도 정의
```

`platform/sealed-secrets/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: sealed-secrets
    repo: https://bitnami-labs.github.io/sealed-secrets
    version: 2.16.2
    releaseName: sealed-secrets
    namespace: kube-system
    valuesFile: values.yaml
```

(이 디렉토리는 Phase 7의 platform ApplicationSet이 자동으로 ArgoCD Application으로 등록할 예정. 지금은 ArgoCD가 sync 안 함 — 다음 단계에서 임시 직접 설치)

- [ ] **Step 2: 임시로 직접 설치 (Phase 7 전까지)**

```bash
kustomize build --enable-helm platform/sealed-secrets/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=120s -n kube-system deployment/sealed-secrets
```
