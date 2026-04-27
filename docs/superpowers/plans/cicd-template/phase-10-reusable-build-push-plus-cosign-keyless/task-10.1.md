# Task 10.1: reusable workflow 작성

**Phase**: 10 (Reusable Build-Push + Cosign Keyless)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 10.1' 섹션

---


**Files:**
- Create: `.github/workflows/reusable-build-push.yml`

- [ ] **Step 1: workflow 파일**

`.github/workflows/reusable-build-push.yml`:
```yaml
name: reusable-build-push

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
        description: "서비스 이름 (GHCR 이미지 이름)"
      dockerfile:
        required: false
        type: string
        default: Dockerfile
      context:
        required: false
        type: string
        default: "."
      platforms:
        required: false
        type: string
        default: linux/arm64
      push:
        required: false
        type: boolean
        default: true
      gitleaks:
        required: false
        type: boolean
        default: true
    outputs:
      image:
        description: "ghcr.io/noah-study/<svc>"
        value: ${{ jobs.build.outputs.image }}
      digest:
        description: "sha256:..."
        value: ${{ jobs.build.outputs.digest }}
      tag-sha:
        description: "sha-<short>"
        value: ${{ jobs.build.outputs.tag_sha }}

permissions:
  contents: read
  packages: write
  id-token: write   # Cosign keyless OIDC

jobs:
  test:
    runs-on: [self-hosted, macmini-arm64]
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
      - name: Detect language and run tests
        run: |
          if [ -f package.json ]; then
            corepack enable
            pnpm install --frozen-lockfile || npm ci
            npm test --if-present
          elif [ -f go.mod ]; then
            go test ./...
          elif [ -f pyproject.toml ]; then
            pip install -e ".[dev]" || pip install -e .
            pytest
          elif [ -f Cargo.toml ]; then
            cargo test
          else
            echo "No recognized lang config; skipping tests"
          fi

  gitleaks:
    if: ${{ inputs.gitleaks }}
    runs-on: [self-hosted, macmini-arm64]
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@cb7149b9b57195b609c63e8518d2c37a83d8c1cf   # v2.3.6
        env:
          GITHUB_TOKEN: ${{ github.token }}

  build:
    needs: [test]
    if: ${{ inputs.push }}
    runs-on: [self-hosted, macmini-arm64]
    outputs:
      image: ${{ steps.meta.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}
      tag_sha: ${{ steps.meta.outputs.tag_sha }}
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11   # v4.1.1

      - id: meta
        name: Compute tags
        run: |
          SHA="${GITHUB_SHA::7}"
          RAW="${GITHUB_REF_NAME}"
          SLUG=$(echo "$RAW" | tr '[:upper:]' '[:lower:]' | sed 's@[/_]@-@g' | sed 's/[^a-z0-9.-]//g')
          IMG="ghcr.io/noah-study/${{ inputs.service-name }}"
          {
            echo "image=$IMG"
            echo "tag_sha=sha-$SHA"
            echo "tag_branch_sha=${SLUG}-${SHA}"
            echo "tag_branch_latest=${SLUG}-latest"
          } >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@4fd812986e6c8c2a69e18311145f9371337f27d4   # v3.4.0
      - uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567   # v3.3.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ github.token }}

      - id: build
        uses: docker/build-push-action@5cd11c3a4ced054e52742c5fd54dca954e0edd85   # v6.7.0
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: true
          provenance: mode=max
          sbom: true
          cache-from: type=registry,ref=${{ steps.meta.outputs.image }}:buildcache
          cache-to: type=registry,ref=${{ steps.meta.outputs.image }}:buildcache,mode=max
          tags: |
            ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.tag_sha }}
            ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.tag_branch_sha }}
            ${{ steps.meta.outputs.image }}:${{ steps.meta.outputs.tag_branch_latest }}
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
            org.opencontainers.image.ref.name=${{ github.ref_name }}

      - uses: sigstore/cosign-installer@4959ce089c160fddf62f7b42464195ba1a56d382   # v3.6.0

      - name: Cosign keyless sign
        env:
          COSIGN_EXPERIMENTAL: "true"
        run: |
          cosign sign --yes \
            "${{ steps.meta.outputs.image }}@${{ steps.build.outputs.digest }}"
          echo "Signed: ${{ steps.meta.outputs.image }}@${{ steps.build.outputs.digest }}"
          # Rekor entry 검증
          cosign verify \
            --certificate-identity-regexp "^https://github.com/noah-study/.*/.github/workflows/reusable-build-push.yml@" \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com \
            "${{ steps.meta.outputs.image }}@${{ steps.build.outputs.digest }}"

      - name: Summary
        run: |
          {
            echo "## Image built and signed"
            echo "- **Image**: ${{ steps.meta.outputs.image }}"
            echo "- **Digest**: ${{ steps.build.outputs.digest }}"
            echo "- **Tags**: ${{ steps.meta.outputs.tag_sha }}, ${{ steps.meta.outputs.tag_branch_sha }}, ${{ steps.meta.outputs.tag_branch_latest }}"
            echo "- **Cosign**: keyless OIDC, Rekor entry verified"
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 2: workflow lint (act 또는 actionlint)**

```bash
brew install actionlint
actionlint .github/workflows/reusable-build-push.yml
```

기대: 0 errors.

- [ ] **Step 3: commit**

```bash
git add .github/workflows/reusable-build-push.yml
git commit -m "feat(ci): reusable build-push workflow with cosign keyless + verify"
git push
```

(실 동작 검증은 Phase 12 sample-app smoke test에서)

---
