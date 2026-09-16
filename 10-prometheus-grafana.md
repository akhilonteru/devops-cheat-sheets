# Prometheus & Grafana Cheat Sheet

> Monitoring stack reference — Prometheus configuration, PromQL, alerting rules, Alertmanager, and Grafana provisioning.

---

## Table of Contents

- [Running Prometheus](#1-running-prometheus)
- [Configuration (prometheus.yml)](#2-configuration-prometheusyml)
- [Service Discovery](#3-service-discovery)
- [PromQL Basics](#4-promql-basics)
- [Useful PromQL Queries](#5-useful-promql-queries)
- [Recording & Alerting Rules](#6-recording--alerting-rules)
- [Alertmanager](#7-alertmanager)
- [Common Exporters](#8-common-exporters)
- [Grafana](#9-grafana)
- [Grafana Provisioning](#10-grafana-provisioning)

---

## 1. Running Prometheus

```bash
# Docker (quick start)
docker run -d --name prometheus \
  -p 9090:9090 \
  -v ./prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:latest

# Docker Compose
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'

volumes:
  prometheus-data:
```

**UI endpoints:** `http://localhost:9090/graph` (query UI), `/targets` (scrape status), `/alerts` (active alerts), `/config` (running config), `/rules`.

## 2. Configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 15s            # Default scrape interval
  scrape_timeout: 10s
  evaluation_interval: 15s        # Rule evaluation interval
  external_labels:
    cluster: prod-us-1
    environment: production

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets:
          - node1:9100
          - node2:9100
    metrics_path: /metrics
    scheme: http
    scrape_interval: 30s
    basic_auth:
      username: monitor
      password_file: /etc/prometheus/secrets/node-password

  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets: ['https://example.com', 'https://api.example.com']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

## 3. Service Discovery

```yaml
scrape_configs:
  # Kubernetes
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [default, staging]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_ip, __meta_kubernetes_pod_annotation_prometheus_io_port]
        separator: ':'
        target_label: __address__

  # AWS EC2
  - job_name: ec2
    ec2_sd_configs:
      - region: us-east-1
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Role]
        target_label: role
      - source_labels: [__meta_ec2_private_ip]
        target_label: private_ip

  # Consul
  - job_name: consul
    consul_sd_configs:
      - server: consul:8500
        services: [api, web]
```

**Common relabel actions:** `keep`, `drop`, `replace`, `labelmap`, `labeldrop`, `labelkeep`.

## 4. PromQL Basics

```
# Instant vector — value at a point in time
up
node_cpu_seconds_total{mode="idle"}

# Range vector — values over a time window [required by rate()/increase()]
node_cpu_seconds_total{mode="idle"}[5m]

# Rate — per-second average rate of counter increase
rate(http_requests_total[5m])

# Filters
http_requests_total{job="api", instance="api-1:8080"}
{__name__=~"node_.*", job="node"}

# Regex matching
up{job=~"node|prometheus"}
http_requests_total{status=~"5.."}
```

**Data types:** instant vector, range vector, scalar, string. **Aggregation:** `sum`, `avg`, `min`, `max`, `count`, `topk`, `bottomk`, `quantile` with `by` / `without`.

## 5. Useful PromQL Queries

```promql
# Up/availability
up                                                        # Target reachability
count by (job) (up == 1)                                  # Healthy targets per job

# Request rates & errors
sum by (job) (rate(http_requests_total[5m]))                          # req/sec
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))           # errors/sec
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))                                # error ratio

# Latency percentiles (histograms)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# CPU / memory (node_exporter)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)   # CPU %
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)           # Mem %
node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes # Disk free %

# Top consumers
topk(5, sum by (pod) (container_memory_working_set_bytes))
topk(5, rate(container_cpu_usage_seconds_total[5m]))

# Saturation & prediction
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 24*3600)   # Bytes left in 24h
sum(kube_pod_status_ready{condition="false"})                               # Not-ready pods

# Comparisons & offsets
node_cpu_seconds_total > 1000
sum(rate(http_requests_total[5m])) offset 1d               # Compare with yesterday
absent(up{job="api"} == 1)                                # Alert when target missing

# Join between metrics
node_memory_MemAvailable_bytes / on(instance) group_left job:node_memory:total   # per-job available
```

## 6. Recording & Alerting Rules

`alerts/api.yml`:
```yaml
groups:
  - name: api.rules
    interval: 30s
    rules:
      # Recording rule — pre-compute an expensive query
      - record: job:api_request_rate:sum
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:api_latency:p99
        expr: histogram_quantile(0.99, sum by (job, le) (rate(http_request_duration_seconds_bucket[5m])))

  - name: api.alerts
    rules:
      - alert: ApiHighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "{{ $labels.job }} has {{ $value | humanizePercentage }} errors over the last 5 minutes."
          runbook: "https://wiki.example.com/runbooks/api-error-rate"

      - alert: DiskWillFillIn72Hours
        expr: predict_linear(node_filesystem_avail_bytes{fstype="ext4"}[1h], 72 * 3600) < 0
        for: 1h
        labels:
          severity: warning
```

```bash
# Validate rules
promtool check rules alerts/*.yml
promtool check config prometheus.yml
promtool query instant http://localhost:9090 'up'        # Test query via CLI
```

## 7. Alertmanager

`alertmanager.yml`:
```yaml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@example.com'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match: { severity: critical }
      receiver: pagerduty
      continue: true
    - match: { team: backend }
      receiver: slack-backend

receivers:
  - name: default
    email_configs:
      - to: 'team@example.com'

  - name: slack-backend
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#alerts-backend'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: pagerduty
    pagerduty_configs:
      - service_key: '{{ .PagerDutyKey }}'

inhibit_rules:
  - source_match: { severity: critical }
    target_match: { severity: warning }
    equal: ['alertname', 'instance']       # Mute warnings when critical fires
```

```bash
amtool config check alertmanager.yml
amtool alert add alertname=Test severity=critical        # Test alert
```

## 8. Common Exporters

| Exporter | Port | Purpose |
| --- | --- | --- |
| `node_exporter` | 9100 | Linux host metrics (CPU/mem/disk/net) |
| `blackbox_exporter` | 9115 | HTTP/TCP/ICMP/DNS probing |
| `cadvisor` | 8080 | Container metrics |
| `kube-state-metrics` | 8080 | Kubernetes object state |
| `postgres_exporter` | 9187 | PostgreSQL |
| `redis_exporter` | 9121 | Redis |
| `nginx-prometheus-exporter` | 9113 | Nginx stub_status |
| `mongodb_exporter` | 9216 | MongoDB |
| `jmx_exporter` | varies | JVM metrics |

```bash
# Blackbox prober config (blackbox.yml)
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 301, 302]
      method: GET
```

## 9. Grafana

```bash
# Run Grafana
docker run -d --name grafana \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  -e GF_USERS_ALLOW_SIGN_UP=false \
  -v grafana-data:/var/lib/grafana \
  -v ./provisioning:/etc/grafana/provisioning \
  -v ./dashboards:/etc/grafana/dashboards \
  grafana/grafana:latest

# CLI
grafana-cli plugins install grafana-clock-panel
grafana-cli plugins ls
grafana-cli admin reset-admin-password newpassword     # Reset forgotten admin password
```

**Default login:** `admin` / `admin` (forced change). **Explore tab** for ad-hoc queries. **Dashboard variables:**
```promql
label_values(node_cpu_seconds_total{job="node"}, instance)          # Query variable
up{instance=~"$instance"}                                            # Use variable
```

## 10. Grafana Provisioning

`provisioning/datasources/prometheus.yml`:
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      timeInterval: 15s
```

`provisioning/dashboards/dashboards.yml`:
```yaml
apiVersion: 1
providers:
  - name: default
    folder: ''
    type: file
    options:
      path: /etc/grafana/dashboards
```

Export dashboard JSON → save into `/etc/grafana/dashboards/` → auto-loaded, versioned in Git.

---
