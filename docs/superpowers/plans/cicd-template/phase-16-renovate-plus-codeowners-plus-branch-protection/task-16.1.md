# Task 16.1: Renovate 룰

**Phase**: 16 (Renovate + CODEOWNERS + Branch Protection)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 16.1' 섹션

---


**Files:**
- Create: `renovate.json`

- [ ] **Step 1: 안전 룰 박기**

`renovate.json`:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    "helpers:pinGitHubActionDigests",
    ":timezone(Asia/Seoul)",
    ":dependencyDashboard"
  ],
  "schedule": ["before 9am on monday"],
  "prHourlyLimit": 4,
  "prConcurrentLimit": 10,
  "labels": ["dependencies"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "schedule": ["at any time"]
  },
  "packageRules": [
    {
      "matchManagers": ["github-actions"],
      "matchUpdateTypes": ["minor", "patch", "digest"],
      "matchPackagePatterns": ["^actions/", "^docker/", "^sigstore/", "^aquasecurity/"],
      "automerge": true,
      "automergeType": "branch",
      "platformAutomerge": true
    },
    {
      "matchManagers": ["helm-values", "helmv3"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    },
    {
      "matchPackageNames": [
        "ingress-nginx",
        "cert-manager",
        "argo-cd",
        "argocd-image-updater",
        "cilium",
        "actions-runner-controller"
      ],
      "automerge": false,
      "labels": ["platform-critical"]
    }
  ]
}
```

- [ ] **Step 2: Renovate GitHub App 설치**

https://github.com/apps/renovate → Install → noah org → Select repos: infra + sample-app + (앞으로 모든 service repos).

서비스 레포에는 minimal `renovate.json` 두기 — `extends: ["github>noah-study/homelab//renovate.json"]`로 본 룰 상속.
