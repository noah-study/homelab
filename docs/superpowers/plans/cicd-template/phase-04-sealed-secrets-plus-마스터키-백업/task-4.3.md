# Task 4.3: seal.sh wrapper + bats 테스트

**Phase**: 4 (Sealed Secrets + 마스터키 백업)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 4.3' 섹션

---


**Files:**
- Create: `scripts/seal.sh`
- Create: `tests/seal.bats`

- [ ] **Step 1: seal.sh wrapper**

`scripts/seal.sh`:
```bash
#!/usr/bin/env bash
# 시크릿을 namespace-scoped sealed secret으로 봉인한다.
# 사용법: ./scripts/seal.sh <namespace> <secret-name> <key>=<value> [<key>=<value> ...]
# 출력: stdout에 sealed YAML
set -euo pipefail

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <namespace> <secret-name> <key>=<value> [<key>=<value>...]"
  exit 1
fi

NAMESPACE="$1"
NAME="$2"
shift 2

# kubectl create secret을 dry-run으로 (encoded yaml 생성)
ARGS=()
for kv in "$@"; do
  ARGS+=("--from-literal=$kv")
done

kubectl create secret generic "$NAME" \
  --namespace="$NAMESPACE" \
  --dry-run=client \
  -o yaml \
  "${ARGS[@]}" \
| kubeseal \
    --controller-name sealed-secrets \
    --controller-namespace kube-system \
    --scope namespace-wide \
    --format yaml
```

```bash
chmod +x scripts/seal.sh
```

- [ ] **Step 2: bats 테스트**

`tests/seal.bats`:
```bash
#!/usr/bin/env bats

setup() {
  export KUBECONFIG=~/.kube/macmini.kubeconfig
}

@test "seal.sh requires at least 3 args" {
  run ./scripts/seal.sh
  [ "$status" -ne 0 ]
  [[ "$output" == *"Usage:"* ]]
}

@test "seal.sh produces SealedSecret yaml" {
  run ./scripts/seal.sh test-namespace test-secret foo=bar
  [ "$status" -eq 0 ]
  [[ "$output" == *"kind: SealedSecret"* ]]
  [[ "$output" == *"namespace: test-namespace"* ]]
}

@test "seal.sh with multiple keys produces SealedSecret with both" {
  run ./scripts/seal.sh test-ns secret1 a=1 b=2
  [ "$status" -eq 0 ]
  # encryptedData 키 2개
  count=$(echo "$output" | grep -c "^    [a-z]:" || true)
  [ "$count" -ge 2 ]
}
```

- [ ] **Step 3: bats 테스트 실행**

```bash
bats tests/seal.bats
```

기대: 3 tests, 0 failures.

- [ ] **Step 4: commit**

```bash
git add platform/sealed-secrets/ scripts/seal.sh scripts/backup-sealed-secrets-key.sh scripts/rotate-sealed-secrets.sh tests/seal.bats
echo "## Phase 4 acceptance" > tests/acceptance/phase-4-sealed-secrets.md
echo "- [x] sealed-secrets controller running" >> tests/acceptance/phase-4-sealed-secrets.md
echo "- [x] master key backed up to 1Password + USB" >> tests/acceptance/phase-4-sealed-secrets.md
echo "- [x] seal.sh wrapper works (3 bats tests pass)" >> tests/acceptance/phase-4-sealed-secrets.md
echo "- [x] rotation script ready (manual run)" >> tests/acceptance/phase-4-sealed-secrets.md
git add tests/acceptance/phase-4-sealed-secrets.md
git commit -m "feat(platform): sealed-secrets + master key backup workflow"
git push
```

---
