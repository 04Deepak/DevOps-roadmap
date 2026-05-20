# 10. Monitoring & Logging

## Table of Contents
- [Observability Concepts](#observability-concepts)
- [Prometheus Setup](#prometheus-setup)
- [PromQL - Querying Metrics](#promql---querying-metrics)
- [Grafana Setup](#grafana-setup)
- [Alertmanager](#alertmanager)
- [ELK Stack (Elasticsearch, Logstash, Kibana)](#elk-stack-elasticsearch-logstash-kibana)
- [EFK Stack (Elasticsearch, Fluentd, Kibana)](#efk-stack-elasticsearch-fluentd-kibana)
- [Loki - Lightweight Log Aggregation](#loki---lightweight-log-aggregation)
- [Full Monitoring Stack with Docker Compose](#full-monitoring-stack-with-docker-compose)
- [Projects](#projects)

---

## Observability Concepts

Observability is the ability to understand the internal state of a system by examining its external outputs. It rests on three pillars:

```
┌──────────────────────────────────────────────────────────────┐
│                   Three Pillars of Observability              │
│                                                               │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│   │   METRICS    │    │    LOGS      │    │   TRACES     │     │
│   │              │    │              │    │              │     │
│   │  Numerical   │    │  Timestamped │    │  Request     │     │
│   │  measurements│    │  text events │    │  journey     │     │
│   │  over time   │    │  from apps   │    │  across      │     │
│   │              │    │              │    │  services    │     │
│   │  Example:    │    │  Example:    │    │  Example:    │     │
│   │  CPU at 72%  │    │  "User 42    │    │  HTTP req    │     │
│   │  RAM 4.2 GB  │    │   logged in" │    │  -> API      │     │
│   │  Req/sec: 50 │    │  "DB timeout │    │  -> DB       │     │
│   │              │    │   after 30s" │    │  -> Cache    │     │
│   └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                               │
│   Tools:             Tools:             Tools:               │
│   Prometheus         ELK Stack          Jaeger               │
│   Grafana            Loki               Zipkin               │
│   Datadog            Fluentd            OpenTelemetry        │
└──────────────────────────────────────────────────────────────┘
```

### Metrics

Metrics are numeric values measured over time. They are lightweight and efficient to store.

**Metric Types:**

| Type | Description | Example |
|------|-------------|---------|
| Counter | Only goes up (resets on restart) | Total HTTP requests served |
| Gauge | Goes up and down | Current CPU usage, active connections |
| Histogram | Samples observations into buckets | Request duration distribution |
| Summary | Similar to histogram, calculates quantiles | Request latency at p50, p95, p99 |

### Logs

Logs are timestamped text records of discrete events. They provide the richest context but are expensive to store and query at scale.

**Log Levels:**

| Level | When to Use |
|-------|-------------|
| DEBUG | Detailed diagnostic information |
| INFO | Routine operational messages |
| WARN | Something unexpected but not a failure |
| ERROR | A failure that needs attention |
| FATAL | Unrecoverable error, system must stop |

### Traces

Traces follow a single request as it moves through multiple services. Each step is called a **span**. Together, spans form a **trace**.

```
Trace ID: abc-123
│
├── Span 1: API Gateway (12ms)
│   ├── Span 2: Auth Service (3ms)
│   └── Span 3: Order Service (8ms)
│       ├── Span 4: Database Query (4ms)
│       └── Span 5: Cache Lookup (1ms)
```

---

## Prometheus Setup

Prometheus is an open-source metrics monitoring system. It uses a **pull model** -- Prometheus scrapes (pulls) metrics from targets at configured intervals.

```
┌─────────────────────────────────────────────────────────┐
│                  Prometheus Architecture                  │
│                                                          │
│  ┌──────────┐  scrape   ┌─────────────┐                │
│  │  Node     │◄─────────│             │                │
│  │  Exporter │           │             │   ┌──────────┐ │
│  └──────────┘           │  Prometheus  │──►│ Grafana  │ │
│  ┌──────────┐  scrape   │  Server      │   └──────────┘ │
│  │  App      │◄─────────│             │                │
│  │  /metrics │           │             │   ┌──────────┐ │
│  └──────────┘           │             │──►│ Alert-   │ │
│  ┌──────────┐  scrape   │             │   │ manager  │ │
│  │  cAdvisor │◄─────────│             │   └──────────┘ │
│  └──────────┘           └─────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### Install Prometheus with Docker

```bash
# Create a directory for Prometheus configuration
mkdir -p ~/monitoring/prometheus
cd ~/monitoring/prometheus
```

### prometheus.yml - Main Configuration

```yaml
# ~/monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s          # How often to scrape targets
  evaluation_interval: 15s      # How often to evaluate rules
  scrape_timeout: 10s           # Timeout for each scrape

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"

# Load alert rules
rule_files:
  - "alert_rules.yml"

# Scrape configurations - what to monitor
scrape_configs:
  # Monitor Prometheus itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Monitor the host machine via Node Exporter
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
    # Optional: add labels to all metrics from this job
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: "my-server"

  # Monitor Docker containers via cAdvisor
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  # Monitor a custom application
  - job_name: "my-app"
    metrics_path: "/metrics"       # Default path
    scrape_interval: 10s           # Override global interval
    static_configs:
      - targets: ["app:8000"]
        labels:
          environment: "development"
          team: "backend"

  # Service discovery example - file-based
  - job_name: "file-sd-targets"
    file_sd_configs:
      - files:
          - "targets/*.json"
        refresh_interval: 30s
```

### Run Prometheus with Docker

```bash
# Run Prometheus container
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v ~/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v ~/monitoring/prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.retention.time=15d \
  --web.enable-lifecycle

# Verify Prometheus is running
curl http://localhost:9090/-/healthy
# Output: Prometheus Server is Healthy.

# Check configured targets
curl http://localhost:9090/api/v1/targets | python3 -m json.tool
```

### Node Exporter - Monitor Host System

Node Exporter exposes hardware and OS-level metrics (CPU, memory, disk, network).

```bash
# Run Node Exporter
docker run -d \
  --name node-exporter \
  -p 9100:9100 \
  --pid="host" \
  --net="host" \
  -v "/:/host:ro" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host

# Verify metrics are exposed
curl http://localhost:9100/metrics | head -20
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
# node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
# node_cpu_seconds_total{cpu="0",mode="system"} 4567.89
```

### cAdvisor - Monitor Docker Containers

```bash
# Run cAdvisor
docker run -d \
  --name cadvisor \
  -p 8080:8080 \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor:latest

# View container metrics
curl http://localhost:8080/metrics | grep container_cpu
```

---

## PromQL - Querying Metrics

PromQL (Prometheus Query Language) is used to select and aggregate time-series data.

### Basic Selectors

```promql
# Select all time series for a metric
node_cpu_seconds_total

# Filter by label (exact match)
node_cpu_seconds_total{mode="idle"}

# Filter by label (regex match)
node_cpu_seconds_total{mode=~"idle|iowait"}

# Exclude a label value
node_cpu_seconds_total{mode!="idle"}

# Multiple label filters
node_cpu_seconds_total{cpu="0", mode="idle"}
```

### Range Vectors and Functions

```promql
# Get values over the last 5 minutes (range vector)
node_cpu_seconds_total{mode="idle"}[5m]

# Rate of increase per second over the last 5 minutes
rate(node_cpu_seconds_total{mode="idle"}[5m])

# Increase in total over the last 1 hour
increase(node_network_receive_bytes_total[1h])

# Instant rate of change (uses last two data points)
irate(node_cpu_seconds_total{mode="idle"}[5m])
```

### Aggregation Operators

```promql
# Sum across all CPUs to get total idle time rate
sum(rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Average memory usage across all instances
avg(node_memory_MemAvailable_bytes)

# Maximum disk usage
max(node_filesystem_avail_bytes)

# Count the number of targets up
count(up == 1)

# Group by a label
sum by (mode)(rate(node_cpu_seconds_total[5m]))

# Exclude a label from grouping
sum without (cpu)(rate(node_cpu_seconds_total[5m]))
```

### Practical PromQL Examples

```promql
# ---- CPU ----

# CPU usage percentage (all cores combined)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Per-CPU usage
100 - (rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100)

# ---- Memory ----

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Memory used in GB
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024^3

# ---- Disk ----

# Disk usage percentage per mount point
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Predict disk full in 4 hours (linear regression)
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0

# ---- Network ----

# Network receive rate in Mbps
rate(node_network_receive_bytes_total{device="eth0"}[5m]) * 8 / 1024 / 1024

# ---- HTTP (application metrics) ----

# Request rate per second
rate(http_requests_total[5m])

# Error rate (5xx responses)
rate(http_requests_total{status=~"5.."}[5m])

# Error percentage
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m]) * 100

# 95th percentile request duration
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Average request duration
rate(http_request_duration_seconds_sum[5m])
  / rate(http_request_duration_seconds_count[5m])
```

---

## Grafana Setup

Grafana is an open-source visualization platform that connects to data sources like Prometheus, Loki, and Elasticsearch.

### Install Grafana with Docker

```bash
# Run Grafana
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  -e "GF_SECURITY_ADMIN_USER=admin" \
  -e "GF_SECURITY_ADMIN_PASSWORD=admin123" \
  grafana/grafana:latest

# Access Grafana at http://localhost:3000
# Default login: admin / admin123
```

### Connect Prometheus as a Data Source

After logging in to Grafana:

1. Go to **Connections > Data Sources > Add data source**
2. Select **Prometheus**
3. Set the URL to `http://prometheus:9090` (if using Docker network) or `http://localhost:9090`
4. Click **Save & Test** -- you should see "Data source is working"

You can also provision the data source via configuration file:

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
```

### Grafana Dashboard JSON

You can import dashboards via JSON. Here is an example dashboard with CPU, memory, and disk panels:

```json
{
  "dashboard": {
    "id": null,
    "uid": "node-exporter-overview",
    "title": "Node Exporter Overview",
    "tags": ["monitoring", "node-exporter"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage %",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 60 },
                { "color": "red", "value": 85 }
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Memory Usage %",
        "type": "gauge",
        "gridPos": { "h": 8, "w": 6, "x": 12, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100",
            "legendFormat": "Memory"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 70 },
                { "color": "red", "value": 90 }
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "Disk Usage %",
        "type": "bargauge",
        "gridPos": { "h": 8, "w": 6, "x": 18, "y": 0 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "(1 - node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"}) * 100",
            "legendFormat": "Disk /"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 70 },
                { "color": "red", "value": 90 }
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "Network Traffic",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "rate(node_network_receive_bytes_total{device!=\"lo\"}[5m]) * 8",
            "legendFormat": "Receive {{ device }}"
          },
          {
            "expr": "rate(node_network_transmit_bytes_total{device!=\"lo\"}[5m]) * 8",
            "legendFormat": "Transmit {{ device }}"
          }
        ],
        "fieldConfig": {
          "defaults": { "unit": "bps" }
        }
      },
      {
        "id": 5,
        "title": "System Uptime",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 12, "y": 8 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "(time() - node_boot_time_seconds) / 86400",
            "legendFormat": "Uptime"
          }
        ],
        "fieldConfig": {
          "defaults": { "unit": "d" }
        }
      },
      {
        "id": 6,
        "title": "Open File Descriptors",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 18, "y": 8 },
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "targets": [
          {
            "expr": "node_filefd_allocated",
            "legendFormat": "Open FDs"
          }
        ]
      }
    ]
  }
}
```

To import: go to **Dashboards > Import**, paste the JSON, and click **Load**.

You can also use pre-built community dashboards from [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards). Popular IDs:

| Dashboard | ID |
|-----------|-----|
| Node Exporter Full | 1860 |
| Docker and System Monitoring | 893 |
| Kubernetes Cluster Monitoring | 315 |
| Nginx Monitoring | 12708 |

Import by ID: **Dashboards > Import > Enter ID > Load**.

---

## Alertmanager

Alertmanager handles alerts sent by Prometheus. It manages grouping, silencing, inhibition, and routing alerts to receivers (email, Slack, PagerDuty, etc.).

### Alert Rules for Prometheus

```yaml
# ~/monitoring/prometheus/alert_rules.yml
groups:
  - name: system_alerts
    rules:
      # High CPU usage
      - alert: HighCpuUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for more than 5 minutes. Current value: {{ $value }}%"

      # High memory usage
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85%. Current value: {{ $value }}%"

      # Disk almost full
      - alert: DiskSpaceLow
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Disk space is running low"
          description: "Disk usage on / is above 85%. Current value: {{ $value }}%"

      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} has been down for more than 1 minute."

  - name: application_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate"
          description: "More than 5% of requests are returning 5xx errors. Current rate: {{ $value }}%"

      # Slow response times
      - alert: SlowResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow response times detected"
          description: "95th percentile response time is above 2 seconds. Current value: {{ $value }}s"
```

### Alertmanager Configuration

```yaml
# ~/monitoring/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alerts@example.com"
  smtp_auth_username: "alerts@example.com"
  smtp_auth_password: "app-password-here"

# Notification templates
templates:
  - "/etc/alertmanager/templates/*.tmpl"

# Routing tree - decides where alerts go
route:
  # Default receiver
  receiver: "slack-general"
  # Group alerts with the same labels together
  group_by: ["alertname", "severity"]
  # Wait this long to buffer alerts of the same group
  group_wait: 30s
  # Wait this long before sending a notification about new alerts
  # added to a group that already had an initial notification
  group_interval: 5m
  # Wait this long before resending an alert
  repeat_interval: 4h

  # Child routes - more specific routing
  routes:
    - match:
        severity: critical
      receiver: "slack-critical"
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: "slack-general"

# Receivers define how to send notifications
receivers:
  - name: "slack-general"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        channel: "#monitoring"
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}*{{ .Annotations.description }}*\n{{ end }}'
        send_resolved: true

  - name: "slack-critical"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
        channel: "#alerts-critical"
        title: 'CRITICAL: {{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}'
        send_resolved: true
    email_configs:
      - to: "oncall@example.com"
        send_resolved: true

# Inhibition rules - suppress alerts when a higher-severity alert is active
inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["alertname", "instance"]
```

### Run Alertmanager

```bash
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v ~/monitoring/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager:latest

# Access web UI at http://localhost:9093
# Check current alerts
curl http://localhost:9093/api/v2/alerts | python3 -m json.tool
```

---

## ELK Stack (Elasticsearch, Logstash, Kibana)

The ELK Stack collects, processes, stores, and visualizes logs.

```
┌────────────────────────────────────────────────────────────┐
│                     ELK Stack Flow                          │
│                                                             │
│  ┌──────────┐    ┌───────────┐    ┌──────────────┐        │
│  │   App     │───►│ Logstash   │───►│ Elasticsearch │        │
│  │   Logs    │    │ (Process)  │    │ (Store/Index) │        │
│  └──────────┘    └───────────┘    └──────┬───────┘        │
│  ┌──────────┐          ▲                  │               │
│  │  Syslog   │──────────┘                  ▼               │
│  └──────────┘                      ┌──────────────┐        │
│  ┌──────────┐                      │   Kibana      │        │
│  │  Files    │──────────────────►  │  (Visualize)  │        │
│  └──────────┘    (Filebeat)        └──────────────┘        │
└────────────────────────────────────────────────────────────┘
```

### ELK Stack docker-compose.yml

```yaml
# ~/monitoring/elk/docker-compose.yml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - "5044:5044"    # Beats input
      - "5000:5000"    # TCP input
      - "9600:9600"    # Monitoring API
    environment:
      - "LS_JAVA_OPTS=-Xms256m -Xmx256m"
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - elk

volumes:
  es-data:

networks:
  elk:
    driver: bridge
```

### Logstash Configuration

```yaml
# ~/monitoring/elk/logstash/config/logstash.yml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

### Logstash Pipeline - Processing Rules

```ruby
# ~/monitoring/elk/logstash/pipeline/logstash.conf

# INPUT: Where logs come from
input {
  # Accept logs over TCP
  tcp {
    port => 5000
    codec => json_lines
  }

  # Accept logs from Filebeat
  beats {
    port => 5044
  }
}

# FILTER: Parse and transform logs
filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Parse Apache/Nginx access logs
  if [type] == "nginx-access" {
    grok {
      match => {
        "message" => '%{IPORHOST:remote_addr} - %{DATA:remote_user} \[%{HTTPDATE:time_local}\] "%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:body_bytes_sent}'
      }
    }
    date {
      match => ["time_local", "dd/MMM/yyyy:HH:mm:ss Z"]
    }
    mutate {
      convert => {
        "status" => "integer"
        "body_bytes_sent" => "integer"
      }
    }
  }

  # Add geographic location from IP
  if [remote_addr] {
    geoip {
      source => "remote_addr"
    }
  }

  # Remove unnecessary fields
  mutate {
    remove_field => ["@version", "host"]
  }
}

# OUTPUT: Where processed logs go
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }

  # Also print to stdout for debugging
  stdout {
    codec => rubydebug
  }
}
```

### Using the ELK Stack

```bash
# Start the ELK stack
cd ~/monitoring/elk
docker compose up -d

# Wait for Elasticsearch to be ready
until curl -s http://localhost:9200/_cluster/health | grep -q '"status":"green\|yellow"'; do
  echo "Waiting for Elasticsearch..."
  sleep 5
done

# Send a test log to Logstash
echo '{"message":"User login successful","user":"alice","level":"info"}' | nc localhost 5000

# Verify data in Elasticsearch
curl -s http://localhost:9200/logs-*/_search?pretty | head -30

# Access Kibana at http://localhost:5601
# Go to Management > Stack Management > Data Views
# Create a data view with pattern: logs-*
# Go to Discover to see your logs
```

---

## EFK Stack (Elasticsearch, Fluentd, Kibana)

The EFK stack replaces Logstash with Fluentd. Fluentd is lighter and is the standard log collector in Kubernetes.

### Fluentd Configuration

```xml
<!-- ~/monitoring/efk/fluentd/conf/fluent.conf -->

<!-- Accept logs over HTTP -->
<source>
  @type http
  port 9880
  bind 0.0.0.0
</source>

<!-- Accept logs forwarded from other Fluentd instances -->
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<!-- Collect Docker container logs -->
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag docker.*
  <parse>
    @type json
    time_key time
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</source>

<!-- Process and filter -->
<filter docker.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    environment "development"
  </record>
</filter>

<!-- Output to Elasticsearch -->
<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix fluentd-logs
  flush_interval 5s
  <buffer>
    @type memory
    flush_mode interval
    flush_interval 5s
    chunk_limit_size 5m
    retry_max_interval 30
  </buffer>
</match>
```

### EFK docker-compose.yml

```yaml
# ~/monitoring/efk/docker-compose.yml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - efk

  fluentd:
    build:
      context: ./fluentd
      dockerfile: Dockerfile
    container_name: fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"
      - "9880:9880"
    depends_on:
      - elasticsearch
    networks:
      - efk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - efk

volumes:
  es-data:

networks:
  efk:
    driver: bridge
```

### Fluentd Dockerfile (with Elasticsearch plugin)

```dockerfile
# ~/monitoring/efk/fluentd/Dockerfile
FROM fluent/fluentd:v1.16-1

USER root

# Install the Elasticsearch output plugin
RUN gem install fluent-plugin-elasticsearch --no-document --version 5.4.3

USER fluent
```

### Send Logs to Fluentd from Docker Containers

```bash
# Run any container with Fluentd logging driver
docker run -d \
  --name myapp \
  --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.myapp" \
  nginx:latest

# Or send logs via HTTP
curl -X POST -d 'json={"message":"hello from curl","level":"info"}' \
  http://localhost:9880/myapp.log
```

---

## Loki - Lightweight Log Aggregation

Loki is a log aggregation system by Grafana Labs. Unlike Elasticsearch, Loki only indexes labels (not the full log text), making it much cheaper to run.

```
┌──────────────────────────────────────────────────────────┐
│                     Loki Architecture                     │
│                                                           │
│  ┌──────────┐    push     ┌───────────┐                  │
│  │ Promtail  │───────────►│   Loki     │◄── Grafana      │
│  │ (Agent)   │            │  (Storage) │    (Query UI)    │
│  └──────────┘            └───────────┘                  │
│                                                           │
│  Loki indexes labels only, NOT the full log text.        │
│  Much cheaper and simpler than Elasticsearch.            │
└──────────────────────────────────────────────────────────┘
```

### Loki Configuration

```yaml
# ~/monitoring/loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h  # 7 days

analytics:
  reporting_enabled: false
```

### Promtail Configuration (Log Collector for Loki)

```yaml
# ~/monitoring/loki/promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Collect local log files
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          __path__: /var/log/syslog

  # Collect Docker container logs
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - json:
          expressions:
            log: log
            stream: stream
            time: time
      - labels:
          stream:
      - timestamp:
          source: time
          format: "2006-01-02T15:04:05.000000000Z"
      - output:
          source: log
```

### Querying Loki with LogQL

LogQL is to logs what PromQL is to metrics.

```logql
# All logs from the "syslog" job
{job="syslog"}

# Filter log lines containing "error"
{job="docker"} |= "error"

# Case-insensitive filter
{job="docker"} |~ "(?i)error"

# Exclude lines containing "debug"
{job="docker"} != "debug"

# Parse JSON logs and filter by field
{job="docker"} | json | level="error"

# Count log lines per minute
count_over_time({job="docker"} |= "error" [1m])

# Rate of error logs per second
rate({job="docker"} |= "error" [5m])

# Top 5 most frequent log messages
topk(5, count_over_time({job="docker"}[1h]))
```

---

## Full Monitoring Stack with Docker Compose

This is a complete, production-style monitoring stack combining Prometheus, Grafana, Alertmanager, Loki, and Promtail.

### Directory Structure

```
~/monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── alert_rules.yml
├── alertmanager/
│   └── alertmanager.yml
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yml
│       └── dashboards/
│           ├── dashboards.yml
│           └── node-exporter.json
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

### Complete docker-compose.yml

```yaml
# ~/monitoring/docker-compose.yml
version: "3.8"

services:
  # ---- Metrics ----

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node-exporter
    command:
      - "--path.rootfs=/host"
    volumes:
      - "/:/host:ro"
    pid: host
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped

  # ---- Alerting ----

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
    networks:
      - monitoring
    restart: unless-stopped

  # ---- Visualization ----

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
    networks:
      - monitoring
    restart: unless-stopped

  # ---- Logging ----

  loki:
    image: grafana/loki:latest
    container_name: loki
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    networks:
      - monitoring
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    networks:
      - monitoring
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
  loki-data:

networks:
  monitoring:
    driver: bridge
```

### Grafana Provisioning - Auto-configure Data Sources

```yaml
# ~/monitoring/grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true

  - name: Alertmanager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093
    editable: true
    jsonData:
      implementation: prometheus
```

### Grafana Provisioning - Auto-load Dashboards

```yaml
# ~/monitoring/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: "default"
    orgId: 1
    folder: "Provisioned"
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: false
```

### Launch and Verify

```bash
# Start the entire stack
cd ~/monitoring
docker compose up -d

# Verify all containers are running
docker compose ps
# NAME            STATUS
# prometheus      Up
# node-exporter   Up
# cadvisor        Up
# alertmanager    Up
# grafana         Up
# loki            Up
# promtail        Up

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep health

# Check Loki is receiving logs
curl -s http://localhost:3100/ready
# Output: ready

# Access the UIs:
# Prometheus:   http://localhost:9090
# Grafana:      http://localhost:3000  (admin / admin123)
# Alertmanager: http://localhost:9093
# cAdvisor:     http://localhost:8080

# Stop the stack
docker compose down

# Stop and remove all data
docker compose down -v
```

---

## Projects

### Project 1: Set Up Prometheus + Grafana Monitoring Stack

**Goal:** Monitor a web application and the host system with Prometheus and visualize metrics in Grafana.

**Steps:**

```bash
# 1. Create project directory
mkdir -p ~/project-monitoring/{prometheus,grafana/provisioning/datasources}
cd ~/project-monitoring

# 2. Create a sample Python app with metrics
cat > app.py << 'PYEOF'
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import random, time

# Define metrics
REQUEST_COUNT = Counter(
    "app_requests_total",
    "Total request count",
    ["method", "endpoint", "status"]
)
REQUEST_LATENCY = Histogram(
    "app_request_duration_seconds",
    "Request latency in seconds",
    ["endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)
ACTIVE_USERS = Gauge(
    "app_active_users",
    "Number of active users"
)

def simulate_traffic():
    endpoints = ["/api/users", "/api/orders", "/api/products", "/health"]
    methods = ["GET", "POST", "PUT"]
    while True:
        endpoint = random.choice(endpoints)
        method = random.choice(methods)
        status = random.choices(["200", "201", "400", "500"], weights=[70, 15, 10, 5])[0]
        duration = random.uniform(0.01, 2.0)

        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status=status).inc()
        REQUEST_LATENCY.labels(endpoint=endpoint).observe(duration)
        ACTIVE_USERS.set(random.randint(10, 200))

        time.sleep(random.uniform(0.1, 0.5))

if __name__ == "__main__":
    start_http_server(8000)
    print("Metrics server running on :8000/metrics")
    simulate_traffic()
PYEOF

# 3. Create Dockerfile for the app
cat > Dockerfile << 'DEOF'
FROM python:3.11-slim
RUN pip install prometheus-client
COPY app.py /app.py
CMD ["python", "/app.py"]
DEOF

# 4. Create prometheus.yml (see configuration example above)

# 5. Create docker-compose.yml with Prometheus, Grafana, Node Exporter, and the app

# 6. Start the stack
docker compose up -d

# 7. Open Grafana, add Prometheus data source, and create a dashboard
#    with panels for request rate, error rate, latency, and active users
```

**Verification checklist:**
- [ ] All containers are running (`docker compose ps`)
- [ ] Prometheus shows all targets as UP at http://localhost:9090/targets
- [ ] App metrics visible at http://localhost:8000/metrics
- [ ] Grafana dashboard displays request rate, error rate, latency, and active users
- [ ] PromQL queries return data in Prometheus web UI

---

### Project 2: Set Up Centralized Logging

**Goal:** Collect logs from multiple Docker containers into a centralized Loki instance and explore them in Grafana.

**Steps:**

```bash
# 1. Create project directory
mkdir -p ~/project-logging/{loki,promtail}
cd ~/project-logging

# 2. Create loki-config.yml (see Loki section above)

# 3. Create promtail-config.yml (see Promtail section above)

# 4. Create a docker-compose.yml with:
#    - Loki
#    - Promtail
#    - Grafana (with Loki data source provisioned)
#    - Two sample apps that generate logs

# 5. Create a sample app that writes structured logs
cat > logger-app.py << 'PYEOF'
import json, time, random, sys

levels = ["INFO", "WARN", "ERROR", "DEBUG"]
messages = [
    "User logged in",
    "Payment processed",
    "Database query slow",
    "Cache miss",
    "API rate limit hit",
    "Connection timeout",
    "Order created",
    "Email sent",
]

while True:
    log = {
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "level": random.choices(levels, weights=[60, 15, 10, 15])[0],
        "message": random.choice(messages),
        "user_id": random.randint(1, 1000),
        "duration_ms": random.randint(1, 5000),
    }
    print(json.dumps(log), flush=True)
    time.sleep(random.uniform(0.5, 3.0))
PYEOF

# 6. Start the stack
docker compose up -d

# 7. Open Grafana, go to Explore, select Loki data source
# 8. Query logs with LogQL:
#    {container="logger-app"} |= "ERROR"
#    {container="logger-app"} | json | level="ERROR"
#    rate({container="logger-app"} |= "ERROR" [5m])
```

**Verification checklist:**
- [ ] Loki is ready (`curl http://localhost:3100/ready`)
- [ ] Grafana can query Loki data source
- [ ] Logs from containers appear in Grafana Explore
- [ ] LogQL queries filter logs correctly by level

---

### Project 3: Configure Alert Rules and Notifications

**Goal:** Set up Prometheus alert rules and Alertmanager to send notifications when things go wrong.

**Steps:**

```bash
# 1. Create project directory
mkdir -p ~/project-alerting/{prometheus,alertmanager}
cd ~/project-alerting

# 2. Write alert_rules.yml with rules for:
#    - Instance down (up == 0)
#    - High CPU (> 80% for 5m)
#    - High memory (> 85% for 5m)
#    - High error rate from the sample app

# 3. Configure alertmanager.yml with a Slack webhook receiver
#    (or use a webhook receiver like webhook.site for testing)

# 4. Add to prometheus.yml:
#    alerting:
#      alertmanagers:
#        - static_configs:
#            - targets: ["alertmanager:9093"]
#    rule_files:
#      - "alert_rules.yml"

# 5. Start the stack with docker compose

# 6. Trigger an alert by stopping a monitored service
docker stop node-exporter

# 7. After 1 minute, check:
#    - Prometheus Alerts page: http://localhost:9090/alerts
#      InstanceDown alert should be "firing"
#    - Alertmanager: http://localhost:9093
#      Alert notification should be visible
#    - Slack channel should receive the notification

# 8. Restart the service and verify the alert resolves
docker start node-exporter
```

**Verification checklist:**
- [ ] Alert rules load without errors in Prometheus
- [ ] Stopping a target triggers InstanceDown alert within the configured `for` duration
- [ ] Alert appears in Alertmanager UI
- [ ] Notification is sent to configured receiver
- [ ] Restarting the target resolves the alert

---

## Quick Reference

| Tool | Purpose | Default Port |
|------|---------|-------------|
| Prometheus | Metrics collection and storage | 9090 |
| Grafana | Visualization and dashboards | 3000 |
| Alertmanager | Alert routing and notifications | 9093 |
| Node Exporter | Host system metrics | 9100 |
| cAdvisor | Container metrics | 8080 |
| Elasticsearch | Log storage and search | 9200 |
| Logstash | Log processing pipeline | 5044, 5000 |
| Kibana | Log visualization | 5601 |
| Fluentd | Log collection and forwarding | 24224 |
| Loki | Lightweight log storage | 3100 |
| Promtail | Log collector for Loki | 9080 |
