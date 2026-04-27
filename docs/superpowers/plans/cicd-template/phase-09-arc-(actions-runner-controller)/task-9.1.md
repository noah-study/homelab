# Task 9.1: ARC 컨트롤러 + Runner Scale Set GitHub App

**Phase**: 9 (ARC (Actions Runner Controller))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 9.1' 섹션

---


- [ ] **Step 1: ARC용 GitHub App 생성**

GitHub Settings → Developer settings → GitHub Apps → New:
- Name: `noah-arc-runner`
- Permissions:
  - Repository: Actions: Read & Write, Administration: Read & Write, Checks: Read, Metadata: Read
  - Organization: Self-hosted runners: Read & Write
- Install on: noah org

App ID, Installation ID, private key를 1Password에 저장 (`gh-app-arc-key`).

- [ ] **Step 2: ARC controller Helm wrapper**

`platform/actions-runner-controller/values.yaml`:
```yaml
# gha-runner-scale-set-controller chart
fullnameOverride: arc-controller
flags:
  logLevel: info
  logFormat: text
```

`platform/actions-runner-controller/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: actions-runner-system
resources:
  - github-app-secret.sealed.yaml
  - runner-scale-set.yaml
helmCharts:
  - name: gha-runner-scale-set-controller
    repo: oci://ghcr.io/actions/actions-runner-controller-charts
    version: 0.10.1
    releaseName: arc
    namespace: actions-runner-system
    valuesFile: values.yaml
```

- [ ] **Step 3: GitHub App 자격증명 봉인**

```bash
ARC_APP_ID=$(op read "op://infra-keys/gh-app-arc-key/app-id")
ARC_INSTALL_ID=$(op read "op://infra-keys/gh-app-arc-key/installation-id")
ARC_PRIVATE_KEY=$(op read "op://infra-keys/gh-app-arc-key/private-key")

./scripts/seal.sh actions-runner-system arc-github-app \
  github_app_id="$ARC_APP_ID" \
  github_app_installation_id="$ARC_INSTALL_ID" \
  github_app_private_key="$ARC_PRIVATE_KEY" \
  > platform/actions-runner-controller/github-app-secret.sealed.yaml
```
