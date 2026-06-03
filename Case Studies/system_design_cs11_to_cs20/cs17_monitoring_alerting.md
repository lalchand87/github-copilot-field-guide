# Case Study 17 — Monitoring and Alerting System

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Time-Series DB selection, Alert evaluation, Cardinality problem.

---

## Business Context

Monitoring is infrastructure that tells you when something is wrong before your customers do. For a payment platform, the consequences of delayed alerting are direct: every minute of undetected payment API failure costs revenue and damages trust. The monitoring system is the last line of defence between a silent failure and a customer-reported outage.

The system itself must be highly available — if monitoring goes down, you're flying blind. This creates an ironic challenge: the monitoring system needs to be monitored.

**Business goals driving architecture:**
- Alert within 60 seconds of anomaly onset
- Zero missed alerts: a firing alert must reach the on-call engineer even if systems are partially degraded
- Historical metrics queryable for 13 months (trend analysis, capacity planning, YoY comparison)
- Dashboard render < 2s (engineers on incident should not wait for dashboards)
- Cardinality management: high-cardinality labels (user_id, request_id) destroy TSDB performance

---

## Recognition Framework

### Signals from the Problem
```
- Write-heavy, read-light: metrics written every 15s per time series; queried infrequently
- Time-ordered data: queries always have time range filter; no random access
- High write throughput: 10K services × 100 metrics each = 1M metric series at 15s resolution
- Aggregation at query time: AVG(latency) over time range; SUM(errors) per service
- Alert evaluation: periodic rule evaluation (every 30s); not ad-hoc queries
- Cardinality: each unique label combination = new series; explosion risk with bad labels
- Long retention: 15s raw for 30 days; 1m aggregated for 13 months
```

### Time-Series DB Comparison

```
Option A: Prometheus (pull-based)
  Model: Prometheus scrapes targets every 15s; targets expose /metrics endpoint
  Storage: local TSDB (not distributed); single-machine storage limit
  Query: PromQL (powerful; time-series native)
  HA: not built-in; requires Thanos or Cortex for HA + long-term storage
  Best for: Kubernetes-native; small to medium infrastructure

Option B: InfluxDB
  Model: push-based; services write to InfluxDB
  Storage: distributed; column-store optimized for time series
  Query: Flux (InfluxDB 2.x) or InfluxQL
  Best for: standalone TSDB; IoT; non-Kubernetes

Option C: VictoriaMetrics
  Model: compatible with Prometheus remote_write; receives pushed metrics
  Storage: highly compressed; very fast writes
  Query: MetricsQL (PromQL-compatible)
  Best for: large-scale Prometheus replacement; better performance at high cardinality

Option D: Amazon Timestream / Google Cloud Monitoring
  Managed; no ops burden; auto-scaling
  Best for: cloud-native; don't want to manage TSDB infrastructure

CHOSEN for this design:
  Prometheus (collection) + Thanos (HA + long-term storage on S3)
  Prometheus: every Kubernetes cluster has a Prometheus instance (standard)
  Thanos: global view across clusters; S3 for long-term retention (13 months)
  Alert manager: Prometheus Alertmanager for alert routing and dedup
```

### Cardinality Problem — Critical Concept

```
Metric: http_requests_total{service="payment", status_code="200", path="/api/v1/payments"}
  Labels: service × status_code × path combinations
  5 services × 5 status codes × 100 paths = 2,500 unique time series
  → Manageable

BAD: http_requests_total{service="payment", user_id="u-12345", request_id="req-abc"}
  Labels: user_id is HIGH CARDINALITY (1M unique values)
  1M users × any other labels = millions of unique time series
  → TSDB memory explosion; query performance degrades to minutes
  → Cardinality error: Prometheus may OOM and crash

RULE: Never use high-cardinality values as labels
  Bad labels (high cardinality): user_id, payment_id, request_id, session_id, IP address
  Good labels (low cardinality): service_name, endpoint_path, status_code, region, tier

Cardinality limit: Prometheus recommends < 10K unique label value combinations per metric
At 10K+ combinations: query becomes a full table scan → O(n) performance degradation

Fix for high-cardinality use cases:
  Need per-user metrics? → Don't use metrics; use tracing (Jaeger) + log aggregation
  Need per-request error tracking? → Structured logs + log aggregation (not metrics)
  Metrics = aggregate statistics; Traces = per-request details; Logs = event records
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Time-Series DB (Prometheus) | Native for time-ordered metric data; PromQL for alerting |
| Pull-based collection | Scraping is self-discovering; no push config needed per service |
| Thanos (global view + S3) | HA across clusters; 13-month retention; dedup |
| Alertmanager | Route, group, deduplicate, and silence alerts |
| Alert evaluation | Periodic PromQL evaluation; not real-time streaming |
| PagerDuty/Opsgenie integration | Alert → on-call notification via escalation policy |

---

## Problem Statement

Monitoring and alerting system for a payment platform. Collect metrics from all services, store with 13-month retention, evaluate alert rules, and page on-call engineers within 60 seconds of anomaly onset.

---

## Functional Requirements
- Collect metrics from all services (push and pull models)
- Dashboards: time-series charts, heatmaps, histograms
- Alert rules: threshold, rate-of-change, anomaly detection
- Alert routing: team-based routing; escalation; grouping
- Silences: suppress known false-positive alerts during maintenance
- Runbooks: link alert → runbook (actionable response)
- Multi-environment: prod, staging, dev in same system (with segregation)

## Non-Functional Requirements
- Metric ingestion: **1M metrics/sec** (10K services × 100 metrics × 1/15s scrape interval)
- Alert evaluation: **every 30 seconds**
- Alert delivery: **< 60 seconds** from anomaly onset to on-call notification
- Dashboard load: **< 2 seconds**
- Retention: 15s raw for 30 days; 1-minute aggregated for 13 months

---

## Capacity Estimation
```
Metrics:
  10K services × 100 metric series each = 1M active metric series
  Scrape interval: 15 seconds
  Write throughput: 1M / 15 = 66K writes/sec
  
Storage:
  Each data point: 16 bytes (timestamp + value, after Prometheus compression)
  But Prometheus achieves ~1.3 bytes/sample with delta-of-delta encoding
  1M series × 4 samples/min × 1.3 bytes = ~5.2 MB/min = ~312 MB/hour
  30-day raw: 312 MB/hr × 720 = ~225 GB per Prometheus instance
  13-month in S3 (1-min aggregated): much smaller → ~100 GB total
  
Thanos:
  Each Prometheus instance uploads 2-hour blocks to S3
  S3 storage: 225 GB × 13 months + Thanos overhead ≈ ~3 TB
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Prometheus | Pull-based metric collection; 30-day local storage |
| Thanos | Global view; deduplication; S3 long-term storage |
| Alertmanager | Alert routing, dedup, silencing, grouping |
| PromQL | Query language for alerts and dashboards |
| Recording Rules | Pre-compute expensive queries; faster dashboards |
| PagerDuty / Opsgenie | On-call notification; escalation policies |

---

## Data Model

```yaml
# Prometheus metric format (Prometheus exposition format):
# Each service exposes /metrics endpoint:
# payment_requests_total{service="payment-service",status="200",endpoint="/pay"} 12345
# payment_latency_p99{service="payment-service"} 0.187
# payment_errors_total{service="payment-service",error_type="timeout"} 42

# Alert rule (in Prometheus alerting rules YAML):
groups:
  - name: payment_service_alerts
    rules:
      # Alert 1: High error rate
      - alert: PaymentErrorRateHigh
        expr: |
          (
            rate(payment_errors_total[5m])
            /
            rate(payment_requests_total[5m])
          ) > 0.01
        for: 2m          # must be true for 2 consecutive minutes before firing
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "Payment error rate > 1% for {{ $labels.service }}"
          description: "Error rate: {{ $value | humanizePercentage }}"
          runbook_url: "https://runbooks.internal/payment-error-rate"

      # Alert 2: P99 latency degradation
      - alert: PaymentLatencyDegraded
        expr: histogram_quantile(0.99, rate(payment_latency_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
          team: payments
        annotations:
          summary: "Payment P99 latency > 500ms"

      # Alert 3: Dead man's switch (absence of metric = problem)
      - alert: PaymentServiceDown
        expr: absent(payment_requests_total)  # no metric = service not scraping
        for: 1m
        labels:
          severity: critical

      # Recording rule (pre-compute for dashboard performance):
      - record: payment:request_rate5m
        expr: rate(payment_requests_total[5m])
```

```yaml
# Alertmanager routing config:
route:
  group_by: [alertname, service]
  group_wait: 30s        # wait before sending first alert (group related alerts)
  group_interval: 5m     # time between grouped alert updates
  repeat_interval: 4h    # re-notify if alert keeps firing

  routes:
    - match:
        team: payments
        severity: critical
      receiver: payments-pagerduty  # immediate PagerDuty call
      continue: true

    - match:
        team: payments
        severity: warning
      receiver: payments-slack     # Slack message only

receivers:
  - name: payments-pagerduty
    pagerduty_configs:
      - routing_key: <PD_integration_key>
        description: "{{ .Alerts.Firing | len }} alerts firing: {{ .GroupLabels }}"

  - name: payments-slack
    slack_configs:
      - api_url: <Slack_webhook>
        channel: "#alerts-payments"
        title: "{{ .Alerts.Firing | len }} alerts: {{ .GroupLabels.alertname }}"
```

---

## High-Level Architecture

```
Services (payment-service, auth-service, etc.)
  │
  ├── Option A: Pull (Prometheus scrapes)
  │     Service exposes GET /metrics (Prometheus exposition format)
  │     Prometheus discovers targets via Kubernetes service discovery
  │     Scrape every 15 seconds
  │
  └── Option B: Push (for short-lived jobs, batch processing)
        Service → Pushgateway → Prometheus scrapes Pushgateway

Prometheus Instance per Cluster:
  ├── Scrapes all targets in cluster every 15s
  ├── Evaluates alert rules every 30s
  ├── Sends firing alerts to Alertmanager
  ├── Local TSDB: 30-day raw storage
  └── Remote write to Thanos Receive OR Prometheus Thanos Sidecar uploads to S3

Thanos (global view + long-term):
  ├── Thanos Query: federated query across all Prometheus instances
  │     Query: "show me payment error rate across all clusters for last 7 days"
  │     Deduplication: removes duplicate series (same metric from HA Prometheus pair)
  ├── Thanos Store: serves queries from S3 (historical data > 30 days)
  └── Thanos Compact: downsamples old data (5m resolution after 30 days; 1h after 1 year)
      Reduces S3 storage and query time for long-range queries

Alertmanager (HA pair):
  ├── Receives alerts from all Prometheus instances
  ├── Groups, deduplicates, silences
  ├── Routes to PagerDuty / Slack / OpsGenie
  └── Gossip protocol between HA pair: consistent state; no double-notification

Grafana (dashboards):
  ├── Query: Thanos Query as data source (all historical data)
  ├── Recording rules: pre-computed expensive queries (dashboard loads fast)
  └── Alerting: Grafana Alerting (in addition to Prometheus) for dashboard-based alerts

On-Call Routing:
  PagerDuty / OpsGenie:
    Escalation policy: alert → primary on-call → 5m escalation → secondary → team lead
    Oncall schedule: weekly rotation; 12-hour shifts
    Runbook link: every alert links to actionable runbook
```

---

## Detailed Components

### PromQL Alert Expression Patterns
```promql
# Pattern 1: Rate-based error threshold
# "Error rate > 1% over 5 minutes for 2 minutes"
(
  rate(http_requests_total{status=~"5.."}[5m])
  /
  rate(http_requests_total[5m])
) > 0.01

# Pattern 2: Absent metric (dead man's switch)
# "If this metric disappears, something is wrong"
absent(up{job="payment-service"})

# Pattern 3: Prediction-based alert
# "If disk usage grows at current rate, it will fill in < 4 hours"
predict_linear(node_filesystem_avail_bytes[1h], 4 * 3600) < 0

# Pattern 4: Sudden change (not just threshold)
# "Error count increased 10× vs 5 minutes ago"
rate(payment_errors_total[5m]) > 10 * rate(payment_errors_total[5m] offset 5m)

# Pattern 5: Percentile latency
# "P99 latency > 500ms"
histogram_quantile(0.99, sum(rate(http_request_duration_bucket[5m])) by (le)) > 0.5

# Anti-pattern (cardinality explosion):
# http_requests_total{user_id="u-12345"}  ← NEVER DO THIS
# 1M users = 1M unique series = TSDB OOM
```

### Recording Rules for Dashboard Performance
```yaml
# Problem: "P99 latency per service for last 7 days" at query time
# → histogram_quantile over 7 days of raw data = slow (millions of data points)

# Solution: recording rules pre-compute and store the result
groups:
  - name: performance_recording_rules
    interval: 30s  # recompute every 30 seconds
    rules:
      # Pre-compute 5-minute P99 per service
      - record: service:http_latency_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum by (service, le) (
              rate(http_request_duration_bucket[5m])
            )
          )

      # Pre-compute request rate per service
      - record: service:request_rate:5m
        expr: |
          sum by (service) (
            rate(http_requests_total[5m])
          )

# Dashboard query: service:request_rate:5m
# → reads pre-computed result (1 series per service)
# → vs raw computation (N buckets × M services)
# → 100× faster for dashboard rendering
```

### Alert Fatigue Prevention
```
Problem: too many alerts → on-call engineers ignore them ("alert fatigue")

Solutions:

1. Grouping (Alertmanager):
   100 microservices all firing "high latency" simultaneously → 1 grouped alert
   "100 services have high latency" is actionable; 100 separate pages are not

2. `for` duration (Prometheus alert rules):
   Alert must be true for N minutes before firing
   `for: 5m` → transient spike → no alert
   `for: 2m` → persistent problem → fire
   Balance: shorter = more alerts; longer = more delay

3. Symptom-based alerting (not cause-based):
   Good: "payment error rate > 1%" (symptom; directly customer-impacting)
   Bad:  "CPU usage > 80%" (cause; may not impact customers)
   Rule: alert on what users experience; not on internal metrics

4. Alert severity tiers:
   Critical: page on-call immediately (PagerDuty, wakes people up at 3am)
   Warning: Slack message (addressed during business hours)
   Info: logged; visible on dashboard; no notification
   Most alerts should be Warning or Info, not Critical

5. Runbooks per alert:
   Every alert must have a runbook: "when you see this alert, do X then Y then Z"
   Engineers follow procedure rather than improvising during incident

6. Dead man's switch:
   Special pattern: "alert if I stop receiving this metric"
   Used to detect: monitoring system failure, scrape target down, agent crash
   opposite of normal alerting (fire on absence, not presence)
```

---

## Scaling Strategy
```
Phase 1 (0 → 100 services):
  Single Prometheus instance + Grafana
  Local storage (30 days)
  Alertmanager single instance
  Manual alert routing to Slack

Phase 2 (100 → 1K services):
  Prometheus HA pair per cluster (identical config; dedup in Alertmanager)
  Alertmanager HA pair (gossip protocol for consistent state)
  PagerDuty integration for on-call rotation

Phase 3 (1K → 10K services):
  Thanos for global view and S3 long-term storage
  Recording rules for all dashboard queries
  Cardinality audit: identify and remove high-cardinality labels
  Remote write to Thanos Receive for immediate global visibility

Phase 4 (multi-region global):
  Prometheus per region; Thanos globally aggregates
  Global dashboards via Thanos Query with cross-region dedup
  Regional alerting (each region pages its own team first)
```

---

## Reliability Strategy
- **Prometheus HA pair**: two identical instances per cluster; Alertmanager deduplicates
- **Alertmanager HA pair**: gossip protocol prevents double-paging on failover
- **Dead man's switch**: AlertManager sends a "watchdog" alert to external system every 5 minutes; if absent for 10 minutes → external system pages on-call ("your monitoring is broken")
- **S3 durability for Thanos**: historical metrics in S3 (11 nines durability)
- **Alertmanager network resilience**: PagerDuty webhooks have retry logic; alert not lost if HTTP call fails transiently

---

## Security Considerations
- **Metric exposure**: /metrics endpoint must be internal-only (not exposed to internet); scrape via VPC
- **Grafana auth**: SSO (OIDC/SAML) integration; team-based dashboard access (engineers see only their service metrics)
- **Alert secrets**: PagerDuty/Slack webhook URLs in secrets manager (not config files)
- **Cardinality-based DoS**: a service that exports 1M label combinations can crash Prometheus; cardinality limits enforced by admission webhook or scrape limit config
- **Audit**: all Grafana dashboard accesses and alert rule changes logged (Case 10 audit service)

---

## Observability (Meta-monitoring)
```
The monitoring system monitors itself:

Prometheus monitors:
  prometheus_tsdb_head_series                 (active series count; alert if > 2M)
  prometheus_rule_evaluation_duration_seconds (alert evaluation latency)
  alertmanager_alerts_firing_total            (currently firing alerts)
  prometheus_target_scrape_duration_seconds   (slow scrapes indicate target issues)

Thanos monitors:
  thanos_compact_group_compactions_total
  thanos_store_series_blocks_queried          (S3 query activity)
  thanos_receive_replication_factor           (replication health)

Dead man's switch:
  AlertManager sends a test alert "WatchdogAlive" to PagerDuty every 5 minutes
  PagerDuty: if WatchdogAlive absent for > 10 minutes → page on-call immediately
  "Your monitoring system is broken; you're flying blind"
```

---

## Chaos Testing
```
Experiment 1: Prometheus Instance Crash
  Inject:    Kill one Prometheus in HA pair
  Expected:  Alertmanager deduplicates; only one copy of each alert fires
  Expected:  Thanos Query fails over to remaining Prometheus instance
  Expected:  No alert loss; no double-paging
  Red flag:  Same alert pages twice (Alertmanager not deduplicating)

Experiment 2: High-Cardinality Metric Injection
  Inject:    Service starts exporting metric with user_id label (1M unique values)
  Expected:  Prometheus scrape limit rejects metric (cardinality guard)
  Expected:  Cardinality alert fires: "service X is exporting high-cardinality metric"
  Expected:  Prometheus does not OOM
  Red flag:  Prometheus memory usage grows unboundedly until OOM

Experiment 3: Alert Delivery Under Partial Outage
  Inject:    PagerDuty API returns 503; simultaneously create critical alert condition
  Expected:  Alertmanager retries PagerDuty webhook; eventually delivers
  Expected:  Slack notification also sent (secondary receiver)
  Expected:  On-call engineer gets paged (eventually, even if delayed)
  Red flag:  Alert lost entirely because PagerDuty was temporarily unavailable

Experiment 4: Dead Man's Switch Test
  Inject:    Stop Alertmanager entirely
  Expected:  WatchdogAlive heartbeat stops reaching PagerDuty
  Expected:  After 10 minutes: PagerDuty fires "monitoring is broken" alert
  Expected:  On-call paged without Alertmanager involvement (external monitoring)
  Red flag:  No alert when monitoring system fails (blind spot)
```

---

## Monthly Cost Estimate
```
Scale: 10K services, 1M metric series

Prometheus HA pairs (2 per Kubernetes cluster; 10 clusters):
  20 × m5.2xlarge ($280/mo)                          = $5,600/mo

Thanos (Query, Store, Compact):
  5 × m5.xlarge ($150/mo)                            = $750/mo

S3 (long-term metric storage):
  ~3 TB × $0.023/GB                                  = $69/mo

Alertmanager HA pair:
  2 × t3.medium ($34/mo)                             = $68/mo

Grafana (2 instances for HA):
  2 × t3.large ($60/mo)                              = $120/mo

PagerDuty / OpsGenie:
  Team plan: ~$600/month for 20 users

Monitoring/networking:
                                                      = $200/mo
─────────────────────────────────────────────────────────────────
Total: ~$7,407/month
(Most cost is Prometheus compute; S3 is negligible for long-term retention)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Prometheus local TSDB: authoritative for recent 30 days.
  → Thanos + S3: authoritative for historical (> 30 days).
  → No "master" truth store; each Prometheus is authoritative for its scrape targets.
  → Thanos deduplicates when HA pair has identical data.

□ Consistency model?
  → Prometheus scrapes are periodic (15s); metrics are always slightly stale.
  → Alert evaluation runs 30s after metric scrape → alert delay up to 45s.
  → Thanos Query across clusters: all data from different Prometheus instances merged.
  → Eventual consistency acceptable for monitoring (dashboards show recent data).

□ Failure mode?
  → Prometheus crash: HA pair continues; Thanos serves historical.
  → Alertmanager crash: HA pair continues; gossip resync on recovery.
  → S3 unavailable: Thanos Store unavailable; Prometheus serves recent 30 days.
  → Dead man's switch: Alertmanager failure detected by external watchdog.

□ Cardinality is the #1 operational risk:
  → Every new service must review metric labels before deployment
  → Cardinality guard: Prometheus scrape limit rejects > 10K series per target
  → High-cardinality signals belong in tracing/logging (not metrics)

□ SCC LENS:
  STATE: Prometheus TSDB (recent), S3/Thanos (historical), Alertmanager (firing alerts)
  COORDINATION: HA pairs with deduplication; dead man's switch for self-monitoring
  CONCENTRATION: popular dashboards → recording rules pre-compute; no live aggregation
```

---

## Staff Engineer Discussion Points

**"Why pull-based (Prometheus scrapes) instead of push-based (services push metrics)?"**
Pull-based has key advantages: (1) Prometheus controls the scrape schedule — no thundering herd if all services push simultaneously; (2) scrape failure is immediately visible (target shows as "down" in Prometheus); (3) auto-discovery via Kubernetes service discovery — no config per service; (4) short-lived jobs (batches) are the only exception: they push to Pushgateway. Push-based is better for short-lived processes that disappear before being scraped.

**"How do you avoid alert fatigue?"**
Four levers: (1) Alert on symptoms, not causes — "payment error rate > 1%" not "CPU > 80%"; (2) `for` duration filters transient spikes (genuine problems persist); (3) Alertmanager grouping — 100 services all hitting same issue = 1 grouped alert, not 100 pages; (4) Severity tiers — Critical wakes people up; Warning is Slack only; most alerts should be Warning. If on-call is seeing > 5 alerts per shift, something is wrong with your alerting discipline.

---

## Cheat Sheet Tie-in

```
CARDINALITY RULE (memorize this):
  Never use high-cardinality values as metric labels (user_id, request_id, IP)
  High cardinality → TSDB OOM → Prometheus crashes
  Use tracing for per-request details; metrics for aggregate statistics

TSDB SELECTION:
  Time-series DB is the right choice because:
    - Queries always have time range filter
    - Data is append-only (never updated)
    - Compression: delta-of-delta encoding → 1.3 bytes/sample (vs 16 bytes raw)
    - Aggregation at query time: average/sum over time range

PROMETHEUS + THANOS PATTERN:
  Prometheus: local scrape → 30-day TSDB → alert evaluation
  Thanos: global view → dedup across HA pairs → S3 long-term
  Alertmanager: route, group, dedup, silence → PagerDuty/Slack

ALERT QUALITY:
  Good alert: symptom-based, has runbook, pages only on persistent condition (for: 2m)
  Bad alert: cause-based, no runbook, pages on transient spike
  Dead man's switch: alert on ABSENCE of metric (inverse of normal alerting)

SCC LENS:
  STATE: Prometheus TSDB (recent hot), S3 (cold historical), Alertmanager (alert state)
  COORDINATION: HA dedup in Alertmanager; dead man's switch for self-monitoring
  CONCENTRATION: high-cardinality labels cause concentration in single TSDB hot path

IAM PLATFORM CONNECTION:
  Auth events → metrics: auth_requests_total{type="login", outcome="success/failure"}
  MFA adoption rate: mfa_enrolled_users / total_users (business metric in Grafana)
  OAuth token issuance rate: alert if drops suddenly (auth service down)
  Failed auth spike: alert → potential credential stuffing attack → feeds into fraud detection
```

---

*Next: Case Study 18 — Analytics Pipeline*
