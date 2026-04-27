# Task 5.2: ingress-nginx 설치

**Phase**: 5 (Cloudflare Tunnel + cert-manager + ingress-nginx)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 5.2' 섹션

---


**Files:**
- Create: `platform/ingress-nginx/kustomization.yaml`
- Create: `platform/ingress-nginx/values.yaml`

- [ ] **Step 1: Helm wrapper**

`platform/ingress-nginx/values.yaml`:
```yaml
controller:
  replicaCount: 1
  service:
    type: ClusterIP   # 외부 노출은 cloudflared가 담당
  ingressClassResource:
    default: true
  config:
    use-forwarded-headers: "true"
    enable-real-ip: "true"
  metrics:
    enabled: true   # Phase 15 Alloy가 수집
    serviceMonitor:
      enabled: false   # Phase 15에서 활성
  podSecurityContext:
    runAsNonRoot: true
    runAsUser: 101
  containerSecurityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
      add: [NET_BIND_SERVICE]
```

`platform/ingress-nginx/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ingress-nginx
helmCharts:
  - name: ingress-nginx
    repo: https://kubernetes.github.io/ingress-nginx
    version: 4.11.3
    releaseName: ingress-nginx
    namespace: ingress-nginx
    valuesFile: values.yaml
```

- [ ] **Step 2: 임시 직접 적용**

```bash
kustomize build --enable-helm platform/ingress-nginx/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=180s -n ingress-nginx deployment/ingress-nginx-controller
```
