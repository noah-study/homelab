# Task 5.4: ArgoCD Ingress 추가 (첫 외부 노출)

**Phase**: 5 (Cloudflare Tunnel + cert-manager + ingress-nginx)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 5.4' 섹션

---


**Files:**
- Create: `platform/argo-cd-ingress/kustomization.yaml`
- Create: `platform/argo-cd-ingress/ingress.yaml`

- [ ] **Step 1: ArgoCD UI ingress**

`platform/argo-cd-ingress/ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: HTTP   # values.yaml에 server.insecure: true
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.noah.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
  tls:
    - hosts: [argocd.noah.dev]
      secretName: noah-dev-wildcard-tls
```

`platform/argo-cd-ingress/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - ingress.yaml
```

(주: 와일드카드 cert는 `ingress-nginx` namespace에 있으나 `secret`을 cross-namespace 참조해야 함. 가장 간단한 해결: cert-manager의 `Certificate`를 namespace당 발급. 또는 reflector/replicator로 복제. 본 plan에서는 namespace당 Certificate를 추가하는 방식 채택 — 다음 step.)

- [ ] **Step 2: argocd namespace에 와일드카드 cert 발급**

`platform/argo-cd-ingress/certificate.yaml`:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: noah-dev-wildcard
  namespace: argocd
spec:
  secretName: noah-dev-wildcard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames: ["noah.dev", "*.noah.dev"]
```

`platform/argo-cd-ingress/kustomization.yaml` 업데이트:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - certificate.yaml
  - ingress.yaml
```

- [ ] **Step 3: 적용 + 검증**

```bash
kubectl apply -k platform/argo-cd-ingress/
kubectl wait --for=condition=ready --timeout=300s -n argocd certificate/noah-dev-wildcard

# Cloudflare DNS의 *.noah.dev CNAME이 tunnel을 가리킨다 가정
curl -fIs https://argocd.noah.dev | head -3
```

기대: HTTP 200, 인증서 valid.

- [ ] **Step 4: 검증 commit**

```bash
git add platform/cert-manager/ platform/ingress-nginx/ platform/cloudflared/ platform/argo-cd-ingress/
echo "## Phase 5 acceptance" > tests/acceptance/phase-5-networking.md
echo "- [x] cert-manager ClusterIssuer letsencrypt-prod ready" >> tests/acceptance/phase-5-networking.md
echo "- [x] *.noah.dev wildcard cert issued" >> tests/acceptance/phase-5-networking.md
echo "- [x] ingress-nginx serving traffic" >> tests/acceptance/phase-5-networking.md
echo "- [x] cloudflared tunnel HEALTHY" >> tests/acceptance/phase-5-networking.md
echo "- [x] https://argocd.noah.dev returns 200 with valid TLS" >> tests/acceptance/phase-5-networking.md
git add tests/acceptance/phase-5-networking.md
git commit -m "feat(platform): cert-manager + ingress-nginx + cloudflared tunnel"
git push
```

---
