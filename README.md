# HNG Stage 6 — Production-Grade Observability Platform

A complete observability and reliability platform built with the LGTM stack.
Monitors system health, tracks SLOs and error budgets, measures DORA metrics,
and routes structured alerts to Slack.

---

## One-command deployment

```bash
# Clone the repo
git clone https://github.com/Dreybest/hng-stage6-observability.git
cd hng-stage6-observability

# Add your Slack webhook URL
echo "SLACK_WEBHOOK_URL=your-webhook-url-here" > .env

# Deploy the full stack
docker compose up -d
```

The entire LGTM stack will be running within 2 minutes.

---

## Stack overview

| Tool | Role | Port |
|---|---|---|
| Prometheus | Metrics collection and storage | 9090 |
| Loki | Log aggregation | 3100 |
| Tempo | Distributed trace storage | 3200 |
| Grafana | Unified observability frontend | 3000 |
| Alertmanager | Alert routing to Slack | 9093 |
| Node Exporter | System metrics (CPU, RAM, Disk) | 9100 |
| Blackbox Exporter | Uptime and SSL probing | 9115 |
| OpenTelemetry Collector | Log and trace pipeline | 4317, 4318 |

---

## Architecture
App Server                          Monitoring Server
──────────────────                  ──────────────────────────────
Your Application  ──── metrics ───► Prometheus
Node Exporter     ──── scrape  ───► Prometheus
OTel Collector    ──── logs    ───► Loki
OTel Collector    ──── traces  ───► Tempo
GitHub Actions    ──── metrics ───► Prometheus
Grafana ◄── all datasources
Alertmanager ──► Slack
Blackbox Exporter ◄── probes ────── Monitoring Server

---

## Grafana dashboards

All dashboards are provisioned as code from `grafana/dashboards/`.
They load automatically on startup — no manual configuration needed.

| Dashboard | File | UID |
|---|---|---|
| Node Exporter — System Health | `node-exporter.json` | node-exporter |
| Blackbox Exporter — Uptime & SSL | `blackbox.json` | blackbox |
| SLO & Error Budget | `slo-error-budget.json` | slo-error-budget |
| DORA Metrics | `dora-metrics.json` | dora-metrics |
| Unified Observability — Drill Down | `unified-observability.json` | unified-observability |

### Accessing dashboards
Grafana:        http://YOUR_SERVER_IP:3000
Username:       admin
Password:       admin123

### Log and trace drill-down
1. Open the Unified Observability dashboard
2. Find a metric spike in the top row stat panels
3. Scroll down to the Live Logs panel
4. Find the error log from that time window
5. Click the `trace_id` link in the log line
6. Tempo opens showing the full trace breakdown
7. Identify the exact service and endpoint responsible

---

## Four Golden Signals — SLI definitions

### Latency
```promql
probe_duration_seconds{job="blackbox-exporter"}
```
Measures how long HTTP probes take to complete.
SLO: 95% of requests complete under 500ms.

### Traffic
```promql
rate(probe_success{job="blackbox-exporter"}[5m])
```
Measures the rate of probe attempts per second.

### Errors
```promql
1 - avg_over_time(probe_success{job="blackbox-exporter"}[5m])
```
Measures the rate of failed probes.
SLO: 99% of probes must succeed (error rate below 1%).

### Saturation
```promql
# CPU saturation
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory saturation
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk saturation
(1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} /
node_filesystem_size_bytes{fstype!="tmpfs"})) * 100
```

---

## SLO targets and error budgets

| SLO | Target | Window | Error budget |
|---|---|---|---|
| Availability | 99.5% | 30 days | 216 minutes/month |
| Latency | 95% under 500ms | 30 days | 5% of requests |
| Error Rate | 99% success | 30 days | 1% of requests |

---

## Error Budget Policy

### What is the error budget?
The error budget is the maximum amount of unreliability allowed
before the SLO is breached. With a 99.5% availability SLO:
Error Budget = (1 - 0.995) x 30 days
= 0.005 x 43,200 minutes
= 216 minutes per month

### Policy by consumption level

| Budget consumed | Status | Action |
|---|---|---|
| 0–50% | Healthy | Ship features freely. Normal operations. |
| 50–75% | Caution | Monitor closely. Reduce deployment frequency. |
| 75–100% | Warning | Stop non-critical deployments. Focus on stability. |
| 100% breached | Critical | Feature freeze. Reliability sprint. Notify stakeholders. |

### Who owns the decision?
- **0–50%:** Engineering team owns decisions autonomously
- **50–100%:** Team lead must be informed before major deployments
- **100% breached:** Team lead declares feature freeze

### Review cadence
SLOs are reviewed at the end of every month. The team asks:
1. Was the SLO met this month?
2. Is the target still meaningful?
3. Do thresholds need adjusting based on what we learned?

---

## Alert rules

All alert rules are version-controlled in `rules/alerts.yml`.
Alerts are never configured in Grafana UI.

| Alert | Severity | Condition |
|---|---|---|
| ServerDown | Critical | Probe fails for 2+ minutes |
| CPUHighWarning | Warning | CPU above 80% for 5+ minutes |
| CPUHighCritical | Critical | CPU above 90% for 10+ minutes |
| MemoryHighWarning | Warning | Memory above 80% for 5+ minutes |
| MemoryHighCritical | Critical | Memory above 90% for 5+ minutes |
| DiskHighWarning | Warning | Disk above 75% for 5+ minutes |
| DiskHighCritical | Critical | Disk above 90% for 5+ minutes |
| SLOFastBurn | Critical | 2% budget consumed in 1 hour |
| SLOSlowBurn | Warning | 5% budget consumed in 6 hours |
| HighChangeFailureRate | Critical | CFR above 5% over 7 days |

---

## Runbooks

Every alert has a corresponding runbook in `runbooks/`:

| Runbook | Alerts covered |
|---|---|
| `server-down.md` | ServerDown |
| `cpu-high.md` | CPUHighWarning, CPUHighCritical |
| `memory-high.md` | MemoryHighWarning, MemoryHighCritical |
| `disk-high.md` | DiskHighWarning, DiskHighCritical |
| `slo-burn-rate.md` | SLOFastBurn, SLOSlowBurn |
| `high-cfr.md` | HighChangeFailureRate |
| `slo-breach.md` | General SLO breach guidance |

---

## DORA metrics

| Metric | Description | Elite benchmark |
|---|---|---|
| Deployment Frequency | How often deployments happen | Multiple per day |
| Lead Time for Changes | Commit to production | Under 1 hour |
| Change Failure Rate | % of deployments that fail | Under 5% |
| Mean Time to Restore | Time from alert to resolution | Under 1 hour |

---

## Toil identified

### Toil 1 — Manual deployment verification
**Problem:** Someone manually checks if a deployment succeeded.
**Solution:** Post-deployment health check step in GitHub Actions
that automatically probes the service after every deployment.

### Toil 2 — Manual rollback process
**Problem:** Rolling back requires SSH access and manual commands.
**Solution:** One-click rollback workflow in GitHub Actions
triggerable from the Actions tab without needing SSH access.

---

## Data retention

| Data type | Retention period | Location |
|---|---|---|
| Prometheus metrics | 15 days | `prometheus_data` volume |
| Loki logs | 15 days | `loki_data` volume |
| Tempo traces | 1 hour (test) / 7 days (production) | `tempo_data` volume |

---

## Repository structure
hng-stage6-observability/
├── docker-compose.yml
├── .env                          # Not committed — holds secrets
├── .gitignore
├── README.md
├── prometheus/
│   └── prometheus.yml
├── loki/
│   └── loki-config.yml
├── tempo/
│   └── tempo-config.yml
├── alertmanager/
│   └── alertmanager.yml
├── otel/
│   └── otel-config.yml
├── rules/
│   └── alerts.yml
├── grafana/
│   ├── dashboards/
│   │   ├── node-exporter.json
│   │   ├── blackbox.json
│   │   ├── slo-error-budget.json
│   │   ├── dora-metrics.json
│   │   └── unified-observability.json
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yml
│       └── dashboards/
│           └── dashboards.yml
└── runbooks/
├── server-down.md
├── cpu-high.md
├── memory-high.md
├── disk-high.md
├── slo-burn-rate.md
├── high-cfr.md
└── slo-breach.md

---

## Stopping the stack

```bash
# Stop all services
docker compose down

# Stop and remove all data volumes (full reset)
docker compose down -v
```

---

*HNG DevOps Track — Stage 6 | Dreybest*