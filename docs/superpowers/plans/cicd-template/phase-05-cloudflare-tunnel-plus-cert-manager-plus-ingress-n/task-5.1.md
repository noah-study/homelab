# Task 5.1: cert-manager 설치 + Cloudflare DNS01 ClusterIssuer

**Phase**: 5 (Cloudflare Tunnel + cert-manager + ingress-nginx)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 5.1' 섹션

---


**Files:**
- Create: `platform/cert-manager/kustomization.yaml`
- Create: `platform/cert-manager/values.yaml`
- Create: `platform/cert-manager/cluster-issuer.yaml`
- Create: `platform/cert-manager/cf-token.sealed.yaml`

- [ ] **Step 1: Helm wrapper**

`platform/cert-manager/values.yaml`:
```yaml
installCRDs: true
global:
  leaderElection:
    namespace: cert-manager
prometheus:
  enabled: false
```

`platform/cert-manager/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: cert-manager
resources:
  - cf-token.sealed.yaml
  - cluster-issuer.yaml
helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    version: v1.16.2
    releaseName: cert-manager
    namespace: cert-manager
    valuesFile: values.yaml
```

- [ ] **Step 2: Cloudflare API 토큰을 SealedSecret으로 봉인**

```bash
# 1Password에서 토큰 추출 (op CLI 또는 수동)
CF_TOKEN=$(op read "op://infra-keys/cf-cert-manager-dns-token/credential")

./scripts/seal.sh cert-manager cloudflare-api-token api-token="$CF_TOKEN" \
  > platform/cert-manager/cf-token.sealed.yaml
```

`platform/cert-manager/cf-token.sealed.yaml` (확인 — git에 안전):
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
spec:
  encryptedData:
    api-token: AgB...   # 실제 봉인 값
  template:
    metadata:
      name: cloudflare-api-token
      namespace: cert-manager
    type: Opaque
```

- [ ] **Step 3: ClusterIssuer 정의**

`platform/cert-manager/cluster-issuer.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: noah@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - "noah.dev"
---
# 와일드카드 인증서를 ingress-nginx namespace에 미리 발급 (모든 서비스 공유)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: noah-dev-wildcard
  namespace: ingress-nginx
spec:
  secretName: noah-dev-wildcard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "noah.dev"
    - "*.noah.dev"
    - "*.dev.noah.dev"
```

- [ ] **Step 4: 임시 직접 적용 (Phase 7 ApplicationSet 적용 전까지)**

```bash
kustomize build --enable-helm platform/cert-manager/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=300s -n cert-manager deployment/cert-manager
kubectl wait --for=condition=available --timeout=120s -n cert-manager deployment/cert-manager-webhook

# 인증서 발급 (DNS01 propagation 대기 ~2분)
kubectl wait --for=condition=ready --timeout=300s -n ingress-nginx certificate/noah-dev-wildcard
```

기대: `kubectl get cert -n ingress-nginx` → READY=True.
