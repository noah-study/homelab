# Security Controls Matrix

**Date**: 2026-04-28
**Source**: spec Section 3.4 (보안 표준 9 항목) + security review F1~F17 (17 finding)
**Owner**: `homelab-security-auditor` agent (자동 검증), 사용자 (수동 항목)
**Phase coverage**: 각 컨트롤이 도입되는 phase 명시. auditor는 "Phase ≤ N"의 컨트롤만 검증.

## 사용법

`homelab-security-auditor` dispatch 시 본 매트릭스를 Read하여:
1. 각 컨트롤의 Verify Command를 실행
2. 결과를 Expected와 비교 → PASS/FAIL/N/A/ERROR
3. FAIL은 Severity + Failure Action 보고

## 매트릭스

| ID | Control | Source | Phase | Verify Command | Expected | Severity if FAIL | Failure Action |
|---|---|---|---|---|---|---|---|
| **SC-01** | Image digest pin (no tag-only) | spec 3.4.2, sec F3 | 8 | `find apps -mindepth 3 -name kustomization.yaml ! -path '*/_base/*' \| xargs grep -L 'digest:\\|sha256:' \| wc -l` | `0` | Critical | builder/Phase 8 — Image Updater `update-strategy: digest` 확인 |
| **SC-02** | Cosign keyless signature on all GHCR images | spec 3.4.2, sec misc | 10 | `gh api /orgs/noah-study/packages/container --paginate -q '.[].name' \| while read pkg; do digest=$(gh api "/orgs/noah-study/packages/container/$pkg/versions" -q '.[0].name'); cosign verify --certificate-identity-regexp "^https://github.com/noah-study/.*/.github/workflows/reusable-build-push.yml@" --certificate-oidc-issuer https://token.actions.githubusercontent.com "ghcr.io/noah-study/$pkg@$digest" 2>&1 \| grep -q Verified \|\| echo "FAIL: $pkg@$digest"; done` | (no FAIL lines) | Critical | builder/Phase 10 — reusable workflow의 cosign sign 단계 검증 |
| **SC-03** | Cosign OIDC subject pinned to specific workflow | spec 3.4.2, sec F7 | deferred (~2026-06) | `grep -E 'certificate-identity' .github/workflows/reusable-build-push.yml` | exact path: `https://github.com/noah-study/.*/.github/workflows/reusable-build-push.yml@refs/heads/main` | High | reusable workflow의 verify 단계 regex 좁힘 |
| **SC-04** | PSA `restricted` on all app namespaces | spec 3.4.6, sec F12 | 6 | `kubectl get ns -o json \| jq -r '.items[] \| select(.metadata.name \| test("-(dev\|prod)$")) \| select(.metadata.labels."pod-security.kubernetes.io/enforce" != "restricted") \| .metadata.name'` | (empty) | High | builder — `_base/namespace.yaml` 또는 overlay에 라벨 추가 |
| **SC-05** | NetworkPolicy default-deny per app namespace | spec 3.4.6, sec F12 | 6, 11 | `for ns in $(kubectl get ns -o name \| grep -E -- '-(dev\|prod)$' \| cut -d/ -f2); do kubectl get netpol -n $ns -o name \| grep -q default-deny \|\| echo "MISSING: $ns"; done` | (empty) | High | builder/Phase 11 — `_base/networkpolicy.yaml` 확인 |
| **SC-06** | Hubble UI not externally exposed | sec F17 | 2 | `kubectl get ingress -A -o json \| jq -r '.items[] \| select(.spec.rules[].host \| test("hubble")) \| "\(.metadata.namespace)/\(.metadata.name)"'` | (empty) | Low | ingress 제거, port-forward만 |
| **SC-07** | GitHub App path restriction (Image Updater PRs only modify dev kustomization) | spec 3.4.4, sec F2 | 8 | `gh pr list --repo noah-study/homelab --search "author:app/noah-image-updater created:>$(date -u -v-7d +%Y-%m-%d)" --json files \| jq -e 'all(.[].files[]; .path \| test("^apps/[^/]+/overlays/macmini/dev/kustomization\\\\.yaml$"))'` | `true` | Critical | GitHub App 권한 + auto-merge workflow의 path filter 점검 |
| **SC-08** | Image Updater write-back uses git-pr (not direct commit) | spec 3.4.4, sec F2 | 8 | `kubectl get applicationset apps-dev -n argocd -o yaml \| grep -E 'write-back-method'` | `git:secret:` (with token, PR mode) | High | apps-dev ApplicationSet annotation 검증 |
| **SC-09** | Sealed Secrets master key backed up (1Password + USB) | spec 3.4.3, sec F4 | 4 | MANUAL: `op item get sealed-secrets-master-* --vault infra-keys` (1P CLI) | item exists | Critical | 사용자 백업 실행 (`./scripts/backup-sealed-secrets-key.sh`) |
| **SC-10** | Sealed Secrets master key rotation policy (≤6 months) | spec 3.4.3, sec F4 | 4 (정책), ad-hoc (실행) | `kubectl -n kube-system get secret -l sealedsecrets.bitnami.com/sealed-secrets-key=active -o json \| jq -r '.items[] \| .metadata.creationTimestamp' \| sort \| tail -1` | within last 180 days | Medium | `./scripts/rotate-sealed-secrets.sh` |
| **SC-11** | SealedSecrets namespace-scoped (`--scope namespace-wide`) | spec 3.4.3 | 4 | `find apps platform -name '*.sealed.yaml' \| xargs -I{} sh -c "yq '.spec.template.metadata.namespace' {} \| grep -q . \|\| echo MISSING: {}"` | (empty) | Medium | seal.sh wrapper에 `--scope namespace-wide` 강제 |
| **SC-12** | Cloudflare account hardware 2FA (FIDO2) | sec F9 | 1 | MANUAL: CF Dashboard → My Profile → Authentication → Security keys count > 0 | true | Critical | hardware key 등록 |
| **SC-13** | Cloudflare DNSSEC enabled for noah.dev | sec F9 | 1 | `dig +short noah.dev DNSKEY \| wc -l` | > 0 | High | CF Dashboard → DNS → DNSSEC enable + DS record 도메인 등록기관 등록 |
| **SC-14** | CAA records limit issuance to letsencrypt.org | sec F9 | 1 | `dig +short noah.dev CAA \| grep -c letsencrypt.org` | > 0 | High | CF Dashboard에 CAA 레코드 추가 |
| **SC-15** | Cloudflare API token scoped to single zone + DNS edit only | sec F9 | 1 | MANUAL: CF Dashboard → My Profile → API Tokens → 'cf-cert-manager-dns-token' permissions | DNS Edit, Zone:noah.dev only | High | 토큰 재발급 + scope 제한 |
| **SC-16** | OTel Alloy redaction processor active (Authorization, Cookie, secret patterns) | spec 3.4.7, sec F10 | 15 | `kubectl -n observability get configmap alloy-config -o yaml \| grep -A 5 'redact_logs\|redact_traces' \| grep -c -E 'authorization\|cookie\|password\|token\|api[_-]?key'` | >= 4 | High | `platform/grafana-alloy/config.alloy` 검증 |
| **SC-17** | gitleaks scan in reusable build-push workflow | spec 3.4.7 | 10 | `grep -c 'gitleaks' .github/workflows/reusable-build-push.yml` | >= 1 | Medium | reusable workflow에 gitleaks 단계 추가 |
| **SC-18** | ARC ephemeral runners (no _work persistence) | spec 3.4.1, sec F1, F8 | 9 | `kubectl get pod -n actions-runner-system -o json \| jq -r '.items[] \| select(.metadata.labels."actions.github.com/scale-set-name" != null) \| .spec.volumes[] \| select(.name == "work") \| .emptyDir // "MISSING"'` | (each line) {} (= emptyDir present) | High | scale-set values.yaml의 work volume = emptyDir 검증 |
| **SC-19** | Self-hosted runner outbound egress restricted (NetworkPolicy) | spec 3.4.1 | 9 | `kubectl get netpol runner-egress-allowlist -n actions-runner-system -o json \| jq '.spec.egress \| length'` | >= 1 | Medium | NetworkPolicy 적용 검증 |
| **SC-20** | Renovate actions pinned to commit SHA (not version tag) | spec 3.4.5, sec F5 | 16 | `grep -rE 'uses: [^@]+@v?[0-9]+\\.' .github/workflows/ \| grep -v '# v[0-9]'` | (empty — all should be SHA + comment) | High | Renovate `pinDigests: true` 확인 + 기존 workflow 재핀 |
| **SC-21** | Renovate auto-merge limited to allowlisted orgs | spec 3.4.5, sec F5 | 16 | `jq -r '.packageRules[] \| select(.automerge == true) \| .matchPackagePatterns // .matchPackageNames' renovate.json` | only `^(actions/\|docker/\|sigstore/\|aquasecurity/)` patterns | High | renovate.json 룰 검증 |
| **SC-22** | trivy-operator alert routing to Discord (Critical/High) | spec 3.4.9 | 14 | `gh secret list -R noah-study/homelab \| grep -q DISCORD_WEBHOOK_URL && grep -q 'CRITICAL\|HIGH' .github/workflows/trivy-alert.yml` | both true | Medium | Discord webhook secret + workflow 추가 |
| **SC-23** | GitHub Environments `production` with reviewer + 5-min wait | spec 3.4.8 | 13 | `gh api /repos/noah-study/homelab/environments/production -q '{reviewers: [.protection_rules[] \| select(.type == "required_reviewers")] \| length, wait_minutes: [.protection_rules[] \| select(.type == "wait_timer") \| .wait_timer][0]}'` | reviewers >= 1, wait_minutes >= 5 | High | GitHub UI에서 Environment 보호 룰 추가 (public repo면 Free에서 가능) |
| **SC-24** | Branch protection on main (PR + CODEOWNERS + status checks) | sec F3 | 16 | `gh api /repos/noah-study/homelab/branches/main/protection -q '{required_pr_reviews: .required_pull_request_reviews.required_approving_review_count, codeowners: .required_pull_request_reviews.require_code_owner_reviews, status_checks: .required_status_checks.contexts \| length}'` | reviews>=1, codeowners=true, status_checks>=1 | High | GitHub UI Settings → Branches |
| **SC-25** | macOS host security (FileVault on, Remote Login off) | sec F13 | 1 | MANUAL: `fdesetup status` → "FileVault is On" + `sudo systemsetup -getremotelogin` → "Off" | both | Medium | 호스트 OS 설정 |
| **SC-26** | OrbStack auto-update enabled | sec F13 | 1 | MANUAL: OrbStack → Settings → "Check for updates automatically" 체크 | true | Low | OrbStack Settings |

## 추가 컨트롤 (deferred — 도입 시 추가)

| ID | Control | 도입 시점 |
|---|---|---|
| **SC-27** | Sigstore Policy Controller — admission verify | ~2026-06 (spec 3.5) |
| **SC-28** | Velero backup automation | 운영 시작 후 30일 내 (spec 3.5) |
| **SC-29** | Falco runtime security | 사고 발생 시 (spec 3.5) |
| **SC-30** | ServiceAccount RBAC 분리 (agent별 KUBECONFIG) | Phase 2 install 시 (agents-design Section 6 Step 1.2) |

## Severity 기준

- **Critical** — 즉시 노출/침해 가능, 24시간 내 수정 필수
- **High** — 의미 있는 공격 표면, 7일 내 수정
- **Medium** — 위생 / 깊이 방어, 30일 내 수정
- **Low** — 정보성 / 모범 사례

## 자동 vs 수동 검증

매트릭스의 Verify Command 컬럼:
- **Bash one-liner** → auditor 자동 실행
- **MANUAL: ...** → 사용자 확인 요청 (auditor가 보고만)

각 phase 종료 시 auditor를 호출하면 매트릭스에서 `Phase ≤ N`인 컨트롤만 검증한다.
