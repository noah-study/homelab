# Task 14.1: trivy-operator 설치

**Phase**: 14 (Trivy-Operator + Alert Routing)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 14.1' 섹션

---


**Files:**
- Create: `platform/trivy-operator/kustomization.yaml`
- Create: `platform/trivy-operator/values.yaml`

- [ ] **Step 1: Helm wrapper**

`platform/trivy-operator/values.yaml`:
```yaml
operator:
  scannerReportTTL: "168h"   # 7일 retention
  scanJobTimeout: "10m"
  vulnerabilityScannerEnabled: true
  configAuditScannerEnabled: true
  rbacAssessmentScannerEnabled: true
  exposedSecretScannerEnabled: true
  metricsVulnIdEnabled: true
trivy:
  ignoreUnfixed: true
  severity: "CRITICAL,HIGH,MEDIUM"
serviceMonitor:
  enabled: false   # Phase 15
```

`platform/trivy-operator/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: trivy-system
resources:
  - namespace.yaml
helmCharts:
  - name: trivy-operator
    repo: https://aquasecurity.github.io/helm-charts
    version: 0.24.1
    releaseName: trivy-operator
    namespace: trivy-system
    valuesFile: values.yaml
```

`platform/trivy-operator/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: trivy-system
  labels:
    pod-security.kubernetes.io/enforce: baseline
```

- [ ] **Step 2: ArgoCD sync (platform ApplicationSet 자동 등록)**

```bash
git add platform/trivy-operator/
git commit -m "feat(platform): trivy-operator for vuln scanning"
git push
argocd app sync platform-trivy-operator
```
