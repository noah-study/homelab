# Task 11.1: apps/_base 표준 매니페스트

**Phase**: 11 (apps/_base + 새 앱 onboarding 스크립트)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 11.1' 섹션

---


**Files:**
- Create: `apps/_base/deployment.yaml`
- Create: `apps/_base/service.yaml`
- Create: `apps/_base/ingress.yaml`
- Create: `apps/_base/servicemonitor.yaml`
- Create: `apps/_base/networkpolicy.yaml`
- Create: `apps/_base/kustomization.yaml`

- [ ] **Step 1: deployment 표준 (PSA restricted 호환 + OTel env)**

`apps/_base/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app.kubernetes.io/name: PLACEHOLDER_APP_NAME
    app.kubernetes.io/component: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: PLACEHOLDER_APP_NAME
  template:
    metadata:
      labels:
        app.kubernetes.io/name: PLACEHOLDER_APP_NAME
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        fsGroup: 65532
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: app
          image: PLACEHOLDER_IMAGE
          ports:
            - { name: http, containerPort: 8080 }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: [ALL] }
          resources:
            requests: { cpu: 50m, memory: 128Mi }
            limits: { memory: 256Mi }
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 10
          readinessProbe:
            httpGet: { path: /ready, port: http }
            initialDelaySeconds: 3
          env:
            - name: POD_NAME
              valueFrom: { fieldRef: { fieldPath: metadata.name } }
            - name: POD_NAMESPACE
              valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://alloy.observability.svc.cluster.local:4317
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: grpc
            - name: OTEL_SERVICE_NAME
              value: PLACEHOLDER_APP_NAME
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "deployment.environment=$(ENV),k8s.namespace.name=$(POD_NAMESPACE),k8s.pod.name=$(POD_NAME)"
            - name: ENV
              value: PLACEHOLDER_ENV
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
      volumes:
        - { name: tmp, emptyDir: {} }
```

- [ ] **Step 2: service + ingress + servicemonitor + networkpolicy**

`apps/_base/service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
  labels:
    app.kubernetes.io/name: PLACEHOLDER_APP_NAME
spec:
  selector:
    app.kubernetes.io/name: PLACEHOLDER_APP_NAME
  ports:
    - { name: http, port: 80, targetPort: http }
```

`apps/_base/ingress.yaml`:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: PLACEHOLDER_HOST
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
  tls:
    - hosts: [PLACEHOLDER_HOST]
      secretName: noah-dev-wildcard-tls
```

`apps/_base/servicemonitor.yaml`:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app
  labels:
    release: alloy
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: PLACEHOLDER_APP_NAME
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

`apps/_base/networkpolicy.yaml`:
```yaml
# 기본 deny + ingress-nginx 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  egress:
    # DNS
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
      ports:
        - { protocol: UDP, port: 53 }
    # OTel collector (Alloy)
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: observability } }
      ports:
        - { protocol: TCP, port: 4317 }
    # 외부 https (앱이 외부 API 호출)
    - to:
        - ipBlock: { cidr: 0.0.0.0/0 }
      ports:
        - { protocol: TCP, port: 443 }
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-nginx
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - { protocol: TCP, port: 8080 }
```

- [ ] **Step 3: kustomization 묶기 + cert 복제**

`apps/_base/certificate.yaml` (각 앱 namespace에 와일드카드 cert 발급):
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: noah-dev-wildcard
spec:
  secretName: noah-dev-wildcard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames: ["noah.dev", "*.noah.dev", "*.dev.noah.dev"]
```

`apps/_base/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - certificate.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - servicemonitor.yaml
  - networkpolicy.yaml
```

- [ ] **Step 4: commit**

```bash
git add apps/_base/
git commit -m "feat(apps): _base Kustomize template with PSA + OTel + NetworkPolicy"
git push
```
