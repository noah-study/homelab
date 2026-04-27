# Task 3.2: ArgoCD 부트스트랩 적용

**Phase**: 3 (ArgoCD Bootstrap (자기관리))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 3.2' 섹션

---


- [ ] **Step 1: kubectl + kustomize로 적용**

```bash
kustomize build --enable-helm bootstrap/argo-cd/ | kubectl apply -f -
kubectl wait --for=condition=available --timeout=300s -n argocd deployment/argocd-server
```

- [ ] **Step 2: admin 비밀번호 추출 + 1Password 저장**

```bash
ARGO_PW=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)
echo "Admin password (save to 1Password as 'argocd-initial-admin'): $ARGO_PW"
```

1Password에 저장 후:
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
# 브라우저: https://localhost:8080 (admin / 위 비밀번호)
```

기대: ArgoCD UI 접근 가능.

- [ ] **Step 3: argocd CLI 로그인**

```bash
brew install argocd
argocd login localhost:8080 --username admin --password "$ARGO_PW" --insecure
argocd account update-password   # 새 비밀번호 설정 후 1Password 업데이트
```
