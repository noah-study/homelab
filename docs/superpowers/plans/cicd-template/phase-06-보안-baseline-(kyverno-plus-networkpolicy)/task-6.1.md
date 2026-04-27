# Task 6.1: Kyverno 설치 + PSA exception audit policy

**Phase**: 6 (보안 Baseline (Kyverno + NetworkPolicy))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 6.1' 섹션

---


**Files:**
- Create: `platform/kyverno/kustomization.yaml`
- Create: `platform/kyverno/values.yaml`
- Create: `platform/kyverno/policies.yaml`

- [ ] **Step 1: Helm wrapper**

`platform/kyverno/values.yaml`:
```yaml
admissionController:
  replicas: 1
backgroundController:
  replicas: 1
cleanupController:
  replicas: 1
reportsController:
  replicas: 1
```

`platform/kyverno/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kyverno
resources:
  - namespace.yaml
  - policies.yaml
helmCharts:
  - name: kyverno
    repo: https://kyverno.github.io/kyverno/
    version: 3.3.4
    releaseName: kyverno
    namespace: kyverno
    valuesFile: values.yaml
```

`platform/kyverno/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kyverno
  labels:
    pod-security.kubernetes.io/enforce: privileged   # Kyverno 자체는 admission webhook
```

- [ ] **Step 2: PSA audit policy + image digest pin policy**

`platform/kyverno/policies.yaml`:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: audit-psa-exceptions
  annotations:
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Audit   # 차단 안 함, 보고만
  rules:
    - name: warn-non-restricted-namespace
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        message: "Namespace must have pod-security.kubernetes.io/enforce=restricted"
        pattern:
          metadata:
            labels:
              pod-security.kubernetes.io/enforce: "restricted"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
  annotations:
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Audit   # Phase 18에서 Enforce로 전환 검토
  background: true
  rules:
    - name: pods-must-use-digest
      match:
        any:
          - resources:
              kinds: [Pod]
              namespaces: ["!kube-system", "!cert-manager", "!ingress-nginx"]
      validate:
        message: "Image must be referenced by digest (sha256:...)"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-network
  annotations:
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-host-network
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "hostNetwork is not allowed"
        pattern:
          spec:
            =(hostNetwork): "false"
```

- [ ] **Step 3: 적용 + 검증**

```bash
kustomize build --enable-helm platform/kyverno/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=300s -n kyverno deployment/kyverno-admission-controller

# Policy report 확인
kubectl get policyreports --all-namespaces
```

기대: kyverno pods Running, ClusterPolicy 3개 등록.
