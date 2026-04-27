# Task 1.2: Cloudflare 계정 보안 강화 (수동 UI)

**Phase**: 1 (Foundation)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 1.2' 섹션

---


**참조**: 보안 리뷰 F9 (DNS 하이재킹 / DNSSEC / CAA 부재)

- [ ] **Step 1: Hardware 2FA 등록**

Cloudflare Dashboard → My Profile → Authentication → Add Security Key (FIDO2/WebAuthn). YubiKey 또는 Titan Security Key 권장.

검증: 로그아웃 후 재로그인 시 하드웨어 키 요구.

- [ ] **Step 2: DNSSEC 활성화**

Cloudflare Dashboard → noah.dev → DNS → Settings → DNSSEC → Enable. 표시되는 DS record를 도메인 등록기관(예: Namecheap/Gandi)에 등록.

검증: `dig noah.dev DNSKEY +short` 응답 존재.

- [ ] **Step 3: CAA 레코드 추가**

Cloudflare DNS에 다음 두 레코드 추가:
```
Type: CAA  Name: noah.dev  Value: 0 issue "letsencrypt.org"
Type: CAA  Name: noah.dev  Value: 0 iodef "mailto:noah@example.com"
```

검증: `dig noah.dev CAA +short` → 두 레코드 표시.

- [ ] **Step 4: cert-manager용 API 토큰 생성**

Cloudflare Dashboard → My Profile → API Tokens → Create Token → "Edit zone DNS" 템플릿:
- Permissions: Zone → DNS → Edit
- Zone Resources: Include → Specific zone → noah.dev
- Client IP Address Filtering: (skip, dynamic IP)
- TTL: (없음)

토큰 값을 1Password에 저장 (이름: `cf-cert-manager-dns-token`). **이 토큰은 phase 5에서 사용**.

- [ ] **Step 5: Tunnel 토큰 발급**

Cloudflare Dashboard → Zero Trust → Networks → Tunnels → Create a tunnel:
- Name: `macmini-home`
- Type: Cloudflared

토큰 값을 1Password에 저장 (이름: `cf-tunnel-macmini-token`). **Phase 5에서 사용**.

(Tunnel ingress rule 등록은 Phase 5에서)
