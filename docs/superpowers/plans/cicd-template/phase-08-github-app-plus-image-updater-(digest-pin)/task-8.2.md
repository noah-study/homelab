# Task 8.2: argocd-image-updater 설치

**Phase**: 8 (GitHub App + Image Updater (digest pin))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 8.2' 섹션

---


**Files:**
- Create: `platform/argocd-image-updater/kustomization.yaml`
- Create: `platform/argocd-image-updater/values.yaml`
- Create: `platform/argocd-image-updater/github-app-secret.sealed.yaml` (위에서 생성됨)

- [ ] **Step 1: Helm wrapper**

`platform/argocd-image-updater/values.yaml`:
```yaml
fullnameOverride: argocd-image-updater
config:
  argocd:
    grpcWeb: false
    serverAddress: argocd-server.argocd.svc.cluster.local:443
    insecure: true
    plaintext: false
  registries:
    - name: GitHub Container Registry
      api_url: https://ghcr.io
      prefix: ghcr.io
      ping: yes
      credentials: pullsecret:argocd/ghcr-pull-secret
      default: true
  gitCommitUser: argocd-image-updater
  gitCommitMail: argocd-image-updater@noreply.github.com
  logLevel: info
metrics:
  enabled: true
  serviceMonitor:
    enabled: false   # Phase 15
```

`platform/argocd-image-updater/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - github-app-secret.sealed.yaml
helmCharts:
  - name: argocd-image-updater
    repo: https://argoproj.github.io/argo-helm
    version: 0.11.2
    releaseName: argocd-image-updater
    namespace: argocd
    valuesFile: values.yaml
```

- [ ] **Step 2: GHCR pull secret 봉인 (image-updater가 GHCR API 호출)**

```bash
# Personal Access Token 또는 (더 안전) GitHub App과 동일 자격으로 GHCR 접근
# 간단하게 read:packages 권한의 fine-grained PAT를 1Password에 두고 사용
GHCR_TOKEN=$(op read "op://infra-keys/ghcr-read-pat/credential")

kubectl create secret docker-registry ghcr-pull-secret \
  --namespace=argocd \
  --docker-server=ghcr.io \
  --docker-username=noah \
  --docker-password="$GHCR_TOKEN" \
  --dry-run=client -o yaml \
| kubeseal --controller-name sealed-secrets --controller-namespace kube-system --format yaml \
  > platform/argocd-image-updater/ghcr-pull-secret.sealed.yaml
```

`platform/argocd-image-updater/kustomization.yaml`에 추가:
```yaml
resources:
  - github-app-secret.sealed.yaml
  - ghcr-pull-secret.sealed.yaml
```

- [ ] **Step 3: ArgoCD가 sync (platform ApplicationSet 자동 등록)**

```bash
git add platform/argocd-image-updater/
git commit -m "feat(platform): argocd-image-updater with GitHub App + digest strategy"
git push

argocd app sync platform-argocd-image-updater
kubectl wait --for=condition=available --timeout=120s -n argocd deployment/argocd-image-updater
kubectl logs -n argocd deployment/argocd-image-updater --tail=50
```

기대: `argocd-image-updater` 로그에 "Starting image updater", 인증 성공.
