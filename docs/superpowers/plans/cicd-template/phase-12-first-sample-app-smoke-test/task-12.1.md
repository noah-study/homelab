# Task 12.1: sample-app 코드 레포 + 첫 배포

**Phase**: 12 (First Sample App Smoke Test)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 12.1' 섹션

---


- [ ] **Step 1: new-app.sh 실행**

```bash
./scripts/new-app.sh sample-app
git push
```

- [ ] **Step 2: 최소한의 Go HTTP 서버 작성 (sample-app 레포)**

```bash
git clone https://github.com/noah-study/sample-app /tmp/sample-app
cd /tmp/sample-app

cat > main.go <<'EOF'
package main
import (
  "fmt"; "net/http"; "os"
)
func main() {
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello from %s\n", os.Getenv("OTEL_SERVICE_NAME"))
  })
  http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
  })
  http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
  })
  http.ListenAndServe(":8080", nil)
}
EOF

cat > go.mod <<'EOF'
module sample-app
go 1.22
EOF

cat > Dockerfile <<'EOF'
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY go.mod ./
RUN go mod download || true
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server main.go

FROM cgr.dev/chainguard/static:latest
COPY --from=build /app/server /server
USER 65532
EXPOSE 8080
ENTRYPOINT ["/server"]
EOF

git checkout -b develop
git add .
git commit -m "feat: initial sample app"
git push -u origin develop
```

- [ ] **Step 3: CI 파이프라인 트리거 + 결과 관찰**

GitHub Actions UI → noah-study/sample-app → `ci` 워크플로우 실행 모니터.

기대 (~3-5분):
- test job: PASS (no tests, skipped)
- build job: GHCR push 3 tags + Cosign 서명 + verify
- 출력에 image digest 표시

GHCR 확인:
```bash
gh api /user/packages/container/sample-app/versions | jq '.[].metadata.container.tags'
# ["sha-abcd123", "develop-abcd123", "develop-latest", ...]
```

- [ ] **Step 4: ArgoCD가 ApplicationSet으로 자동 등록**

```bash
sleep 60
argocd app list | grep sample-app
# dev-sample-app    Synced    Healthy
```

- [ ] **Step 5: Image Updater가 dev kustomization에 PR**

GitHub UI → noah-study/homelab → Pull requests:
- Title: `image-updater: bump sample-app to digest sha256:...`
- Author: `noah-image-updater[bot]`
- Path: `apps/sample-app/overlays/macmini/dev/kustomization.yaml`만 수정

`auto-merge-image-updater.yml`이 자동 머지 → main 머지 → ArgoCD가 dev sync → pod 기동.

- [ ] **Step 6: dev 라이브 검증**

```bash
sleep 60
curl -fs https://sample-app.dev.noah.dev/healthz
# 200 OK

curl -fs https://sample-app.dev.noah.dev/
# hello from sample-app
```

- [ ] **Step 7: 시간 측정 + commit**

```bash
echo "## Phase 12 acceptance" > tests/acceptance/phase-12-smoke.md
echo "- [x] new-app.sh sample-app: $(date)" >> tests/acceptance/phase-12-smoke.md
echo "- [x] First push to develop → dev live in ≤ 10 min" >> tests/acceptance/phase-12-smoke.md
echo "- [x] Image Updater PR auto-merged" >> tests/acceptance/phase-12-smoke.md
echo "- [x] Cosign verify PASS in build logs" >> tests/acceptance/phase-12-smoke.md
echo "- [x] https://sample-app.dev.noah.dev returns hello" >> tests/acceptance/phase-12-smoke.md
git add tests/acceptance/phase-12-smoke.md
git commit -m "test: phase 12 sample-app smoke test passed"
git push
```

---
