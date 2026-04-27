# Task 9.2: Runner Scale Set 정의

**Phase**: 9 (ARC (Actions Runner Controller))
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 9.2' 섹션

---


**Files:**
- Create: `platform/actions-runner-controller/runner-scale-set.yaml`

- [ ] **Step 1: Org 레벨 runner set**

`platform/actions-runner-controller/runner-scale-set.yaml`:
```yaml
# AutoscalingRunnerSet — gha-runner-scale-set chart 또는 raw CR
# 여기서는 chart로 별도 release를 만드는 대신, controller가 watching하는 CR을 직접 생성
apiVersion: actions.github.com/v1alpha1
kind: AutoscalingRunnerSet
metadata:
  name: macmini-arm64
  namespace: actions-runner-system
spec:
  githubConfigUrl: https://github.com/noah   # org 레벨
  githubConfigSecret: arc-github-app
  runnerScaleSetName: macmini-arm64
  minRunners: 0
  maxRunners: 4   # Mac mini RAM 한도
  runnerGroup: Default
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: runner
          image: ghcr.io/actions/actions-runner:2.321.0
          imagePullPolicy: IfNotPresent
          command: ["/home/runner/run.sh"]
          resources:
            requests:
              cpu: 200m
              memory: 1Gi
            limits:
              memory: 4Gi
          env:
            - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
              value: "false"   # docker-in-docker 대신 BuildKit
          volumeMounts:
            - name: work
              mountPath: /home/runner/_work
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: work
          emptyDir: {}   # ephemeral! pod 종료 시 폐기
        - name: tmp
          emptyDir: {}
```

(주: 실제로는 `gha-runner-scale-set` chart로 release를 띄우는 것이 더 표준. 위는 단순화 — chart 사용 시:)

`platform/actions-runner-controller/kustomization.yaml` 업데이트:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: actions-runner-system
resources:
  - github-app-secret.sealed.yaml
helmCharts:
  - name: gha-runner-scale-set-controller
    repo: oci://ghcr.io/actions/actions-runner-controller-charts
    version: 0.10.1
    releaseName: arc
    namespace: actions-runner-system
    valuesFile: values.yaml
  - name: gha-runner-scale-set
    repo: oci://ghcr.io/actions/actions-runner-controller-charts
    version: 0.10.1
    releaseName: macmini-arm64
    namespace: actions-runner-system
    valuesFile: scale-set-values.yaml
```

`platform/actions-runner-controller/scale-set-values.yaml`:
```yaml
githubConfigUrl: https://github.com/noah
githubConfigSecret: arc-github-app
runnerScaleSetName: macmini-arm64
minRunners: 0
maxRunners: 4
template:
  spec:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1001
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:2.321.0
        resources:
          requests: { cpu: 200m, memory: 1Gi }
          limits: { memory: 4Gi }
        env:
          - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
            value: "false"
        volumeMounts:
          - { name: work, mountPath: /home/runner/_work }
        volumeMounts:
          - { name: tmp, mountPath: /tmp }
    volumes:
      - { name: work, emptyDir: {} }
      - { name: tmp, emptyDir: {} }
```

- [ ] **Step 2: Outbound NetworkPolicy (allowlist)**

`platform/actions-runner-controller/network-policy.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: runner-egress-allowlist
  namespace: actions-runner-system
spec:
  podSelector:
    matchLabels:
      actions.github.com/scale-set-name: macmini-arm64
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: {}   # cluster 내부 (kube-dns 등)
    - ports:
        - { protocol: TCP, port: 443 }
        - { protocol: TCP, port: 80 }   # GitHub redirects
      to:
        - ipBlock:
            cidr: 0.0.0.0/0
            # GHCR, GitHub API, Sigstore, npmjs, pypi, crates 등 모든 공용 https
            # 더 빡빡하게 하려면 GitHub IP 범위 + 알려진 registry 목록
```

(실용 관점: 처음엔 0.0.0.0/0 https. 사고 발생 시 좁힘.)

`platform/actions-runner-controller/kustomization.yaml`의 resources에 추가:
```yaml
resources:
  - github-app-secret.sealed.yaml
  - network-policy.yaml
```

- [ ] **Step 3: 적용 + 첫 runner 등록 검증**

```bash
git add platform/actions-runner-controller/
git commit -m "feat(platform): actions-runner-controller with ephemeral runners"
git push
argocd app sync platform-actions-runner-controller

kubectl wait --for=condition=available --timeout=180s -n actions-runner-system deployment/arc-gha-rs-controller
# ARC가 자체 listener pod 생성, runner는 job 트리거 시 동적 생성
kubectl get pods -n actions-runner-system
```

GitHub UI → Settings → Actions → Runner groups → "macmini-arm64" 등록 확인.

- [ ] **Step 4: 검증 commit**

```bash
echo "## Phase 9 acceptance" > tests/acceptance/phase-9-arc.md
echo "- [x] arc-controller Running" >> tests/acceptance/phase-9-arc.md
echo "- [x] Runner scale set macmini-arm64 visible in GitHub Actions UI" >> tests/acceptance/phase-9-arc.md
echo "- [x] Egress NetworkPolicy applied to runner pods" >> tests/acceptance/phase-9-arc.md
echo "- [x] Runner pods are ephemeral (emptyDir _work, no persistent state)" >> tests/acceptance/phase-9-arc.md
git add tests/acceptance/phase-9-arc.md
git commit -m "test: phase 9 acceptance"
git push
```

---
