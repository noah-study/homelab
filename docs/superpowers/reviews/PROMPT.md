# Ralph Loop Review Task

You are reviewing the infra CI/CD template design at `/Users/noah/Development/homelab/docs/superpowers/specs/2026-04-26-infra-cicd-template-design.md`.

Produce three separate critical review documents in `/Users/noah/Development/homelab/docs/superpowers/reviews/`, one per perspective:

## Review 1 — `review-security.md` (adversarial security)

Threat model the design. Find concrete attack paths. Cover:

- GitHub Actions self-hosted runner risks (runner repo poisoning, fork PR exploitation, secret exfiltration via build steps, runner persistence between jobs on macOS)
- GHCR/Cosign supply chain gaps — signature verification is deferred 1-2 months; what's the actual exposure window? Can a compromised actions ref push unsigned images that ArgoCD blindly deploys?
- Sealed Secrets master key handling — single point of total compromise. Recovery workflow risk. 1Password as backup target trust model.
- Cloudflare Tunnel trust boundary — what if cloudflared token leaks?
- ingress-nginx CVE history — recent critical CVEs (CVE-2025-1974 IngressNightmare etc.)
- NetworkPolicy default-deny correctness with Cilium — common misconfiguration patterns
- PodSecurityStandards `restricted` enforcement gaps — what does `_base` need to comply?
- RBAC for ArgoCD/Image Updater write-back to git — token scope, blast radius if compromised
- Renovate auto-merge attack surface — typosquatting, malicious patch versions, lockfile poisoning
- Secret leakage via OTel/logs — env vars accidentally captured, request bodies in traces
- Mac mini macOS host attack surface — Linux VM escape, OrbStack vulnerabilities, host SSH if exposed
- kubeseal master key 1Password backup workflow risks — when to backup, who backs up, rotation handling
- Trivy-operator findings handling — who acts on alerts? noise vs real signal
- DNS/DNSSEC for noah.dev — domain hijacking → all TLS broken
- Cosign keyless OIDC trust — what GitHub repo identities are allowed?

For EACH finding give: severity (Critical/High/Medium/Low), concrete attack scenario, specific mitigation, Day-1 vs deferred.

## Review 2 — `review-scalability.md` (scale & performance)

Stress-test the design at 5 / 20 / 50 / 100 services. Cover:

- Mac mini resource ceiling — concrete numbers for M2 Pro 16GB / 32GB. Idle vs loaded. Linux VM overhead.
- k3s single-node limits — sqlite vs etcd write throughput, max pods per node (110 default), ingress capacity
- ArgoCD reconciliation performance — Application count limits, default 3-min polling vs webhook, controller memory at N apps
- Image Updater polling cost at scale — N services × M tags × poll interval = GHCR API calls/min
- ApplicationSet generator performance with many directories — git generator latency
- GHCR storage growth — 3 tags per build × builds per day × N services
- Self-hosted runner queue saturation under PR storms — single runner = serial. Concurrency settings.
- Grafana Cloud Free tier ceiling break-points — concrete formulas
- Renovate PR volume explosion — when N service repos each get weekly bumps
- dev environment contention — feat/* branches stacking
- When does the architecture genuinely break? What's the migration path?
- Adding a 2nd node — k3s server vs agent, etcd promotion, network considerations
- When k3s becomes inadequate (move to vanilla k8s or managed)

Use concrete numbers, not handwaving.

## Review 3 — `review-free-economics.md` (free-tier economics)

Project actual monthly costs at 5 / 20 / 50 services. Cover:

- Grafana Cloud Free real-world series count — kube-state-metrics base load, node-exporter base, cAdvisor per-pod, N apps × default histograms. Where exactly does it break?
- GHCR storage and bandwidth — private repo (500MB storage / 1GB transfer free), when do we exceed?
- Cloudflare Free tier — Tunnel limits, DNS limits, Universal SSL, rate limiting
- GitHub Actions self-hosted minutes (free) — but bandwidth/disk costs, GHCR API rate limits
- Sigstore Rekor — free but rate limits?
- Mac mini electricity cost — kWh/month by load profile, regional pricing
- Hidden costs — domain renewal, future SaaS triggers
- Optimization headroom — what does relabel buy in series? log sampling? image layer caching?
- Honest break-even — at what scale does staying free cost more in operator time than paying for Pro tier (~USD 19-50/month)
- Specific paid upgrades to consider in order: which one first when?

Use real numbers and citations where possible.

---

## Review Document Structure

Each file uses this structure:
```
# Review: <Perspective>

## TL;DR (3-5 lines)

## Findings (numbered, severity-tagged)
### F1. [Severity] Title
- **Scenario**: ...
- **Mitigation**: ...
- **Day-1 vs deferred**: ...

(repeat)

## Cross-cutting concerns

## Recommendation (actionable items tied to specific sections of the design doc)
```

## Iteration Strategy

Each loop:
1. Read all three review docs (if they exist)
2. Identify the WEAKEST one — shortest, shallowest, fewest concrete numbers, least Critical/High findings
3. Deepen it: add findings, replace handwaving with numbers, link to design doc sections
4. Verify all claims against the design doc

## Completion Criteria

Output `<promise>REVIEWS COMPLETE</promise>` ONLY when ALL THREE reviews satisfy:
- (a) at least 8 distinct findings each
- (b) at least 3 Critical/High findings each
- (c) every finding has concrete scenario or numbers (not generic principles)
- (d) Recommendation section has actionable items tied to specific design doc sections (e.g., "Section 2.4: add explicit X")
- (e) Cross-cutting concerns section is non-empty

Where the design doc is ambiguous on a point relevant to a review, note it as `Design ambiguity: ...` rather than guessing.
