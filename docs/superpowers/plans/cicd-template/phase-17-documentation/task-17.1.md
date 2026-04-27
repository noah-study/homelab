# Task 17.1: docs/onboarding.md

**Phase**: 17 (Documentation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 17.1' 섹션

---


**Files:**
- Create: `docs/onboarding.md`

- [ ] **Step 1: 작성**

`docs/onboarding.md`:
```markdown
# 새 서비스 5분 onboarding

## 사전 조건
- macOS 호스트에서 kubectl, gh, jq, op (1Password CLI) 설치
- KUBECONFIG=~/.kube/macmini.kubeconfig
- gh auth status OK

## 절차

```bash
./scripts/new-app.sh <service-name>
git push   # infra 레포에 PR 생성됨

# PR 머지 후 (CODEOWNERS 본인 강제 리뷰):
# 자동 진행 모니터:
argocd app list | grep <service-name>
gh run watch                     # ARC runner build/sign 모니터
gh pr list --label image-updater  # Image Updater PR 자동 머지
```

## 검증

```bash
curl -fs https://<service-name>.dev.noah.dev/healthz
```

## prod 승급

```bash
gh workflow run promote.yml -f service=<service-name> -f sha=<short-sha>
# GitHub Actions UI에서 production 환경 승인 (5분 wait)
# PR 자동 생성 → 본인 리뷰 → 머지 → ArgoCD prod sync
```

## 시크릿 추가

```bash
./scripts/seal.sh <service-name>-dev <secret-name> KEY=value [KEY=value...]
# 출력을 apps/<service-name>/overlays/macmini/dev/secrets.sealed.yaml에 append
git add apps/<service-name>/overlays/macmini/dev/secrets.sealed.yaml
git commit -m "feat(<service-name>): add <secret-name> for dev"
git push
```

## Dockerfile / 코드 스타일
- 멀티 스테이지 빌드 권장
- `cgr.dev/chainguard/static:latest` 또는 distroless 사용
- `USER 65532` 명시
- `EXPOSE 8080` 만 (다른 포트 쓰면 _base 패치 필요)
- `/healthz`, `/ready` 엔드포인트 필수
```
