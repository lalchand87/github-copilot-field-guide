# Case Study 18 — Analytics Pipeline (Batch + Streaming)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Lambda vs Kappa Architecture, Exactly-Once in Flink, Backfill strategy.

---

## Business Context

An analytics pipeline computes business metrics from raw event data. For a payment platform: daily transaction volume, fraud rate, merchant performance, user cohort analysis, real-time fraud signals. The pipeline serves two audiences: real-time dashboards (operations, fraud team) and batch reports (finance, product, compliance).

The fundamental design tension: batch processing is accurate but stale; streaming is fresh but harder to make accurate. Lambda architecture runs both; Kappa architecture bets on streaming alone.

**Business goals driving architecture:**
- Real-time: fraud team needs fraud metrics within 1 minute
- Batch: finance needs accurate daily totals (no approximation acceptable)
- Backfill: new metrics can be computed on historical data
- Cost: avoid re-computing what can be incrementally updated

---

## Recognition Framework

### Signals from the Problem
```
- Two audiences with different freshness needs: operations (< 1 min) vs finance (batch)
- Raw events must be replayable for backfill (new metric or bug fix)
- Streaming aggregation is approximate at window boundaries (late-arriving events)
- Batch is accurate but delayed by hours
- Huge data volume (100B events/day) → must partition and parallelize
- Aggregations at multiple granularities: per-minute, hourly, daily, weekly
```

### Lambda Architecture vs Kappa Architecture

```
Lambda Architecture (batch + streaming layers):
  Two parallel pipelines:
    Speed layer (streaming): Flink/Spark Streaming → real-time approximate results
    Batch layer (batch): Spark/Hive → exact results (computed on full dataset)
    Serving layer: query reads from both; serves batch result if available, else streaming
  
  Pros: exact batch results; streaming handles freshness; well-understood
  Cons: two codebases to maintain; same logic written twice; operational complexity

Kappa Architecture (streaming only):
  Single pipeline: Kafka → Flink → serving store
  "Reprocessing = replay Kafka with new job"
  
  Pros: single codebase; simpler ops; reprocessing is natural (just replay)
  Cons: Kafka retention limits reprocessing window; stateful streaming is complex
         exactly-once is harder; late-arriving events require longer windows

CHOSEN: Lambda Architecture for this design
  Rationale:
    Finance requires exact batch results (not approximate streaming)
    Fraud team requires < 1 min freshness
    Both cannot be served by streaming alone without enormous state
    Lambda is the accepted pattern for financial analytics

  Implementation discipline to avoid Lambda's complexity trap:
    Shared library: batch and streaming jobs use same aggregation logic
    Same Kafka events as input to both layers
    Result: "two running modes of same code" not "two separate codebases"
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Lambda Architecture | Batch accuracy (finance) + streaming freshness (fraud) |
| Kafka (raw event store) | Replay for backfill; both batch and streaming read from Kafka |
| Flink (streaming) | Stateful windowed aggregation; exactly-once; watermarks for late events |
| Spark (batch) | Distributed processing of 100B events/day; DataFrame API |
| ClickHouse (serving) | Fast aggregation queries on pre-aggregated + raw data |
| Airflow (orchestration) | DAG scheduling for batch jobs; dependency management |

---

## Problem Statement

Analytics pipeline for a payment platform. Ingests 100B events/day, computes metrics at multiple granularities, serves real-time dashboards (< 1 min lag) and accurate batch reports (daily). Supports backfill for new metrics.

---

## Functional Requirements
- Ingest all platform events (payment, auth, fraud, user) from Kafka
- Compute: total transaction volume (count, amount) per merchant, category, region
- Compute: fraud rate, chargeback rate, success rate
- Compute: user cohort metrics, retention, conversion funnels
- Real-time: metric available within 1 minute of event
- Batch: accurate daily/weekly/monthly aggregation
- Backfill: compute historical values for new metrics

## Non-Functional Requirements
- Event throughput: **100B events/day** (1.15M events/sec)
- Real-time latency: **< 1 minute** from event to metric update
- Batch latency: **daily jobs complete within 4 hours** (by 06:00 UTC for previous day)
- Backfill: **1 year of history processable within 24 hours**

---

## Capacity Estimation
```
Kafka throughput:
  1.15M events/sec × 500 bytes avg = 575 MB/sec
  Daily Kafka storage: 575 MB/sec × 86400 = ~50 TB/day
  With RF=3: 150 TB/day in Kafka
  7-day retention: 1.05 PB in Kafka cluster

Spark batch:
  100B events × 500 bytes = 50 TB/day raw input
  After parquet compression (5×): ~10 TB/day in data lake
  1 year: ~3.6 PB in S3 data lake (parquet)

ClickHouse serving layer:
  Pre-aggregated: hourly × 10K merchants × 50 metrics = 5M rows/day
  Much smaller than raw: < 100 GB/day in ClickHouse
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Lambda Architecture | Batch for accuracy; streaming for freshness; both from Kafka |
| Kafka | Raw event store; replayable; both layers read from same topic |
| Flink | Stateful streaming; watermarks; exactly-once; 1-min windowed aggregation |
| Spark (EMR/Databricks) | Batch processing 50 TB/day; DataFrame API; S3 data lake |
| ClickHouse | Aggregation queries on pre-aggregated data; fast dashboards |
| Airflow | DAG orchestration for batch jobs; retry; dependencies |
| S3 Data Lake (Parquet) | Cheap long-term storage; Spark reads directly; Athena for ad-hoc |

---

## Data Model

```
Data Lake (S3 Parquet — raw events, partitioned by date):
  s3://analytics-data-lake/events/year=2024/month=01/day=15/hour=14/
    part-00001.parquet
    part-00002.parquet
  
  Parquet schema (payment events):
    event_id, event_type, timestamp, user_id, merchant_id,
    amount, currency, status, category, region, device_type,
    payment_method, session_id, ip_hash
  
  Partitioning: year/month/day/hour → Spark/Athena pruning on date queries
  Format: Parquet (columnar; 5-10× compression vs raw JSON)
  Retention: forever (data lake is the permanent store)
```

```sql
-- ClickHouse: pre-aggregated metrics (both streaming and batch write here)
CREATE TABLE merchant_hourly_metrics (
  merchant_id      String,
  hour_bucket      DateTime,
  payment_count    UInt64,
  payment_volume   Decimal(20, 2),
  success_count    UInt64,
  failure_count    UInt64,
  fraud_count      UInt64,
  chargeback_count UInt64,
  avg_amount       Decimal(10, 2),
  layer            LowCardinality(String)  -- 'streaming' or 'batch'
) ENGINE = MergeTree()
  PARTITION BY toYYYYMM(hour_bucket)
  ORDER BY (merchant_id, hour_bucket);

-- Serving query (serving layer merges batch and streaming):
-- SELECT merchant_id, hour_bucket,
--   COALESCE(batch.payment_count, streaming.payment_count) as payment_count
-- FROM merchant_hourly_metrics
-- WHERE layer = 'batch' OR (layer = 'streaming' AND hour_bucket > (SELECT MAX(hour_bucket) FROM merchant_hourly_metrics WHERE layer = 'batch'))
-- Simple: batch result is authoritative when available; streaming fills the gap
```

---

## API Design (Analytics Query API)

```
GET /api/v1/analytics/merchants/{merchant_id}/metrics
  ?granularity=hourly&from=2024-01-01&to=2024-01-31&metric=payment_volume,fraud_rate
Response 200: {
  "merchant_id": "merch-123",
  "granularity": "hourly",
  "metrics": {
    "payment_volume": [
      {"bucket": "2024-01-01T00:00:00Z", "value": 1234567.89, "source": "batch"},
      {"bucket": "2024-01-01T01:00:00Z", "value": 987654.32, "source": "streaming"}
    ],
    "fraud_rate": [...]
  }
}

GET /api/v1/analytics/platform/summary?date=2024-01-15
Response 200: {
  "total_transactions": 8640000,
  "total_volume_inr": 52000000000,
  "success_rate": 0.9823,
  "fraud_rate": 0.0012,
  "source": "batch"  // always batch for daily summaries
}

POST /api/v1/analytics/backfill
Body: { "metric": "new_chargeback_rate", "from": "2023-01-01", "to": "2024-01-01" }
Response 202: { "job_id": "backfill-123", "estimated_duration_hours": 8 }
```

---

## High-Level Architecture

```
Event Sources (payment-service, auth-service, fraud-service)
  │
  └── Produce to Kafka (partitioned by merchant_id for ordering)
        Topics: payment_events, auth_events, fraud_events, user_events

STREAMING LAYER (Flink):
  Flink Job: reads from Kafka → 1-min tumbling windows → aggregate → write to ClickHouse
  
  Per window (per merchant_id, per minute):
    - COUNT(payment_id) → payment_count
    - SUM(amount) → payment_volume
    - COUNT(CASE WHEN status='FRAUD' THEN 1 END) → fraud_count
  
  Watermarks: Flink uses event time (not processing time)
    Watermark = max_event_time_seen - 2_minutes (allows 2-min late arrivals)
    Late events after watermark: go to side output for later reconciliation
  
  Checkpointing: every 30s to S3 → exactly-once semantics
  Output: write aggregated results to ClickHouse with layer='streaming'

DATA LAKE WRITER (Kafka → S3):
  Separate Flink/Kafka Connect job:
    Reads from Kafka → batches into Parquet files (1-hour batches)
    Uploads to S3: events/year=YYYY/month=MM/day=DD/hour=HH/
    This is the raw event archive; source for batch processing

BATCH LAYER (Spark on EMR/Databricks):
  Triggered by Airflow DAG at 01:00 UTC (process previous day's data)
  
  Job: process yesterday's events from S3
    sparkSession.read.parquet("s3://analytics-data-lake/events/year=2024/month=01/day=15/")
    → GROUP BY merchant_id, date_trunc('hour', timestamp)
    → aggregate metrics
    → write to ClickHouse with layer='batch' (overwrites streaming results for same buckets)
  
  Batch result is authoritative; overwrites streaming approximation for completed periods

SERVING LAYER (ClickHouse + Query API):
  Merge logic:
    For hours > 2 days ago: serve batch (always available; accurate)
    For last 2 days: serve streaming (batch not yet complete)
    For last 2 hours: streaming only (most fresh)
  
  Dashboard query: ClickHouse MergeTree → < 2s for 30-day merchant history

AIRFLOW ORCHESTRATION:
  daily_batch_dag:
    raw_data_available (sensor) → spark_batch_job → validate_results → update_clickhouse
    retry: 3× on failure; SLA: complete by 06:00 UTC
```

---

## Detailed Components

### Flink Streaming with Exactly-Once
```java
// Flink job: exactly-once payment aggregation
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Enable exactly-once checkpointing
env.enableCheckpointing(30000);  // every 30 seconds
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setCheckpointStorage("s3://flink-checkpoints/payment-agg/");

// Read from Kafka with WatermarkStrategy
DataStream<PaymentEvent> events = env
    .fromSource(
        KafkaSource.<PaymentEvent>builder()
            .setBootstrapServers("kafka:9092")
            .setTopics("payment_events")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setValueOnlyDeserializer(new PaymentEventDeserializer())
            .build(),
        WatermarkStrategy
            .<PaymentEvent>forBoundedOutOfOrderness(Duration.ofMinutes(2))
            .withTimestampAssigner((event, ts) -> event.getTimestamp()),
        "Kafka Payment Source"
    );

// Tumbling window aggregation per merchant per minute
DataStream<MerchantHourlyMetric> aggregated = events
    .keyBy(PaymentEvent::getMerchantId)
    .window(TumblingEventTimeWindows.of(Time.hours(1)))
    .allowedLateness(Time.minutes(5))  // accept late events up to 5 min after window close
    .sideOutputLateData(lateOutputTag)
    .aggregate(new PaymentAggregator());

// Write to ClickHouse sink (exactly-once via JDBC transaction)
aggregated.addSink(new ClickHouseSink("merchant_hourly_metrics", "streaming"));

// Handle late events (write to separate reconciliation table)
aggregated.getSideOutput(lateOutputTag)
          .addSink(new LateEventReconciliationSink());
```

### Late Event Handling
```
The watermark problem:
  Event timestamp: when the payment actually happened (device time)
  Processing time: when Kafka received the event
  Late event: event with old timestamp arrives after window has closed

  Example:
    Window: 14:00–15:00 payment events (hourly window)
    Window closes at 15:02 (with 2-min watermark)
    At 15:10: an event with timestamp 14:45 arrives (arrived 25 min late due to network)
    This is a late event for the 14:00–15:00 window

  Strategy:
    allowedLateness(5 minutes): accept late events up to 5 min after window close → update result
    After 5 min: route to side output (late event sink)
    Late events: reconciled in daily batch (batch is always accurate anyway)
  
  Why batch handles late events perfectly:
    Batch runs at 01:00 UTC on previous day's data
    All events by then have arrived (even slowest mobile events arrive within hours)
    Batch result is always 100% accurate for completed days
```

### Backfill Strategy
```
New metric added: "chargeback_rate" (wasn't tracked before)
Want to backfill 1 year of history.

Approach:
  1. Kafka retention: only 7 days → cannot replay from Kafka for 1 year
  2. Use S3 data lake (Parquet): contains all raw events forever
  
  Spark backfill job:
    sparkSession.read.parquet("s3://analytics-data-lake/events/year=2023/")
    .filter("event_type = 'CHARGEBACK'")
    .groupBy("merchant_id", date_trunc("hour", "timestamp"))
    .agg(count("*").alias("chargeback_count"))
    .write.mode("append").format("clickhouse").save("merchant_hourly_metrics")
  
  Duration: 3.6 PB / Spark cluster throughput (e.g., 100 nodes = ~360 GB/min)
  → 3.6 PB / (360 GB/min × 60 min) ≈ ~17 hours for 1 year backfill
  → Acceptable for a 24-hour SLA

  Incremental processing (delta backfill):
    If backfill job fails midway: track last processed partition in metadata table
    Resume from last checkpoint rather than starting over

Key lesson: S3 data lake is the permanent archive that enables backfill.
  Kafka is a short-term buffer (7 days).
  S3 Parquet is forever.
  Never throw away raw events — you'll need them for future metrics you haven't designed yet.
```

### Airflow DAG for Daily Batch
```python
with DAG(
    'daily_payment_analytics',
    schedule_interval='0 1 * * *',  # 01:00 UTC daily
    catchup=True,  # IMPORTANT: enables automatic backfill for missed days
) as dag:

    wait_for_data = S3KeySensor(
        task_id='wait_for_data',
        bucket_name='analytics-data-lake',
        bucket_key='events/year={{ execution_date.year }}/month={{ execution_date.strftime("%m") }}/day={{ execution_date.strftime("%d") }}/_SUCCESS',
        timeout=7200,  # wait up to 2 hours for data
    )

    run_spark_job = EMRAddStepsOperator(
        task_id='run_spark_aggregation',
        job_flow_id='{{ var.value.emr_cluster_id }}',
        steps=[{
            'Name': 'Daily Payment Analytics',
            'ActionOnFailure': 'CONTINUE',
            'HadoopJarStep': {
                'Jar': 'command-runner.jar',
                'Args': [
                    'spark-submit',
                    's3://analytics-jobs/daily_payment_analytics.py',
                    '--date', '{{ ds }}',
                    '--output-table', 'merchant_hourly_metrics'
                ]
            }
        }],
        retries=2,
    )

    validate_results = PythonOperator(
        task_id='validate_results',
        python_callable=validate_batch_output,
        op_kwargs={'date': '{{ ds }}', 'expected_min_rows': 100000}
    )

    sla_alert = SLAMissCallback(
        sla=timedelta(hours=5),  # must complete by 06:00 UTC
        callbacks=[notify_on_call]
    )

    wait_for_data >> run_spark_job >> validate_results
```

---

## Scaling Strategy
```
Phase 1 (0 → 1B events/day):
  Kafka → Flink streaming → ClickHouse
  No batch layer; streaming is "accurate enough"
  Simple; acceptable for most dashboards

Phase 2 (1B → 10B events/day):
  Add Spark batch for daily accuracy (Lambda architecture)
  S3 data lake for raw event archival
  Airflow for batch orchestration

Phase 3 (10B → 100B events/day):
  Kafka partitions scaled for throughput
  Flink parallelism increased (more task managers)
  Spark on EMR (auto-scaling cluster; not fixed size)
  ClickHouse cluster sharded by merchant_id

Phase 4 (multi-region):
  Regional Kafka clusters; central analytics lake (S3)
  Regional Spark jobs; global aggregation in separate job
  ClickHouse replicated for multi-region dashboard serving
```

---

## Reliability Strategy
- **Flink exactly-once**: checkpointing every 30s to S3; resume from checkpoint on crash
- **Spark batch retry**: Airflow retries up to 3×; idempotent (batch overwrites same buckets)
- **Data lake as source of truth**: if Flink/Spark both fail, raw data still in S3; reprocess anytime
- **ClickHouse replication**: RF=2; reads continue on node failure
- **SLA monitoring**: Airflow SLA callback if daily batch doesn't complete by 06:00 UTC

---

## Security Considerations
- **PII in raw events**: user_id stored as hash in data lake (same as Case 10 pattern)
- **S3 data lake access**: IAM roles; Spark EMR role can only read/write specific prefixes
- **ClickHouse access**: analytics API auth; analysts get read-only access; no raw event access
- **GDPR deletion**: on erasure request, delete from S3 Parquet (Spark rewrite job to exclude user)
  and from ClickHouse (UPDATE to null for user's rows; no DELETE in ClickHouse MergeTree)

---

## Observability
```
Pipeline metrics:
  kafka_consumer_lag by job        (streaming lag; alert if > 5 min)
  flink_records_per_second         (throughput; alert if drops by > 20%)
  flink_checkpoint_duration_ms     (checkpointing health; alert if > 60s)
  batch_job_duration_seconds       (alert if > 4 hours for daily batch)
  batch_job_success_rate           (alert if < 100%; data not delivered)
  clickhouse_replication_lag       (serving layer health)
  data_freshness_lag_seconds       (actual data age in dashboards)

Data quality metrics (validate after each batch):
  row_count_delta_vs_previous_day  (alert if > 20% change without explanation)
  null_rate_per_column             (data quality check)
  merchant_count_sanity            (expected merchants in output)
```

---

## Chaos Testing
```
Experiment 1: Flink Job Crash Mid-Window
  Inject:    Kill Flink job manager during active window computation
  Expected:  Flink job manager restarts; restores from last checkpoint (< 30s ago)
  Expected:  In-progress window state restored; processing continues
  Expected:  No duplicate results (exactly-once guarantee)
  Expected:  Some streaming metrics temporarily stale (up to 30s checkpoint age)
  Red flag:  Aggregation counts doubled (checkpoint not preventing duplicate processing)

Experiment 2: Spark Batch Job Failure and Retry
  Inject:    Kill Spark job at 50% completion; trigger Airflow retry
  Expected:  Retry starts from scratch (batch is idempotent: overwrites same ClickHouse rows)
  Expected:  ClickHouse data from partial run is overwritten on retry
  Expected:  Final output identical to successful full run
  Red flag:  Partial run data retained; metrics incorrect for partial hour range

Experiment 3: Late Events (5 Minutes After Window Close)
  Inject:    Delay event delivery by 6 minutes; window already closed and results written
  Expected:  Late events after allowedLateness → side output (late event sink)
  Expected:  Streaming result for that window NOT retroactively updated
  Expected:  Batch job (next day) correctly includes these late events
  Expected:  Final daily totals correct (batch is authoritative)
  Red flag:  Late events lost entirely (side output not processed)
```

---

## Monthly Cost Estimate
```
Scale: 100B events/day, Lambda architecture

Kafka (raw event buffer):
  10 × kafka.m5.2xlarge ($300/mo)                    = $3,000/mo
  Storage: 1 PB × $0.10/GB (EBS)                    = $100,000/mo → use S3 Tiered
  With Tiered Storage (MSK): $0.023/GB              = $23,000/mo

Flink (streaming):
  20 × m5.2xlarge ($280/mo)                          = $5,600/mo

Spark EMR (batch; auto-scaling; 4h/day):
  50 × m5.4xlarge × 4h/day × 30 days × $0.043/hr   = $10,320/mo

S3 Data Lake:
  3.6 PB × $0.023/GB                                = $82,800/mo (year 1 growing)
  S3 Intelligent-Tiering after 90 days: ~$50,000/mo

ClickHouse (serving):
  5 × r6g.4xlarge ($800/mo)                          = $4,000/mo

Airflow (managed: AWS MWAA):
                                                      = $1,500/mo
─────────────────────────────────────────────────────────────────
Total: ~$148,220/month
S3 data lake is 55% of cost — raw event storage at 100B events/day is expensive
Optimization: aggressive compression; older partitions to Glacier ($0.004/GB)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → S3 data lake (raw Parquet events): permanent; authoritative; base for all reprocessing.
  → ClickHouse batch layer: authoritative for completed periods (batch has run).
  → ClickHouse streaming layer: approximate; replaced by batch when available.
  → Rule: batch is always correct; streaming is always fresh; serving layer knows which to use.

□ Consistency model?
  → Streaming: eventually consistent (Flink 30s checkpoint lag + window size).
  → Batch: strongly consistent within the batch (reads all events for the period).
  → Serving layer: hybrid (batch for past; streaming for present).

□ Failure mode?
  → Flink crash: checkpoint restores; < 30s data gap; batch will correct.
  → Kafka node down: RF=3; streaming continues; no event loss.
  → Spark batch fails: Airflow retries 3×; SLA alert if 3× insufficient.
  → ClickHouse unavailable: dashboards down; batch writes queue until recovery.

□ Lambda Architecture complexity mitigation:
  → Shared library: same aggregation logic used by Flink and Spark jobs
  → Single Kafka source: both layers read from same topics
  → Clear serving logic: batch overwrites streaming for completed periods

□ SCC LENS:
  STATE: S3 (permanent raw), ClickHouse (streaming + batch aggregated), Kafka (7-day buffer)
  COORDINATION: Airflow orchestrates batch; Flink checkpointing for streaming state
  CONCENTRATION: Kafka ingestion (1.15M events/sec) → partition by merchant_id for even distribution
```

---

*Next: Case Study 19 — Recommendation Engine*
