[🏠 Home](../README.md) · [Monitoring](README.md)

# 📊 Monitoring & Observability Stack — Prometheus, Grafana, Loki & ELK

> Quick notes + basic/intermediate hands-on scenarios for learning the most common open-source observability stack.
> Covers: **Prometheus** (metrics) → **Grafana** (visualization) → **Loki** (lightweight logs) → **ELK** (heavyweight logs/search).

---

## Table of Contents

1. [Why Observability — The Three Pillars](#1-why-observability--the-three-pillars)
2. [Prometheus](#2-prometheus)
3. [Grafana](#3-grafana)
4. [Loki](#4-loki)
5. [ELK Stack (Elasticsearch, Logstash, Kibana)](#5-elk-stack-elasticsearch-logstash-kibana)
6. [Loki vs ELK — Which One?](#6-loki-vs-elk--which-one)
7. [Combined Production Stack](#7-combined-production-stack)
8. [Quick Reference Card](#8-quick-reference-card)

---

## 1. Why Observability — The Three Pillars

```
                     ┌─────────────────────────────┐
                     │        OBSERVABILITY         │
                     └─────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                           │
   ┌─────────┐               ┌───────────┐               ┌───────────┐
   │ METRICS │               │   LOGS    │               │  TRACES   │
   │(numbers │               │ (events / │               │ (request  │
   │over time)│              │  text)    │               │  journey) │
   └─────────┘               └───────────┘               └───────────┘
        │                          │                           │
   Prometheus                 Loki / ELK                  Jaeger / Tempo
   "Is it broken?"          "Why is it broken?"        "Where is it slow?"
        │                          │                           │
        └──────────────────────────┴──────────────────────────┘
                                   │
                              Grafana
                     (single pane of glass to view all 3)
```

| Pillar | Answers | Tool in this guide |
|--------|---------|---------------------|
| Metrics | "Is CPU high? Is error rate up?" | Prometheus |
| Logs | "What exactly happened at 3:14 AM?" | Loki / ELK |
| Traces | "Which microservice added the latency?" | (not covered here — Jaeger/Tempo) |
| Visualization | "Show me all of the above in one place" | Grafana |

> ⚠️ **Gotcha:** Metrics tell you *something* is wrong (a graph spikes); logs tell you *why*. Teams that only set up dashboards and never centralize logs end up SSH-ing into servers during incidents — defeating the purpose of observability.

---

## 2. Prometheus

### 2.1 Quick Notes

Prometheus is a **pull-based** time-series metrics system — it scrapes `/metrics` HTTP endpoints on a schedule rather than waiting for apps to push data.

| Concept | Description |
|---------|-------------|
| **Scrape** | Prometheus polls a target's `/metrics` endpoint every N seconds |
| **Target** | Any service exposing metrics in Prometheus text format |
| **Exporter** | Adapter that converts a system/app's stats into Prometheus format (`node_exporter`, `blackbox_exporter`, `mysqld_exporter`) |
| **Metric types** | `Counter` (only increases), `Gauge` (up/down), `Histogram` (bucketed observations), `Summary` (client-side quantiles) |
| **Label** | Key-value pair attached to a metric for filtering (`instance`, `job`, `status`) |
| **PromQL** | Query language for selecting/aggregating time-series |
| **Alertmanager** | Separate component that receives firing alerts and routes them (Slack, PagerDuty, email, dedup, grouping) |
| **TSDB** | Prometheus's local on-disk time-series database (not meant for years of retention) |

### 2.2 Architecture — Mental Model

```
   ┌────────────┐   scrape /metrics   ┌────────────┐
   │  Node App   │◄────────────────────│            │
   │ (port 8080) │                     │            │
   └────────────┘                     │            │
   ┌────────────┐   scrape /metrics   │ PROMETHEUS │──── stores ───► TSDB (local disk)
   │Node Exporter│◄────────────────────│  (server)  │
   │ (port 9100) │                     │            │
   └────────────┘                     │            │──── evaluates ──► alerts.yml (rules)
   ┌────────────┐   scrape /metrics   │            │
   │  MySQL      │◄────────────────────│            │
   │  Exporter   │                     └─────┬──────┘
   └────────────┘                            │
                                     fires alert
                                              │
                                              ▼
                                     ┌─────────────────┐
                                     │  Alertmanager    │──► Slack / PagerDuty / Email
                                     └─────────────────┘
                                              ▲
                                              │
                                     ┌─────────────────┐
                                     │     Grafana      │  (reads via PromQL, for dashboards)
                                     └─────────────────┘
```

> Note the arrow direction: Prometheus **reaches out** to targets. This is the opposite of Loki/ELK, where agents **push** logs to a central server.

### 2.3 PromQL Cheatsheet

```promql
# CPU usage % (100 - idle)
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory used (bytes)
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# HTTP 5xx error rate
rate(http_requests_total{status=~"5.."}[5m])

# Request rate per second
rate(http_requests_total[5m])

# 99th percentile latency from a histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Is a target up? (0 or 1)
up{job="node"}

# Top 5 pods by memory
topk(5, container_memory_usage_bytes)
```

### 2.4 Basic Scenario — Install Prometheus + Node Exporter

```bash
# 1. Create a dedicated user (least privilege)
sudo useradd --no-create-home --shell /bin/false prometheus

# 2. Download and install the binary
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar -xzf prometheus-2.52.0.linux-amd64.tar.gz
sudo cp prometheus-2.52.0.linux-amd64/{prometheus,promtool} /usr/local/bin/

# 3. Config
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo tee /etc/prometheus/prometheus.yml <<EOF
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs: [{ targets: ['localhost:9090'] }]
  - job_name: 'node'
    static_configs: [{ targets: ['localhost:9100'] }]
EOF

# 4. systemd service
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
After=network.target
[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now prometheus

# 5. Install Node Exporter (system metrics)
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.8.0.linux-amd64.tar.gz
sudo cp node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
After=network.target
[Service]
ExecStart=/usr/local/bin/node_exporter
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now node_exporter
```

**Verify:** Open `http://<vm-ip>:9090` → **Status → Targets**. Both `prometheus` and `node` should show **UP**.

### 2.5 Intermediate Scenario — Alerting Rules

```yaml
# /etc/prometheus/alerts.yml
groups:
  - name: infrastructure
    rules:
      - alert: HighCPU
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "High CPU on {{ $labels.instance }}"

      - alert: DiskAlmostFull
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 10m
        labels: { severity: critical }
        annotations:
          summary: "Disk nearly full on {{ $labels.instance }}"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "Target {{ $labels.instance }} is down"
```

```yaml
# add to prometheus.yml
rule_files:
  - /etc/prometheus/alerts.yml
alerting:
  alertmanagers:
    - static_configs: [{ targets: ['localhost:9093'] }]
```

```bash
promtool check rules /etc/prometheus/alerts.yml   # validate BEFORE reloading
sudo systemctl reload prometheus
```

### 2.6 DevOps Gotchas — Prometheus

> ⚠️ **Gotcha: High-cardinality labels blow up memory.** Never put unbounded values (user IDs, full URLs, request IDs) into a label. Each unique label combination creates a new time-series — 100k unique labels can OOM Prometheus.

> ⚠️ **Gotcha: Local TSDB is not for long-term storage.** Default retention is 15 days. For longer history, use remote-write to Thanos, Cortex, or Mimir — don't just bump `--storage.tsdb.retention.time` indefinitely on a single node.

> ⚠️ **Gotcha: `rate()` needs at least 2 data points.** If your scrape interval is 15s and you query `rate(x[10s])`, you'll get gaps/`NaN`. Always make the range at least 4x the scrape interval.

> ⚠️ **Gotcha: Counters reset on restart.** A `Counter` resets to 0 when the app restarts. Always wrap counters in `rate()` or `increase()` — never graph the raw counter value, it produces a misleading sawtooth.

> ⚠️ **Gotcha: Prometheus doesn't discover new targets by magic.** In Kubernetes, if you don't add a `ServiceMonitor` or scrape annotation, Prometheus will never know a new pod exists — it isn't push-based.

---

## 3. Grafana

### 3.1 Quick Notes

Grafana is a visualization layer that sits **on top of** data sources — it doesn't store data itself (mostly). It queries Prometheus, Loki, Elasticsearch, MySQL, CloudWatch, etc., and renders panels/dashboards.

| Concept | Description |
|---------|-------------|
| **Data source** | Backend Grafana queries (Prometheus, Loki, Elasticsearch...) |
| **Dashboard** | A collection of panels, usually JSON under the hood |
| **Panel** | One visualization (graph, stat, table, heatmap, logs) |
| **Variable** | Dropdown template filter, e.g. `$instance`, reusable across panels |
| **Alert rule** | Grafana can alert independently of Alertmanager (Grafana-managed alerting) |
| **Provisioning** | Define data sources/dashboards as YAML/JSON files (Dashboards-as-Code) |
| **Folder** | Organizes dashboards by team/project |

### 3.2 Architecture — Mental Model

```
        ┌───────────────────────────────────────────────────┐
        │                      GRAFANA                        │
        │   (dashboards, alerting, user auth, panels)         │
        └───────────────────────────────────────────────────┘
              │            │              │            │
        query │      query │        query │      query │
              ▼            ▼              ▼            ▼
      ┌────────────┐ ┌───────────┐ ┌──────────────┐ ┌─────────┐
      │ Prometheus  │ │   Loki    │ │Elasticsearch │ │  MySQL  │
      │  (metrics)  │ │  (logs)   │ │  (logs/docs) │ │ (biz DB)│
      └────────────┘ └───────────┘ └──────────────┘ └─────────┘
```

Grafana never scrapes or collects data itself — it is a **read-only glass** over other systems.

### 3.3 Basic Scenario — Install Grafana + Connect to Prometheus

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://apt.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

**Access:** `http://<ip>:3000` — default login `admin` / `admin` (you'll be forced to change it).

**Add Prometheus data source:** Settings → Data Sources → Add → Prometheus → URL `http://localhost:9090` → Save & Test.

**Import a pre-built dashboard:** Dashboards → Import → ID `1860` (Node Exporter Full) → pick your Prometheus data source → Import.

### 3.4 Intermediate Scenario — Provisioning as Code + Alerts

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: false
```

```yaml
# /etc/grafana/provisioning/dashboards/default.yaml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Provisioned'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

```bash
sudo mkdir -p /var/lib/grafana/dashboards
# drop dashboard JSON exports here
sudo systemctl restart grafana-server
```

**Grafana-managed alert (UI):** Edit panel → Alert tab → condition `WHEN last() OF query(A) IS ABOVE 85` → attach a **Contact point** (Slack webhook / email) under Alerting → Contact points.

### 3.5 DevOps Gotchas — Grafana

> ⚠️ **Gotcha: Dashboards created in the UI are NOT version controlled by default.** If someone edits a panel and doesn't export the JSON, the change is lost on the next provisioning sync (provisioned dashboards are treated as read-only "from disk" — UI edits to them don't persist across restarts unless you explicitly save).

> ⚠️ **Gotcha: Default admin/admin credentials left unchanged.** Grafana exposed to the internet with default credentials is a classic breach vector — always change the password and put it behind SSO/reverse proxy auth in production.

> ⚠️ **Gotcha: Data source proxy mode vs browser mode.** "Server (proxy)" access means Grafana's backend calls the data source; "Browser" means the user's browser calls it directly — mixing these up causes confusing CORS/network errors depending on where Grafana is hosted vs where users sit.

> ⚠️ **Gotcha: Time range mismatches make graphs look "empty."** A dashboard set to "Last 6 hours" while your Prometheus retention or Loki retention is shorter will silently show partial/no data — always check the data source's actual retention window.

---

## 4. Loki

### 4.1 Quick Notes

Loki is Grafana Labs' log aggregation system — deliberately modeled after Prometheus, but for logs. It **does not** index full log text; it only indexes **labels**, making it cheap to run compared to Elasticsearch.

| Concept | Description |
|---------|-------------|
| **Log stream** | All log lines sharing the exact same label set |
| **Label** | Key-value metadata (`job`, `host`, `namespace`) — keep cardinality **low** |
| **Promtail** | Agent that tails log files and ships them to Loki |
| **LogQL** | Loki's query language (label selector + filter + optional metric extraction) |
| **Chunk** | Compressed block of log lines stored on disk/object storage |

### 4.2 Architecture — Mental Model

```
   ┌──────────────┐  tail /var/log/*.log   ┌──────────────┐  push (HTTP)   ┌─────────┐
   │  App / Server │──────────────────────►│   Promtail    │───────────────►│  Loki   │
   │  (log files)  │                        │   (agent)     │                │(storage)│
   └──────────────┘                        └──────────────┘                └────┬────┘
                                                                                  │
                                                                          LogQL query
                                                                                  │
                                                                                  ▼
                                                                            ┌──────────┐
                                                                            │  Grafana │
                                                                            └──────────┘
```

Compare this to Prometheus: **Promtail pushes** to Loki (unlike Prometheus, which pulls). Loki is push-based on the log-shipping side.

### 4.3 LogQL Cheatsheet

```logql
# All logs for the nginx job
{job="nginx"}

# Simple text filter
{job="nginx"} |= "ERROR"

# Regex filter
{job="nginx"} |~ "5[0-9]{2}"

# Parse fields out of a log line, then filter numerically
{job="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <_>` | status >= 500

# Count matching lines over a window (a metric derived from logs!)
count_over_time({job="nginx"} |= "ERROR" [5m])

# Rate of log lines per second
rate({job="nginx"}[5m])
```

### 4.4 Basic Scenario — Loki + Promtail via Docker Compose

```yaml
# docker-compose.yml
version: "3"
services:
  loki:
    image: grafana/loki:2.9.0
    ports: ["3100:3100"]
    command: -config.file=/etc/loki/local-config.yaml
    volumes: [loki-data:/loki]

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log:ro
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml
    depends_on: [loki]

volumes:
  loki-data:
```

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels: { job: varlogs, host: myserver, __path__: /var/log/*.log }
  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels: { job: nginx, __path__: /var/log/nginx/*.log }
```

```bash
docker compose up -d
```

**In Grafana:** Add Loki data source (`http://localhost:3100`) → Explore → query `{job="varlogs"}`.

### 4.5 Intermediate Scenario — Parse Nginx Logs + Alert on Error Rate

```yaml
# promtail-config.yaml — with pipeline stages
scrape_configs:
  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels: { job: nginx, __path__: /var/log/nginx/access.log }
    pipeline_stages:
      - regex:
          expression: '^(?P<ip>\S+) .* "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<bytes>\d+)'
      - labels:
          method:
          status:
```

```logql
# Alert when 5xx error rate > 5/min (used in a Grafana or Loki Ruler alert)
sum(rate({job="nginx"} |= "\" 5" [1m])) > 5
```

### 4.6 DevOps Gotchas — Loki

> ⚠️ **Gotcha: Putting high-cardinality data into labels defeats Loki's design.** Never label by `user_id`, `request_id`, `trace_id`, or IP address — that turns every unique value into its own stream, which is exactly the disk/memory explosion Loki was built to avoid. Put those in the log line body and filter/parse with LogQL instead.

> ⚠️ **Gotcha: Out-of-order log ingestion is rejected by default.** If Promtail (or multiple agents) sends logs whose timestamps are older than already-ingested logs for the same stream, Loki will reject them (`entry out of order`) unless `unordered_writes` is configured.

> ⚠️ **Gotcha: Loki is not full-text search.** LogQL filters are evaluated by scanning chunks (grep-like), not an inverted index. Broad, unfiltered queries over huge time ranges (`{job=~".+"}` for 30 days) will be slow and expensive — always narrow with labels first.

> ⚠️ **Gotcha: Promtail position file gets lost on container restart if not persisted.** Without a volume for `/tmp/positions.yaml`, Promtail may re-read (duplicate) or skip logs after a restart.

---

## 5. ELK Stack (Elasticsearch, Logstash, Kibana)

### 5.1 Quick Notes

ELK (also seen as "Elastic Stack") is a full-text search and log analytics platform — heavier than Loki, but far more powerful for ad-hoc search, complex parsing, and BI-style dashboards.

| Concept | Description |
|---------|-------------|
| **Elasticsearch** | Distributed search engine; documents stored as JSON, fully indexed (every field is searchable) |
| **Index** | Like a database table — logs are grouped into daily/rolling indices |
| **Logstash** | ETL pipeline: collect → parse (grok) → transform → output |
| **Filebeat** | Lightweight log shipper (simpler alternative to Logstash for tailing files) |
| **Kibana** | Web UI for search, dashboards, alerting on top of Elasticsearch |
| **ILM** | Index Lifecycle Management — automatically rolls over and deletes old indices |
| **KQL** | Kibana Query Language for filtering in Discover/Dashboards |

### 5.2 Architecture — Mental Model

```
   ┌──────────────┐  tail logs   ┌───────────┐  parse/enrich  ┌───────────────┐
   │ App / Server  │─────────────►│  Filebeat  │───────────────►│   Logstash    │
   │ (log files)   │              │  (agent)   │                │ (grok, filter)│
   └──────────────┘              └───────────┘                └───────┬───────┘
                                                                        │ index
                                                                        ▼
                                                                ┌───────────────┐
                                                                │ Elasticsearch │
                                                                │ (search index)│
                                                                └───────┬───────┘
                                                                        │ query
                                                                        ▼
                                                                 ┌───────────┐
                                                                 │  Kibana   │
                                                                 └───────────┘
```

> Filebeat can also skip Logstash entirely and ship straight to Elasticsearch (with "ingest pipelines" doing the parsing) — this is common for simpler setups.

### 5.3 Basic Scenario — Run ELK via Docker Compose

```yaml
# docker-compose.yml
version: "3"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports: ["9200:9200"]
    volumes: [es-data:/usr/share/elasticsearch/data]

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    ports: ["5601:5601"]
    environment: [ELASTICSEARCH_HOSTS=http://elasticsearch:9200]
    depends_on: [elasticsearch]

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.0
    volumes: ["./logstash.conf:/usr/share/logstash/pipeline/logstash.conf"]
    ports: ["5044:5044"]
    depends_on: [elasticsearch]

volumes:
  es-data:
```

```conf
# logstash.conf
input {
  beats { port => 5044 }
}
filter {
  if [log][file][path] =~ "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
    date { match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"] target => "@timestamp" }
    mutate { convert => { "response" => "integer" "bytes" => "integer" } }
  }
}
output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

```bash
docker compose up -d
# Kibana: http://localhost:5601
```

### 5.4 Intermediate Scenario — Filebeat + Kibana Dashboard

```yaml
# filebeat.yml (on your app server)
filebeat.inputs:
  - type: filestream
    id: nginx-access
    paths: ["/var/log/nginx/access.log"]
    tags: ["nginx", "access"]

processors:
  - add_host_metadata: { when.not.contains.tags: forwarded }

output.logstash:
  hosts: ["your-logstash-host:5044"]
```

```bash
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.13.0-amd64.deb
sudo dpkg -i filebeat-8.13.0-amd64.deb
sudo cp filebeat.yml /etc/filebeat/filebeat.yml
sudo systemctl enable --now filebeat
```

**In Kibana:**
1. Stack Management → Index Patterns → create `nginx-logs-*`
2. Discover → search: `response:500` or `tags:error`
3. Dashboard → Create → add Lens panels (bar chart of `response` codes, line chart of requests over time)

**KQL examples:**
```
response >= 500
http.request.method : "POST"
tags : "nginx" and response : 404
@timestamp >= now-1h
```

### 5.5 DevOps Gotchas — ELK

> ⚠️ **Gotcha: Elasticsearch is memory-hungry.** The JVM heap should be ~50% of available RAM, capped at 32 GB (beyond that, Java loses "compressed oops" and performance degrades). Running ES with default heap on a small VM causes OOM kills.

> ⚠️ **Gotcha: Unbounded indices fill the disk.** Without ILM policies, daily indices accumulate forever. Set up rollover + delete phases from day one, not after the disk hits 100%.

> ⚠️ **Gotcha: Mapping explosion.** If you log unstructured JSON with dynamically changing field names (e.g., a field per user ID), Elasticsearch will create a new mapped field for each one — eventually hitting the default 1000-field mapping limit and rejecting new documents.

> ⚠️ **Gotcha: Grok patterns are brittle.** A single log format change upstream (e.g., a new field inserted mid-line) silently breaks Logstash's grok match, and those lines land in `_grokparsefailure` — unmonitored, this becomes a silent data-loss bug.

> ⚠️ **Gotcha: `xpack.security.enabled=false` is a dev-only shortcut.** Never run Elasticsearch without authentication in anything reachable from a shared network — unauthenticated ES clusters are a well-known ransomware target.

---

## 6. Loki vs ELK — Which One?

| Criteria | Loki | ELK |
|----------|------|-----|
| Indexing | Labels only (cheap) | Full-text (every field indexed) |
| Resource usage | Low | High (JVM heap, memory) |
| Query power | Good for filtering by known labels + grep-like text search | Powerful ad-hoc full-text search & aggregations |
| Setup complexity | Simple | Higher (JVM tuning, ILM, mappings) |
| Best fit | Kubernetes/cloud-native, "logs next to metrics in Grafana" | Enterprise log analytics, security/SIEM use cases, complex search |
| Query language | LogQL | Lucene/DSL/KQL |
| Native Grafana integration | First-class (same vendor) | Via Elasticsearch data source plugin |

> ⚠️ **Gotcha:** Don't run both "just in case." Pick one based on actual need — running ELK for what could be simple Loki queries wastes significant infrastructure budget; running Loki when you need SIEM-grade full-text search across unpredictable fields will frustrate your security team.

---

## 7. Combined Production Stack

```
                         ┌───────────────────────────────────────┐
                         │              GRAFANA                    │
                         │  (single pane: metrics + logs + alerts) │
                         └───────┬───────────────┬─────────────────┘
                                 │               │
                     PromQL query│               │LogQL / ES query
                                 ▼               ▼
                        ┌────────────┐    ┌────────────┐
                        │ Prometheus │    │ Loki / ELK  │
                        └─────┬──────┘    └─────┬──────┘
                              │ scrape          │ push
                    ┌─────────┴─────────┐  ┌────┴─────────┐
                    │  node_exporter     │  │  Promtail /   │
                    │  app /metrics      │  │  Filebeat     │
                    │  cAdvisor (K8s)    │  │  (log tailer) │
                    └────────────────────┘  └───────────────┘
                              ▲                     ▲
                              └─────────┬───────────┘
                                        │
                                 Your application
                                (VM / container / pod)
```

**Alert flow:**
```
Prometheus rule fires ──► Alertmanager ──► groups/dedups ──► Slack / PagerDuty / Email
Grafana alert fires    ──► Contact point ──► Slack / PagerDuty / Email / Webhook
```

---

## 8. Quick Reference Card

| Tool | Default Port | Purpose | Query Language | Push or Pull |
|------|--------------|---------|-----------------|--------------|
| Prometheus | 9090 | Metrics collection | PromQL | Pull |
| Node Exporter | 9100 | System metrics | — | Pull (scraped) |
| Alertmanager | 9093 | Alert routing/dedup | — | — |
| Grafana | 3000 | Visualization | Depends on source | Reads via query |
| Loki | 3100 | Log aggregation | LogQL | Push (via Promtail) |
| Promtail | 9080 | Log shipping agent | — | Push |
| Elasticsearch | 9200 | Log/doc search & storage | Query DSL / KQL | Push (via Beats/Logstash) |
| Kibana | 5601 | ES visualization | KQL | Reads via query |
| Logstash | 5044 (beats input) | Log ETL pipeline | Grok patterns | Push |

**Common stack combos:**
- **Cloud-native / Kubernetes:** Prometheus + Grafana + Loki + Promtail (kube-prometheus-stack Helm chart bundles most of this)
- **Enterprise log analytics / SIEM:** Filebeat + Logstash + Elasticsearch + Kibana
- **Hybrid:** Prometheus + Grafana for metrics, ELK for logs (common when a security team already owns Elastic)

---

[🏠 Back to Home](../README.md) · [Monitoring Index](README.md)
