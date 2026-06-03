# Case Study 29 — Observability Platform (OpenTelemetry, Distributed Tracing)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Three Pillars, Trace Propagation, Sampling Strategy, Exemplars.

---

## Business Context

Monitoring (Case 17) tells you something is wrong. Observability tells you *why*. When a payment is slow, you need to know: was it the fraud check, the card network call, the DB query, or the webhook delivery? A distributed trace answers this by stitching together all the spans across all services into a single timeline.

For a payment platform with 20+ microservices, a single payment touches fraud-service, ledger-service, card-network-adapter, notification-service, and the API gateway. Without distributed tracing, debugging latency is guesswork. With it, you see exactly which service added how many milliseconds.

**Business goals driving architecture:**
- Root cause analysis in minutes, not hours, during incidents
- SLO tracking: which services are breaching their latency/error SLOs?
- Capacity planning: which services are bottlenecks under load?
- Compliance: complete audit of which service accessed which data (for GDPR/PCI)
- Developer productivity: find the slow DB query without reading 20 service logs

---

## Recognition Framework

### Three Pillars of Observability

```
Pillar 1: Metrics (Case 17)
  What: numeric measurements over time (request rate, latency, error count)
  Query: "What is the P99 latency of payment-service over the last hour?"
  Tool: Prometheus + Thanos
  Cost: cheap (aggregated numbers; not per-request)
  When: monitoring; alerting; dashboards; capacity planning

Pillar 2: Logs
  What: structured event records from services
  Query: "Show me all ERROR logs from payment-service in the last 10 minutes"
  Tool: Loki, Elasticsearch, CloudWatch
  Cost: expensive at scale (every request generates logs)
  When: debugging a known issue; security audit; error investigation

Pillar 3: Traces
  What: end-to-end journey of a single request across multiple services
  Query: "Show me the full trace for payment pay-abc-123"
  Tool: Jaeger, Tempo, Zipkin
  Cost: variable (depends on sampling rate; full trace can be large)
  When: latency investigation; dependency mapping; performance optimisation

The three pillars are complementary:
  Metric alert fires → navigate to dashboard → find slow service
  Look at traces for that service → find which operation is slow
  Look at logs for that operation → find the root cause
  
  Exemplars: link metrics to traces (a specific P99-violating metric sample → its trace_id)
```

### OpenTelemetry — The Unified Standard

```
Before OpenTelemetry:
  Each vendor had its own SDK: Jaeger SDK, Zipkin SDK, Datadog SDK
  Changing vendors: rewrite all instrumentation code
  Three separate sets of SDKs for metrics, logs, and traces
  Result: vendor lock-in; high switching cost

OpenTelemetry (OTEL):
  Vendor-neutral standard for collecting traces, metrics, and logs
  Single SDK per language (Java, Python, Go, etc.) for all three signals
  Exporters: send to Jaeger, Tempo, Prometheus, Datadog — just change exporter
  Auto-instrumentation: Spring Boot, Django, etc. auto-instrumented with zero code changes
  
  Components:
    OTEL SDK: in-process; collects telemetry from your code
    OTEL Collector: sidecar or standalone; receives, processes, and exports telemetry
    Exporter: sends to backend (Jaeger, Tempo, Prometheus, Elasticsearch)
  
  Key concept — Context Propagation:
    Trace ID must flow across service boundaries (HTTP, Kafka, gRPC)
    W3C TraceContext header: traceparent: 00-{trace_id}-{span_id}-{flags}
    Every service: extract trace context from incoming request; inject into outgoing
    Result: all spans from all services share the same trace_id → single trace view
```

---

## Problem Statement

Observability platform for a distributed payment system. Distributed tracing with OpenTelemetry. Logs correlated by trace_id. Metrics with exemplars (link to trace). SLO tracking. Sampling strategy to manage cost.

---

## Functional Requirements
- Distributed traces: end-to-end request journey across all services
- Trace propagation: trace context flows through HTTP, Kafka, gRPC calls
- Log correlation: logs tagged with trace_id + span_id (find logs for a specific trace)
- Metrics with exemplars: high-latency metric samples link to their trace
- SLO tracking: define SLOs; burn rate alerts when error budget consumed too fast
- Sampling: head-based and tail-based sampling to control cost

## Non-Functional Requirements
- Trace ingestion: **1M traces/sec** (one trace per payment; 50K TPS × 20 spans avg)
- Trace storage: **48 hours hot** (for active incident investigation); 30 days cold
- Query latency: **< 2 seconds** for trace lookup by trace_id or service/operation
- Sampling overhead: **< 1% CPU and memory** impact on instrumented services

---

## Capacity Estimation
```
Traces:
  50K payments/sec × 1 trace each = 50K traces/sec
  Each trace: 20 spans avg (20 service calls per payment)
  Each span: ~500 bytes (operation, timestamps, tags, logs)
  50K traces × 20 spans × 500 bytes = 500 MB/sec raw telemetry

With sampling (10% sample rate):
  500 MB/sec × 10% = 50 MB/sec stored
  48h hot storage: 50 MB/sec × 172800 = ~8.6 TB
  30d cold storage (1% sampling): ~2.6 TB

OTEL Collector throughput:
  One collector per host: handles local service telemetry
  Central collector cluster: aggregates, samples, and exports
  10 central collectors × 50 MB/sec = 500 MB/sec capacity (matches peak)

Logs:
  50K TPS × 5 log lines avg × 200 bytes = 50 MB/sec
  With 30d retention: ~130 TB
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| OpenTelemetry SDK | Vendor-neutral; auto-instruments Spring Boot; single SDK for all signals |
| OTEL Collector (Agent + Gateway) | Local agent per host; central gateway for batching + sampling |
| Jaeger / Grafana Tempo | Distributed trace storage and query backend |
| Tail-Based Sampling | Sample ALL slow/errored traces; sample only 1% of fast/successful |
| Exemplars | Link metrics (P99 latency) to a specific trace_id for that sample |
| Trace-Log Correlation | Every log line includes trace_id + span_id → cross-signal navigation |
| SLO Burn Rate Alerts | Alert when error budget consumed 2× faster than expected |

---

## Data Model

```
Trace structure:
  Trace:
    trace_id: "abc123" (16 bytes; same for all spans in one request)
    spans: [Span, ...]
  
  Span:
    trace_id: "abc123"
    span_id: "span01"        (unique per span)
    parent_span_id: "root"   (null for root span; parent's span_id for children)
    service_name: "payment-service"
    operation_name: "POST /api/v1/payments"
    start_time: epoch_ns
    end_time: epoch_ns
    duration_ms: 234
    status: OK / ERROR
    attributes: {
      "http.method": "POST",
      "http.url": "/api/v1/payments",
      "payment.id": "pay-abc",
      "payment.amount": 50000,
      "db.statement": "INSERT INTO payments ...",  (if DB span)
      "error.message": "Connection timeout"        (if error)
    }
    events: [                    (span events = log-like entries within a span)
      { time: epoch_ns, name: "fraud_check_started" },
      { time: epoch_ns, name: "fraud_check_completed", attributes: { score: 0.12 } }
    ]

Trace context propagation headers (W3C standard):
  traceparent: 00-{trace_id_hex}-{span_id_hex}-01
  tracestate: vendor-specific (ignored in most setups)
  
  HTTP: injected in request headers
  Kafka: injected in message headers
  gRPC: injected in metadata
```

```yaml
# OTEL Collector config (central gateway)
receivers:
  otlp:
    protocols:
      grpc:           # receive from service agents
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:              # batch for efficiency
    timeout: 5s
    send_batch_size: 1000
  
  tail_sampling:      # key: decide AFTER seeing full trace
    decision_wait: 10s  # wait 10s for all spans of trace to arrive
    policies:
      - name: error-policy
        type: status_code
        status_code: { status_codes: [ERROR] }  # sample ALL errors (100%)
      - name: slow-policy
        type: latency
        latency: { threshold_ms: 500 }         # sample ALL traces > 500ms
      - name: sample-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }  # 1% of all others

exporters:
  jaeger:
    endpoint: jaeger-collector:14250
  prometheus:          # for trace-derived metrics
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, tail_sampling]
      exporters: [jaeger]
```

---

## High-Level Architecture

```
Services (payment-service, fraud-service, etc.)
  │
  ├── OTEL SDK (auto-instrumented; Spring Boot OpenTelemetry starter)
  │     On every request:
  │       - Create root span (API Gateway) or child span (internal service)
  │       - Extract traceparent from incoming request headers
  │       - Inject traceparent into outgoing requests (HTTP/Kafka/gRPC)
  │       - Collect: operation name, duration, HTTP status, error info, custom attributes
  │       - Collect: log lines tagged with trace_id + span_id
  │     
  ├── OTEL Agent (sidecar per pod)
  │     Receives telemetry from service SDK via localhost gRPC
  │     Adds resource attributes: k8s.pod.name, k8s.namespace, service.version
  │     Forwards to central OTEL Collector gateway
  │
  └── OTEL Collector Gateway (central; 10 instances behind LB)
        Receives from all agents
        Tail-based sampling: buffer 10s of spans; decide per trace
          ERROR traces: keep 100%
          SLOW traces (> 500ms): keep 100%
          Normal traces: keep 1%
        Export:
          Traces → Jaeger (or Grafana Tempo)
          Metrics → Prometheus (via remote_write)
          Logs → Loki (or Elasticsearch)

Trace Storage (Jaeger or Grafana Tempo):
  Jaeger all-in-one: good for < 5 TB; simple
  Grafana Tempo: S3-backed; scales to petabytes; better for long retention
  
  Tempo storage:
    Trace data: S3 (cheap; scalable)
    Trace index (for search by service/operation/tag): local disk or Cassandra
    Block compaction: background process merges and compresses old blocks

Log Storage (Loki):
  Loki: "like Prometheus but for logs" — label-based; cheap S3 storage
  Log line: { trace_id, span_id, service_name, level, message, attributes }
  Query: {service="payment-service"} |= "pay-abc" → logs for specific payment
  Cross-signal: click trace_id in Jaeger → jump to Loki logs for that trace

Dashboarding (Grafana):
  Service Map: auto-generated from trace data (which service calls which)
  SLO Dashboard: error budget consumption; burn rate
  Exemplar view: scatter plot of request latency; click outlier → open trace
```

---

## Detailed Components

### Trace Propagation in Practice
```java
// Spring Boot with OTEL auto-instrumentation: zero code changes for HTTP
// Incoming request: traceparent header automatically extracted
// Outgoing HTTP calls via RestTemplate: traceparent header automatically injected

// Manual instrumentation for custom operations:
@Autowired
private Tracer tracer;  // OTEL Tracer bean

public FraudDecision evaluateFraud(Payment payment) {
    // Create a child span for this operation
    Span span = tracer.spanBuilder("fraud.evaluate")
        .setAttribute("payment.id", payment.getId().toString())
        .setAttribute("payment.amount", payment.getAmount())
        .startSpan();

    try (Scope scope = span.makeCurrent()) {
        // All log lines within this scope automatically tagged with trace_id + span_id
        log.info("Starting fraud evaluation for payment {}", payment.getId());

        FraudDecision decision = fraudEngine.evaluate(payment);

        span.setAttribute("fraud.score", decision.getScore());
        span.setAttribute("fraud.decision", decision.getDecision().toString());

        if (decision.isBlocked()) {
            span.setStatus(StatusCode.ERROR, "Payment blocked by fraud engine");
            span.setAttribute("fraud.block_reason", decision.getReasonCode());
        }

        return decision;

    } catch (Exception e) {
        span.recordException(e);
        span.setStatus(StatusCode.ERROR, e.getMessage());
        throw e;
    } finally {
        span.end();
    }
}
```

### Kafka Trace Propagation
```java
// Kafka producer: inject trace context into message headers
public void publishPaymentEvent(PaymentEvent event) {
    ProducerRecord<String, PaymentEvent> record =
        new ProducerRecord<>("payment.events", event.getPaymentId(), event);

    // OTEL Kafka instrumentation handles this automatically in Spring Kafka
    // But manual approach:
    Context currentContext = Context.current();
    W3CTraceContextPropagator.getInstance().inject(
        currentContext,
        record.headers(),
        (headers, key, value) -> headers.add(key, value.getBytes())
    );

    kafkaProducer.send(record);
}

// Kafka consumer: extract trace context and create child span
@KafkaListener(topics = "payment.events")
public void handlePaymentEvent(ConsumerRecord<String, PaymentEvent> record) {
    // Extract trace context from Kafka headers
    Context extractedContext = W3CTraceContextPropagator.getInstance().extract(
        Context.root(),
        record.headers(),
        (headers, key) -> {
            Header header = headers.lastHeader(key);
            return header != null ? new String(header.value()) : null;
        }
    );

    // Create child span linked to the original trace
    Span span = tracer.spanBuilder("payment.event.process")
        .setParent(extractedContext)
        .startSpan();

    // Now this Kafka consumer's spans are linked to the original payment trace
    // → Single trace spans API gateway → payment-service → Kafka → analytics-service
}
```

### Tail-Based Sampling — Why It Matters
```
Head-based sampling (decision made at trace start):
  Sampler: "1% of all requests"
  Problem: the 1 slow request in 1000 might NOT be in the 1%
  You miss the important traces (the errors, the slow ones)
  You keep 99% of boring traces (normal, fast, no errors)

Tail-based sampling (decision made after trace complete):
  Buffer all spans for 10 seconds
  After 10s, see the full trace: duration, error status, interesting attributes
  Decision: keep if ERROR (100%) or SLOW > 500ms (100%) or random 1% of rest
  
  Result:
    100% of error traces → you never miss a payment failure
    100% of slow traces → you never miss a P99 violation
    1% of normal traces → you still have baseline for comparison
  
  Trade-off:
    Requires buffering 10s of spans → higher memory in OTEL Collector
    Decision latency: 10s before sending to Jaeger
    Acceptable: traces are for debugging; 10s delay doesn't matter

Head-based is simpler but tail-based is more valuable for debugging.
Most modern observability platforms (Grafana Tempo, Honeycomb) support tail-based.
```

### SLO Tracking and Error Budget
```
SLO (Service Level Objective):
  "Payment API latency P99 < 2 seconds for 99.9% of requests over 30 days"
  
  Error budget = 100% - 99.9% = 0.1% of requests can violate the SLO
  Over 30 days: 50K TPS × 86400 × 30 = 129.6B requests × 0.1% = 129.6M budget requests
  
  Burn rate: if you're burning budget at 2× the normal rate, you'll exhaust it in 15 days
  Burn rate alert: fire when burn rate > 2× sustained for > 1 hour

Prometheus recording rules for SLOs:
  # Count requests that violated SLO (latency > 2s)
  record: payment_api:slo_violations:rate5m
  expr: sum(rate(http_request_duration_bucket{service="payment-api",le="2000"}[5m]))
        / sum(rate(http_request_duration_count{service="payment-api"}[5m]))
        
  # Burn rate = current violation rate / allowed violation rate
  # Alert if burn rate > 14.4 for 1h (fast burn: uses 2% budget in 1h)
  alert: PaymentSLOFastBurn
  expr: |
    (
      1 - payment_api:slo_violations:rate5m
    ) > (14.4 * (1 - 0.999))  # 14.4× the allowed rate
  for: 1m
  annotations:
    summary: "Payment API burning error budget 14× faster than allowed"
    runbook: "https://runbooks/payment-slo-burn"
```

### Exemplars (Metric ↔ Trace Link)
```
Problem: P99 alert fires for payment latency.
  You see the aggregate metric: P99 = 2.3 seconds.
  You want to know: WHICH specific request caused this?
  Without exemplars: you'd have to search through traces manually.

Exemplars solution:
  When computing histograms, attach a "sample" to the high buckets:
  The sample is: { trace_id: "abc123", value: 2.3, timestamp: ... }
  
  In Prometheus exposition format:
  http_request_duration_bucket{le="2000"} 9871 # {trace_id="abc123"} 2.3 1712345678

  In Grafana:
    View latency heatmap → see the P99 data point → hover → click → jump to Jaeger trace
    trace_id="abc123" → full 20-span trace → find fraud-service took 1.8s of the 2.3s
    fraud-service trace → find one specific Redis call took 1.7s (timeout)
  
  Implementation in Spring Boot OTEL:
    Histogram automatically attaches trace_id to high-latency buckets
    Prometheus scrapes exemplars: GET /metrics with Accept: application/openmetrics-text
    Grafana: shows exemplars as dots on the histogram; click to open trace
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K TPS):
  Jaeger all-in-one (single container)
  1% head-based sampling
  Basic Grafana dashboard

Phase 2 (1K → 10K TPS):
  Jaeger distributed (collector + query + storage)
  Cassandra backend for Jaeger
  Structured logging with trace_id injection
  Log → Elasticsearch (ELK stack)

Phase 3 (10K → 50K TPS):
  OTEL Collector as gateway (fan-out to multiple backends)
  Tail-based sampling (keep errors + slow traces)
  Grafana Tempo (S3 backend; cheaper than Cassandra)
  Loki for logs (replace Elasticsearch for cost)
  Exemplars: link metrics to traces

Phase 4 (global, 50K+ TPS):
  Regional OTEL collectors (reduce cross-region telemetry traffic)
  Adaptive sampling (ML-based: keep traces that look anomalous)
  Long-term trace archive (S3 Glacier; 90-day accessibility)
  Service dependency graph auto-generated from traces
```

---

## Reliability Strategy
- **OTEL Collector HA**: 10 instances behind LB; if collector fails, SDK retries with backoff
- **Tail sampling buffer loss**: if collector crashes, buffered 10s of spans lost; acceptable (rare events; most traces still complete)
- **Jaeger/Tempo storage HA**: replicated; reads continue if one node fails
- **Impact on services**: OTEL SDK is async (telemetry sent in background thread); service latency unaffected by collector slowness
- **Circuit breaker**: if OTEL Collector is unavailable, SDK drops telemetry (not service requests)

---

## Security Considerations
- **PII in traces**: payment.id OK; card numbers or user passwords must NOT appear in spans → span sanitizer in OTEL Collector (regex scrubber on attribute values)
- **Trace data access**: traces contain service internals; access restricted to engineers + on-call; not visible to customers
- **Internal endpoints in traces**: HTTP URLs may contain internal service names → acceptable (traces are internal only)
- **Trace poisoning**: malicious trace context in incoming request → OTEL SDK validates format; invalid → new trace (not hijacked)
- **Log levels in production**: DEBUG logs disabled in production (too expensive); INFO and above only

---

## Observability (Meta-observability)
```
OTEL Collector health:
  otelcol_process_cpu_seconds_total
  otelcol_receiver_accepted_spans_total
  otelcol_exporter_sent_spans_total
  otelcol_processor_dropped_spans_total  (alert if > 0; sampling dropped spans OK; errors are not)
  tail_sampler_buffer_size              (alert if > 80% full → collector capacity issue)

Jaeger/Tempo health:
  jaeger_query_requests_total
  tempo_ingester_bytes_received_total
  tempo_compactor_blocks_total

Application tracing coverage:
  % of services with auto-instrumentation (target 100%)
  avg_spans_per_trace (baseline; anomaly = missing instrumentation)
  traces_without_root_span (orphaned spans; context propagation bug)
```

---

## Chaos Testing
```
Experiment 1: OTEL Collector Failure
  Inject:    Kill all OTEL Collector instances
  Expected:  Services continue to function (SDK drops telemetry; no service impact)
  Expected:  SDK buffers spans in memory briefly; drops after buffer full
  Expected:  Alert fires: "OTEL collector unavailable; telemetry being dropped"
  Expected:  When collector recovers: new spans flowing; buffered spans lost
  Red flag:  Service requests fail because of OTEL collector failure (telemetry in critical path)

Experiment 2: Trace Propagation Across Kafka
  Inject:    Send payment; trace should span API → payment-service → Kafka → analytics
  Expected:  All spans share same trace_id
  Expected:  Jaeger: single trace view shows all 4 services with parent-child relationships
  Expected:  Kafka span shows: produced message headers contain traceparent
  Red flag:  Analytics span has different trace_id (context not propagated through Kafka)

Experiment 3: Tail Sampling Correctness
  Inject:    Send 1000 requests; 10 are errors; 10 are slow (> 500ms); 980 are normal
  Expected:  100% of 10 error traces → kept (10 in Jaeger)
  Expected:  100% of 10 slow traces → kept (10 in Jaeger)
  Expected:  ~1% of 980 normal traces → ~10 kept (±5 acceptable for probabilistic)
  Expected:  Total stored: ~30 traces (not all 1000)
  Red flag:  Error traces dropped (sampling policy not prioritising errors)

Experiment 4: PII Scrubbing in Traces
  Inject:    Add a span attribute: "payment.card_number" = "4111111111111111"
  Expected:  OTEL Collector processor detects card number pattern (regex)
  Expected:  Attribute value replaced with "[REDACTED]" in stored span
  Expected:  Jaeger shows: payment.card_number = "[REDACTED]"
  Red flag:  Card number stored in Jaeger (PCI violation)
```

---

## Monthly Cost Estimate
```
Scale: 50K TPS, 1M traces/sec, 10% sampling → 100K stored traces/sec

OTEL Collector Cluster:
  10 × c5.2xlarge ($280/mo)                         = $2,800/mo

Grafana Tempo (trace storage, S3-backed):
  Tempo ingesters: 3 × m5.2xlarge ($280/mo)         = $840/mo
  S3 storage: 8.6 TB × 30 days × $0.023/GB          = $5,934/mo
  Tempo compactor: 1 × m5.large ($75/mo)             = $75/mo

Loki (log storage, S3-backed):
  130 TB × $0.023/GB                                 = $2,990/mo
  Loki ingesters: 3 × m5.2xlarge ($280/mo)          = $840/mo

Jaeger Query (UI + API):
  2 × c5.xlarge ($150/mo)                            = $300/mo

Grafana:
  Already in Case 17; no additional cost

Prometheus (metrics; already Case 17):
  Additional Prometheus storage for exemplars: +10%   = $620/mo
─────────────────────────────────────────────────────────────────
Total incremental (traces + logs): ~$14,399/month
Dominant costs: S3 for traces ($5,934) + logs ($2,990) = 62% of cost
Optimize: increase sampling window for tail sampling; reduce stored trace size
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → OTEL SDK: generates telemetry (source)
  → OTEL Collector: receives, processes, routes (transport)
  → Jaeger/Tempo: stores and serves traces (authoritative for trace data)
  → Loki: stores and serves logs (authoritative for log data)
  → Prometheus: stores metrics (Case 17)

□ Sampling strategy:
  → Head-based: simple but misses important traces; only use at edge
  → Tail-based: smart; keeps all errors and slow traces; recommended
  → Adaptive: ML-based anomaly detection (Phase 4)

□ Context propagation verified:
  → HTTP: W3C traceparent header (auto by OTEL SDK)
  → Kafka: message headers (OTEL Kafka instrumentation)
  → gRPC: metadata (OTEL gRPC instrumentation)
  → DB calls: auto-instrumented by OTEL JDBC/R2DBC

□ PII protection:
  → No card numbers, passwords, or tokens in span attributes
  → OTEL Collector processor: regex scrubber on all attribute values
  → Verified in CI: test that no PII patterns appear in test traces

□ Three pillars linked:
  → Metrics → Traces: exemplars (click on P99 data point → trace)
  → Traces → Logs: trace_id in every log line (click trace → filter logs)
  → Logs → Traces: log query by trace_id (find trace from log)

□ SCC LENS:
  STATE: Tempo/Jaeger (trace data), Loki (log data), Prometheus (metrics)
  COORDINATION: trace context propagation (W3C header carries trace_id across all services)
  CONCENTRATION: OTEL Collector is the telemetry aggregation point → 10 instances + LB
```

---

## Staff Engineer Discussion Points

**"When should you use distributed tracing vs just logging?"**
Traces are for latency decomposition across service boundaries. Logs are for understanding what happened within a single service. Use tracing when: (1) a request crosses 3+ services and you need to find which one is slow, (2) you're investigating a latency pattern and need to see the full call graph, (3) you want to understand service dependencies. Use logs when: (1) investigating errors within a single service, (2) debugging business logic, (3) compliance/audit requirements. In practice: both — traces guide you to the right service; logs tell you what happened inside. The trace_id correlation is the bridge.

**"How do you instrument a legacy Spring Boot service that can't have code changes?"**
OpenTelemetry Java agent: attach the OTEL Java agent JAR at JVM startup with no code changes. `java -javaagent:opentelemetry-javaagent.jar -jar myapp.jar`. The agent auto-instruments: Spring MVC (HTTP spans), JDBC (DB query spans), Kafka (producer/consumer spans), gRPC, Redis (Lettuce/Jedis). For most Spring Boot applications, you get 80% observability coverage with zero code changes. The remaining 20% requires manual instrumentation for business-level spans (e.g., "fraud evaluation" span with payment.id attribute). Start with the agent; add manual spans for the high-value operations.

---

## Cheat Sheet Tie-in

```
THREE PILLARS:
  Metrics: aggregate numbers; cheap; alerting; Prometheus (Case 17)
  Logs: event records; expensive; debugging within service; Loki
  Traces: request journey across services; tail-sampled; Jaeger/Tempo
  Link them: exemplars (metric → trace) + trace_id in logs (trace → logs)

TRACE ANATOMY:
  trace_id: same for all spans in one request
  span_id: unique per operation
  parent_span_id: forms the call tree
  W3C traceparent: propagates across HTTP/Kafka/gRPC

TAIL SAMPLING RULE:
  ERROR traces → 100% sample (never miss a failure)
  SLOW traces (> P99 threshold) → 100% sample (never miss a latency issue)
  Normal traces → 1% sample (baseline; cost control)
  Tail-based: decide AFTER trace is complete (10s buffer in OTEL Collector)

SLO BURN RATE:
  SLO: 99.9% of payments complete in < 2s
  Error budget: 0.1% of requests can violate
  Fast burn: > 14.4× normal violation rate for 1h → alarm (uses 2% of monthly budget)
  Slow burn: > 3× normal for 6h → warning

OTEL COLLECTOR PIPELINE:
  Receivers (OTLP gRPC) → Processors (batch, tail_sample, PII scrub) → Exporters (Tempo, Loki)
  Agent per pod (local) → Gateway cluster (central) → Storage backends

SCC LENS:
  STATE: Tempo (traces), Loki (logs), Prometheus (metrics) — separate stores per signal
  COORDINATION: W3C trace context propagation across all services
  CONCENTRATION: OTEL Gateway Collector → single aggregation point → cluster + LB

IAM PLATFORM TIE-IN:
  Auth service traces: span for each OIDC token validation, LDAP bind, MFA check
  Useful for: "which step in the login flow is slow?" → trace shows where
  Compliance: every access decision → trace shows service identity + resource + decision
  Security: anomaly detection on trace patterns (unexpected service calling auth service)
```

---

*Next: Case Study 30 — Multi-Region / Disaster Recovery Architecture*
