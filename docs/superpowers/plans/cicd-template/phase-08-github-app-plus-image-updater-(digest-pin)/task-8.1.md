# Task 8.1: GitHub App 생성 (수동 UI)

**Phase**: 8 (GitHub App + Image Updater (digest pin))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 8.1' 섹션

---


- [ ] **Step 1: GitHub App 생성**

GitHub Settings → Developer settings → GitHub Apps → New GitHub App:
- Name: `noah-image-updater`
- Homepage URL: `https://github.com/noah-study/homelab`
- Webhook: 비활성
- Permissions:
  - Repository permissions:
    - Contents: Read & Write
    - Pull requests: Read & Write
    - Metadata: Read
- Where can this app be installed: Only on this account

생성 후:
- App ID 기록 (예: 1234567)
- Generate a private key → `.pem` 파일 다운로드 → 1Password에 저장 (`gh-app-image-updater-key`)
- Install App → noah-study/homelab 레포에만 설치, Installation ID 기록

- [ ] **Step 2: GitHub App 자격증명을 SealedSecret으로**

```bash
GH_APP_PRIVATE_KEY=$(op read "op://infra-keys/gh-app-image-updater-key/private-key")
GH_APP_ID=1234567   # 위에서 기록한 값
GH_APP_INSTALL_ID=12345678   # 위에서 기록한 값

# argocd-image-updater는 Bearer token 형식을 기대 — argocd-image-updater 용 자체 secret format
# write-back-method: git:secret:... 로 git 토큰을 SealedSecret에서 참조

# 1) GitHub App → installation token 변환은 argocd-image-updater가 내부 처리
#    private key + app id + installation id를 secret에 저장
./scripts/seal.sh argocd image-updater-github-token \
  githubAppID="$GH_APP_ID" \
  githubAppInstallationID="$GH_APP_INSTALL_ID" \
  githubAppPrivateKey="$GH_APP_PRIVATE_KEY" \
  > platform/argocd-image-updater/github-app-secret.sealed.yaml
```
