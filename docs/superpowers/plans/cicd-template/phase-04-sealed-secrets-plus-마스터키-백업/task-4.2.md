# Task 4.2: 마스터키 백업 자동화 + 검증

**Phase**: 4 (Sealed Secrets + 마스터키 백업)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 4.2' 섹션

---


**Files:**
- Create: `scripts/backup-sealed-secrets-key.sh`
- Create: `scripts/rotate-sealed-secrets.sh`

- [ ] **Step 1: 백업 스크립트 작성**

`scripts/backup-sealed-secrets-key.sh`:
```bash
#!/usr/bin/env bash
# Sealed Secrets 마스터키를 추출해 두 곳에 저장한다.
# 사용법: ./scripts/backup-sealed-secrets-key.sh
# 필요: kubectl, op (1Password CLI), USB 마운트 경로
set -euo pipefail

DATE=$(date -u +%Y-%m-%dT%H-%M-%SZ)
TMPFILE=$(mktemp)
trap "rm -f $TMPFILE" EXIT

echo "[1/3] Extracting active sealed-secrets keys..."
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > "$TMPFILE"

KEYS=$(grep -c "^kind: Secret" "$TMPFILE" || true)
if [ "$KEYS" -lt 1 ]; then
  echo "ERROR: no active sealed-secrets key found"
  exit 1
fi
echo "  found $KEYS key(s)"

echo "[2/3] Saving to 1Password..."
op document create "$TMPFILE" \
  --title "sealed-secrets-master-${DATE}" \
  --vault "infra-keys" \
  --tags "sealed-secrets,backup"

echo "[3/3] Saving to offline USB (mount /Volumes/INFRA-BACKUP first)..."
USB_PATH="/Volumes/INFRA-BACKUP/sealed-secrets/${DATE}.yaml"
if [ ! -d "/Volumes/INFRA-BACKUP" ]; then
  echo "WARNING: /Volumes/INFRA-BACKUP not mounted. Skipping USB backup."
  echo "Mount the encrypted USB and rerun for USB backup."
else
  mkdir -p "$(dirname "$USB_PATH")"
  cp "$TMPFILE" "$USB_PATH"
  echo "  saved to $USB_PATH"
fi

echo "Done. Verify in 1Password and USB."
```

- [ ] **Step 2: 실행 권한 + 첫 실행**

```bash
chmod +x scripts/backup-sealed-secrets-key.sh
op signin   # 1Password CLI 로그인
./scripts/backup-sealed-secrets-key.sh
```

기대: 1Password에 `sealed-secrets-master-*` 문서 생성, USB 마운트 시 USB에도 저장.

- [ ] **Step 3: 회전 스크립트 (6개월 주기 cron 후보)**

`scripts/rotate-sealed-secrets.sh`:
```bash
#!/usr/bin/env bash
# Sealed Secrets 마스터키를 회전한다.
# 새 키가 active가 되고, 기존 키는 컨트롤러가 보관해 기존 SealedSecret도 풀 수 있음.
# 새로 봉인되는 시크릿은 새 키로.
# 사용법: ./scripts/rotate-sealed-secrets.sh
set -euo pipefail

echo "[1/3] Forcing key rotation (controller will generate new active key)..."
kubectl -n kube-system rollout restart deployment/sealed-secrets

echo "  waiting for rollout..."
kubectl -n kube-system rollout status deployment/sealed-secrets --timeout=120s

echo "[2/3] Backup new key..."
./scripts/backup-sealed-secrets-key.sh

echo "[3/3] List active keys (should include new one)"
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o name | sort

echo "Done. Existing SealedSecrets remain decryptable until controller GC removes old keys."
echo "To re-seal everything with the new key: ./scripts/reseal-all.sh (TODO: separate script)"
```

```bash
chmod +x scripts/rotate-sealed-secrets.sh
```
