# Task 1.3: Mac mini 호스트 준비

**Phase**: 1 (Foundation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 1.3' 섹션

---


**참조**: 보안 리뷰 F13 (macOS 호스트 + OrbStack VM escape)

- [ ] **Step 1: macOS 보안 설정**

```bash
# Remote Login 비활성
sudo systemsetup -setremotelogin off

# FileVault 활성 확인 (이미 활성이면 skip)
fdesetup status   # → "FileVault is On"
# 비활성 상태면: 시스템 설정 → 개인정보 보호 및 보안 → FileVault → 켜기
```

- [ ] **Step 2: OrbStack 설치**

```bash
brew install --cask orbstack
```

설치 후 OrbStack 실행. Settings → "Check for updates automatically" 활성. Settings → File sharing 최소화 (필요 디렉토리만).

- [ ] **Step 3: Linux VM 생성 (Ubuntu 24.04 ARM64)**

```bash
orb create ubuntu macmini
orb shell macmini
# VM 안에서:
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get install -y curl jq vim git
```

- [ ] **Step 4: kubectl, helm, kubeseal, k9s 설치 (호스트 macOS)**

```bash
brew install kubectl helm kubeseal k9s sops age yq
```

- [ ] **Step 5: GitHub CLI 인증 + bats 테스트 프레임워크**

```bash
gh auth status   # 이미 로그인되어 있어야 함, 아니면: gh auth login
brew install bats-core
```

- [ ] **Step 6: 검증 commit**

```bash
echo "## Phase 1 acceptance" > tests/acceptance/phase-1-foundation.md
echo "- [x] git repo on GitHub" >> tests/acceptance/phase-1-foundation.md
echo "- [x] CF DNSSEC enabled, CAA records present" >> tests/acceptance/phase-1-foundation.md
echo "- [x] CF API token + Tunnel token in 1Password" >> tests/acceptance/phase-1-foundation.md
echo "- [x] OrbStack VM 'macmini' running" >> tests/acceptance/phase-1-foundation.md
git add tests/acceptance/phase-1-foundation.md
git commit -m "test: phase 1 acceptance checklist"
git push
```

---
