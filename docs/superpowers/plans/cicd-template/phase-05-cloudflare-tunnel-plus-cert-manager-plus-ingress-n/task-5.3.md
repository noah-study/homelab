# Task 5.3: cloudflared (Tunnel) 배포

**Phase**: 5 (Cloudflare Tunnel + cert-manager + ingress-nginx)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 5.3' 섹션

---


**Files:**
- Create: `platform/cloudflared/kustomization.yaml`
- Create: `platform/cloudflared/values.yaml`
- Create: `platform/cloudflared/tunnel-token.sealed.yaml`

- [ ] **Step 1: Tunnel 라우팅 룰 등록 (Cloudflare UI)**

Cloudflare Dashboard → Zero Trust → Networks → Tunnels → `macmini-home` → Public Hostnames:
- Subdomain: `*` Domain: `noah.dev` → Service: `https://ingress-nginx-controller.ingress-nginx.svc.cluster.local:443`
- Subdomain: `*` Domain: `dev.noah.dev` → 동일

(또는 단일 룰 + ingress-nginx host 분기 — 위가 더 명시적)

- [ ] **Step 2: Tunnel 토큰 SealedSecret**

```bash
CF_TUNNEL_TOKEN=$(op read "op://infra-keys/cf-tunnel-macmini-token/credential")
./scripts/seal.sh platform cloudflared-token token="$CF_TUNNEL_TOKEN" \
  > platform/cloudflared/tunnel-token.sealed.yaml
```

- [ ] **Step 3: cloudflared Helm wrapper**

`platform/cloudflared/values.yaml`:
```yaml
fullnameOverride: cloudflared
image:
  repository: cloudflare/cloudflared
replicaCount: 1   # HA 원하면 2
existingSecret: cloudflared-token   # 위에서 봉인한 SealedSecret
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 65532
containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: true
```

`platform/cloudflared/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: platform
resources:
  - tunnel-token.sealed.yaml
helmCharts:
  - name: cloudflare-tunnel
    repo: https://cloudflare.github.io/helm-charts
    version: 0.3.2
    releaseName: cloudflared
    namespace: platform
    valuesFile: values.yaml
```

- [ ] **Step 4: 임시 직접 적용 + 검증**

```bash
kustomize build --enable-helm platform/cloudflared/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=120s -n platform deployment/cloudflared
```

Cloudflare Dashboard → Zero Trust → Networks → Tunnels → `macmini-home`이 **HEALTHY** 상태.
