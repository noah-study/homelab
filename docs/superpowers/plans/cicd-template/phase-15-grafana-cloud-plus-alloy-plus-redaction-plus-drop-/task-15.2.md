# Task 15.2: Alloy 설치 + redaction + drop rules

**Phase**: 15 (Grafana Cloud + Alloy + Redaction + Drop Rules)
**원본 plan**: `../../../2026-04-26-infra-cicd-template-plan.md` 의 'Task 15.2' 섹션

---


**Files:**
- Create: `platform/grafana-alloy/kustomization.yaml`
- Create: `platform/grafana-alloy/values.yaml`
- Create: `platform/grafana-alloy/grafana-cloud-token.sealed.yaml`
- Create: `platform/grafana-alloy/config.alloy`

- [ ] **Step 1: Grafana Cloud 토큰 봉인**

```bash
GRAFANA_TOKEN=$(op read "op://infra-keys/grafana-cloud-alloy-token/credential")
./scripts/seal.sh observability grafana-cloud-token \
  token="$GRAFANA_TOKEN" \
  > platform/grafana-alloy/grafana-cloud-token.sealed.yaml
```

- [ ] **Step 2: Alloy config (drop rules + redaction)**

`platform/grafana-alloy/config.alloy`:
```hcl
// Prometheus scrape (k8s discovery)
discovery.kubernetes "pods" {
  role = "pod"
}

prometheus.scrape "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [prometheus.relabel.drop_high_cardinality.receiver]
}

// Drop high-cardinality / unused metrics
prometheus.relabel "drop_high_cardinality" {
  forward_to = [prometheus.remote_write.grafana.receiver]

  rule {
    source_labels = ["__name__"]
    regex         = "container_blkio_.*|container_tasks_.*|container_fs_inodes_.*|container_memory_failures_.*"
    action        = "drop"
  }
  rule {
    regex  = "id|container_id|pod_uid"
    action = "labeldrop"
  }
}

prometheus.remote_write "grafana" {
  endpoint {
    url = "https://prometheus-prod-XX-XX.grafana.net/api/prom/push"   // 실제 URL로 교체
    basic_auth {
      username = "INSTANCE_ID"   // 실제 ID로 교체
      password = remote.kubernetes.secret.grafana_token.data["token"]
    }
  }
}

// OTel receiver
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output {
    logs    = [otelcol.processor.attributes.redact_logs.input]
    metrics = [otelcol.processor.attributes.redact_metrics.input]
    traces  = [otelcol.processor.attributes.redact_traces.input]
  }
}

// Redaction processor (보안 F10)
otelcol.processor.attributes "redact_logs" {
  action {
    key        = "http.request.header.authorization"
    action     = "delete"
  }
  action {
    key        = "http.request.header.cookie"
    action     = "delete"
  }
  action {
    pattern    = "(?i)(password|api[_-]?key|token|secret)"
    action     = "delete"
  }
  output { logs = [otelcol.exporter.loki.default.input] }
}

otelcol.processor.attributes "redact_metrics" {
  output { metrics = [otelcol.exporter.prometheus.default.input] }
}

otelcol.processor.attributes "redact_traces" {
  action {
    key     = "http.request.header.authorization"
    action  = "delete"
  }
  output { traces = [otelcol.exporter.otlphttp.tempo.input] }
}

// Exporters (실제 URL로 교체)
otelcol.exporter.loki "default" {
  forward_to = [loki.write.grafana.receiver]
}

loki.write "grafana" {
  endpoint {
    url = "https://logs-prod-XXX.grafana.net/loki/api/v1/push"
    basic_auth {
      username = "INSTANCE_ID"
      password = remote.kubernetes.secret.grafana_token.data["token"]
    }
  }
}

otelcol.exporter.otlphttp "tempo" {
  client {
    endpoint = "https://tempo-prod-XX-prod-XX-XX.grafana.net"
    auth     = otelcol.auth.basic.tempo.handler
  }
}

otelcol.auth.basic "tempo" {
  username = "INSTANCE_ID"
  password = remote.kubernetes.secret.grafana_token.data["token"]
}

remote.kubernetes.secret "grafana_token" {
  namespace = "observability"
  name      = "grafana-cloud-token"
}
```

(주: 실제 URL/instance ID는 Grafana Cloud Portal에서 복사. 위는 템플릿 — `XX`를 본인 값으로 치환.)

- [ ] **Step 3: Helm wrapper (Alloy)**

`platform/grafana-alloy/values.yaml`:
```yaml
alloy:
  configMap:
    create: false   # 별도 ConfigMap 사용
    name: alloy-config
  extraPorts:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
controller:
  type: daemonset
serviceMonitor:
  enabled: false
```

`platform/grafana-alloy/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: observability
resources:
  - grafana-cloud-token.sealed.yaml
configMapGenerator:
  - name: alloy-config
    files:
      - config.alloy
helmCharts:
  - name: alloy
    repo: https://grafana.github.io/helm-charts
    version: 0.10.0
    releaseName: alloy
    namespace: observability
    valuesFile: values.yaml
```

- [ ] **Step 4: 적용 + 검증**

```bash
git add platform/grafana-alloy/
git commit -m "feat(observability): Grafana Alloy with redaction + drop rules"
git push
argocd app sync platform-grafana-alloy

kubectl wait --for=condition=ready --timeout=180s -n observability pod -l app.kubernetes.io/name=alloy
kubectl logs -n observability daemonset/alloy --tail=30
```

기대: 로그에 "remote write success", drop rule 적용 확인.

Grafana Cloud → Explore → Prometheus: `count by (__name__)({namespace=~".+"})` → 차원이 줄었음 확인. Series 카운트 < 8000.

- [ ] **Step 5: 검증 commit**

```bash
echo "## Phase 15 acceptance" > tests/acceptance/phase-15-observability.md
echo "- [x] Alloy DaemonSet running, scraping cluster + receiving OTel" >> tests/acceptance/phase-15-observability.md
echo "- [x] Grafana Cloud Explore shows metrics from sample-app" >> tests/acceptance/phase-15-observability.md
echo "- [x] Active series < 8K (free tier 80% SLO)" >> tests/acceptance/phase-15-observability.md
echo "- [x] Logs from sample-app pods visible (Loki)" >> tests/acceptance/phase-15-observability.md
echo "- [x] Authorization header redacted in trace attributes" >> tests/acceptance/phase-15-observability.md
git add tests/acceptance/phase-15-observability.md
git commit -m "test: phase 15 observability acceptance"
git push
```

---
