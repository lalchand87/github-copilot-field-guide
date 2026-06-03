# Case Study 04 — TinyURL Analytics

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Recognition Framework, Pipeline Architecture, Chaos Testing.

---

## Business Context

Analytics is the monetisation layer on top of a URL shortener. Free users get total click count. Paid users get time-series breakdown, geographic data, device breakdown, and referrer analysis. The business model depends on analytics feeling near-real-time — paid users want to see their marketing campaign perform in real-time.

The non-negotiable constraint: **the redirect must never be slowed down by analytics**. A 1ms analytics delay on a redirect is a product failure even if the data is perfectly accurate. This constraint determines the entire architecture.

**Business goals driving architecture:**
- Analytics completely decoupled from the redirect critical path (fire-and-forget)
- Paid analytics dashboard: < 5 minute data freshness
- Raw event replay must be possible — if a bug corrupts aggregates, reprocess from raw
- Storage cost matters: 864M events/day × 200 bytes = ~173 GB/day raw
- Real-time total click counter must be sub-second (Redis INCR)

---

## Recognition Framework

### Signals from the Problem
```
- 10K events/sec is high write volume — but writes are fire-and-forget
- Redirect path has zero tolerance for analytics write latency
- Dashboard queries are aggregations: COUNT, GROUP BY time/geo/referrer/device
- Raw events must be replayable on bug fix (Kafka retention required)
- Real-time total count must be < 1s (too fast for batch aggregation)
- Long-term storage of billions of rows for paid users (90-day raw; forever aggregated)
- Query patterns: time-series (dashboard) + ad-hoc deep queries (columnar needed)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Event Streaming (Kafka) | Decouple redirect path from analytics storage; replayable on bug fix |
| Columnar Storage (ClickHouse) | Aggregation queries on billions of rows — 100× faster than row-based |
| Streaming Aggregation (Flink) | Pre-compute counts in near-real-time; dashboard never queries raw events |
| Write-Behind Cache (Redis INCR) | Sub-second total click counter; approximate but instant |
| Lambda Architecture | Raw events (ClickHouse) + pre-aggregated (PostgreSQL) — both needed |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Synchronous DB write on redirect | Adds 10–50ms to redirect; violates the core constraint |
| PostgreSQL for raw events | Row-based; GROUP BY on 864M events/day takes minutes; columnar is 100× faster |
| Single Kafka partition | Ordering per URL is desirable; single partition is a throughput bottleneck |
| Real-time aggregation only (no raw events) | Bug in Flink aggregation logic = no way to reprocess; raw events are the safety net |
| Redis sorted sets for all click timestamps | Memory cost prohibitive at 10K events/sec; 500M entries for 90-day retention |
| Batch-only processing (no streaming) | Dashboard freshness would be hours; paid users expect minutes |
| PostgreSQL for hourly aggregates | Fine, but only if paired with columnar for raw events — don't use PostgreSQL alone |

### Why Kafka and Not Just Writing to ClickHouse Directly?
```
Direct write to ClickHouse from redirect handler:
  Problem 1: ClickHouse insert takes 10–50ms — back to the same latency problem
  Problem 2: ClickHouse optimal insert batch size is 1K–10K rows; single-row inserts are slow
  Problem 3: If ClickHouse is slow or down, redirect is affected
  Problem 4: No replay capability — if ClickHouse data is corrupted, events are lost

Kafka as buffer:
  Redirect emits to Kafka in < 1ms (async producer, fire-and-forget)
  Redirect returns 302 immediately — does not wait for Kafka ack
  ClickHouse consumer batches 10K events every 5 seconds → efficient bulk insert
  7-day Kafka retention → full replay capability on any consumer bug
  ClickHouse consumer can lag without affecting redirects
```

### Why ClickHouse and Not PostgreSQL for Raw Events?
```
PostgreSQL (row-based):
  Query: SELECT country_code, COUNT(*) FROM clicks WHERE short_code = 'aB3xQ'
         AND clicked_at > NOW() - INTERVAL '30 days'
  At 864M events/day × 30 days = 25.9B rows
  PostgreSQL: full table scan = hours (even with index, aggregation is slow)

ClickHouse (columnar):
  Same query on 25.9B rows: ~2 seconds
  Why: reads only the columns needed (short_code, country_code, clicked_at)
  ClickHouse MergeTree: data sorted by (short_code, clicked_at) → efficient range scan
  Compression: columnar data compresses 5–10× → actually reads 2.6B–5.2B bytes

Verdict: For aggregation on billions of rows, columnar is not optional — it is required.
```

---

## Problem Statement

Extend a URL shortener with an analytics platform. For each short URL, track: total clicks, clicks over time (hourly/daily), geographic distribution, referrer breakdown, and device type. Analytics should be near-real-time (< 5 min lag). The redirect critical path must be completely unaffected.

---

## Functional Requirements
- Record a click event every time a short URL is accessed (async, fire-and-forget)
- Query: total clicks for a short code (real-time, < 1s)
- Query: clicks grouped by hour/day (< 2s for last 30 days)
- Query: top countries, top referrers, top devices (< 2s)
- Dashboard: < 5 min data freshness for paid users
- Raw event export for paid users (last 90 days)
- Retention: raw events 90 days; hourly aggregates forever

## Non-Functional Requirements
- Click recording: **fire-and-forget; redirect must not wait**
- Query latency: **< 2s** for all dashboard queries
- Write volume: **10,000 click events/sec** peak
- Dashboard freshness: **< 5 minutes**
- Availability: analytics can lag; redirect must never be affected

---

## Capacity Estimation
```
Writes:
  10,000 events/sec = 864M events/day
  Per event: ~200 bytes (short_code, timestamp, country, referrer, ua, session_id)
  Daily raw storage: 864M × 200 bytes = ~173 GB/day
  90-day raw retention: ~15 TB (uncompressed); ~2–3 TB compressed (ClickHouse ~7× compression)

Aggregates (pre-computed, stored in PostgreSQL):
  Hourly stats: 10M short_codes × 24h × 100 bytes = ~2.4 GB/day → trivially small
  Country stats: 10M × 250 countries × 1 day × 50 bytes = ~125 GB/day → too large
  → Store only top-10 countries per URL per day (not all 250)

Kafka:
  10,000 events/sec × 200 bytes = 2 MB/sec = ~172 GB/day in Kafka
  7-day retention: ~1.2 TB of Kafka storage (before replication)
  RF=3: ~3.6 TB total Kafka storage for 7 days

Redis:
  1 key per short_code: INCR clicks:{short_code}
  10M active short_codes × 8 bytes = 80 MB → trivially small
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Kafka | Decouple redirect from analytics; 7-day replayable buffer |
| Flink Streaming | 1-min tumbling windows; pre-aggregate before dashboard queries |
| ClickHouse | Columnar storage; 2s queries on 25B rows |
| Redis INCR | Sub-second total click counter (approximate) |
| Pre-aggregation | Dashboard queries hit PostgreSQL hourly_stats, not raw ClickHouse |

---

## Data Model

```sql
-- Raw events in ClickHouse (append-only, never update or delete)
CREATE TABLE click_events (
  short_code    String,
  clicked_at    DateTime,
  country_code  LowCardinality(String),   -- 250 values; LowCardinality = dictionary encoding
  referrer      String,
  device_type   LowCardinality(String),   -- mobile / desktop / tablet / bot
  browser       LowCardinality(String),
  session_id    String,
  ip_hash       String                    -- SHA-256(IP); never store raw IP (GDPR)
) ENGINE = MergeTree()
  PARTITION BY toYYYYMM(clicked_at)       -- partition by month for efficient purging
  ORDER BY (short_code, clicked_at)       -- primary sort: all clicks for a URL together
  TTL clicked_at + INTERVAL 90 DAY DELETE; -- auto-delete raw events after 90 days

-- Pre-aggregated hourly stats in PostgreSQL (fast dashboard queries)
CREATE TABLE hourly_stats (
  short_code    VARCHAR(8),
  hour_bucket   TIMESTAMP,       -- truncated to hour: 2024-01-01 14:00:00
  click_count   BIGINT           DEFAULT 0,
  PRIMARY KEY (short_code, hour_bucket)
);

-- Pre-aggregated country stats (top countries per URL per day)
CREATE TABLE daily_country_stats (
  short_code    VARCHAR(8),
  day_bucket    DATE,
  country_code  CHAR(2),
  click_count   BIGINT,
  PRIMARY KEY (short_code, day_bucket, country_code)
);

-- Real-time counters in Redis
-- INCR clicks:{short_code}          → approximate running total
-- INCR clicks_today:{short_code}    → today's count (EXPIRE at midnight)
```

---

## API Design

```
GET /api/v1/urls/{short_code}/analytics
Query params: period=7d|30d|90d  (default: 7d)
Response 200: {
  "short_code": "aB3xQ",
  "period": "7d",
  "total_clicks": 14203,
  "total_clicks_today": 342,
  "clicks_by_hour": [
    {"hour": "2024-01-01T14:00:00Z", "count": 342},
    ...
  ],
  "top_countries": [
    {"country": "IN", "country_name": "India", "count": 4200, "pct": 29.5},
    ...
  ],
  "top_referrers": [{"referrer": "twitter.com", "count": 2100}, ...],
  "top_devices":   [{"device": "mobile", "count": 9800, "pct": 68.9}]
}

GET /api/v1/urls/{short_code}/analytics/realtime
Response 200: {
  "clicks_last_5min": 43,
  "clicks_last_hour": 892,
  "clicks_today": 14203
}

GET /api/v1/urls/{short_code}/analytics/export?from=2024-01-01&to=2024-01-31
Response 202: { "export_id": "exp-123", "status": "processing" }
→ Async: generates Parquet file from ClickHouse → S3 presigned download URL
```

---

## High-Level Architecture

```
Redirect Service (critical path — never modified by analytics)
  │
  └── On each redirect:
        producer.sendAsync("click-events", clickEvent)
        // Returns immediately; does NOT wait for Kafka ack
        // If Kafka producer buffer is full: drop event (analytics < availability)

Kafka Topic: click-events
  Partitions:       64 (partition by short_code → ordering per URL)
  Retention:        7 days
  Replication:      3
  Producer setting: acks=1 (leader ack only; fire-and-forget acceptable)
  Compression:      lz4 (fast; good ratio for JSON)

Consumer Group 1: ClickHouse Writer
  Batch size:       10,000 events or 5 seconds (whichever first)
  Inserts to:       ClickHouse click_events table
  Consumer lag:     alert if > 100K messages (~10 seconds of events)

Consumer Group 2: Flink Streaming Aggregator
  Window:           1-minute tumbling window
  Key by:           short_code
  Aggregates:       COUNT(), COUNT() by country, COUNT() by referrer, COUNT() by device
  Output:           PostgreSQL UPSERT into hourly_stats, daily_country_stats
  Checkpointing:    every 30s to S3 (resume from checkpoint on restart)

Consumer Group 3: Redis Real-Time Counter
  For each event:   INCR clicks:{short_code}
                    INCR clicks_today:{short_code} (with EXPIREAT midnight)
  Latency:          < 1ms per event; handles 10K/sec easily

Dashboard API (query service, separate from redirect service):
  GET total clicks     → Redis (< 1ms, approximate)
  GET time-series      → PostgreSQL hourly_stats (< 50ms, indexed)
  GET country stats    → PostgreSQL daily_country_stats (< 50ms)
  GET ad-hoc deep      → ClickHouse (< 2s on 25B rows)
  GET raw export       → ClickHouse → async Parquet → S3
```

---

## Detailed Components

### Kafka Producer (In Redirect Handler)
```java
// In redirect service — fire-and-forget
public void recordClick(ClickEvent event) {
    ProducerRecord<String, ClickEvent> record =
        new ProducerRecord<>("click-events",
                             event.getShortCode(),  // partition key
                             event);

    // sendAsync: returns immediately; callback is async
    kafkaProducer.send(record, (metadata, exception) -> {
        if (exception != null) {
            // Log and drop: analytics loss is acceptable; redirect is not
            metrics.increment("click_event.kafka.drop");
        }
    });
    // redirect handler continues immediately — Kafka is NOT in the critical path
}

// Producer config for fire-and-forget:
//   acks=1              (only leader ack; not all replicas)
//   linger.ms=10        (batch events for 10ms before sending; more efficient)
//   buffer.memory=32MB  (producer buffer; if full: drop oldest → analytics < redirect)
//   compression.type=lz4
```

### Flink Aggregation Job
```java
// 1-minute tumbling window aggregation
DataStream<ClickEvent> clicks = kafkaSource.readFromKafka("click-events");

clicks
  .keyBy(ClickEvent::getShortCode)
  .window(TumblingProcessingTimeWindows.of(Time.minutes(1)))
  .aggregate(new ClickAggregator())
  .addSink(new PostgresSink());

// ClickAggregator: for each (short_code, 1-min window):
//   - total count
//   - count by country_code
//   - count by device_type
//   - count by referrer (top 10)

// PostgresSink uses UPSERT:
//   INSERT INTO hourly_stats (short_code, hour_bucket, click_count)
//   VALUES (?, ?, ?)
//   ON CONFLICT (short_code, hour_bucket)
//   DO UPDATE SET click_count = hourly_stats.click_count + EXCLUDED.click_count;

// Checkpointing: every 30s to S3
//   On job restart: resume from last checkpoint
//   Flink exactly-once semantics with idempotent UPSERT = correct aggregates
```

### ClickHouse Insert Batching
```
Consumer reads from Kafka in batches:
  Batch when: accumulated 10,000 events OR 5 seconds elapsed (whichever first)

  INSERT INTO click_events VALUES (batch of 10,000 rows)
  → ClickHouse: bulk insert is fast; single-row insert is slow (MergeTree overhead)
  → 10K rows / 5s = 2K rows/sec average; ClickHouse handles this easily

  Commit Kafka offset AFTER successful ClickHouse insert
  → At-least-once delivery (duplicate events possible on crash)
  → ClickHouse ReplacingMergeTree or dedup by session_id if exact counts needed
  → For analytics: approximate counts are fine; at-least-once is acceptable
```

### Real-Time Counter Architecture
```
Redis counters (approximate; Redis is not source of truth):
  INCR clicks:{short_code}              → lifetime total (never expires)
  INCR clicks_today:{short_code}        → today's count
  EXPIREAT clicks_today:{short_code}    → expire at midnight UTC

Why approximate?
  Redis INCR is not transactional with Kafka consumption
  If Redis crashes: counter resets to 0
  Solution: periodically sync Redis from ClickHouse aggregate (nightly job)
  For billing: use ClickHouse (authoritative); Redis is display-only

Real-time "last 5 minutes" query:
  SELECT COUNT(*) FROM click_events
  WHERE short_code = ? AND clicked_at > NOW() - INTERVAL 5 MINUTE
  ClickHouse: < 100ms even on hot URLs (MergeTree primary key optimisation)
```

---

## Scaling Strategy
```
Phase 1 (0 → 100 events/sec):
  Write clicks synchronously to PostgreSQL
  SELECT COUNT(*) for total clicks
  No streaming; batch aggregation acceptable at this scale

Phase 2 (100 → 1K events/sec):
  Kafka buffer + async write to PostgreSQL
  Redis INCR for real-time total
  Background aggregation job (runs every 5 min)

Phase 3 (1K → 10K events/sec):
  ClickHouse for raw events (PostgreSQL can no longer handle raw at this volume)
  Flink streaming aggregation (1-min windows)
  PostgreSQL for pre-aggregated data only (not raw events)

Phase 4 (10K → 100K events/sec):
  Kafka partitions scaled to 128 (more parallelism)
  Flink parallelism increased (more task managers)
  ClickHouse cluster (3+ nodes, sharded by short_code hash)

Phase 5 (100K+ events/sec):
  Tiered storage: ClickHouse (hot 90 days) → S3 Parquet (cold, Athena for queries)
  Multi-region Kafka with MirrorMaker for geo-distributed deployments
  Separate ClickHouse clusters per region
```

---

## Reliability Strategy
- **Kafka durability**: RF=3, acks=1 for producer (fire-and-forget acceptable; analytics < redirect)
- **Flink checkpointing**: every 30s to S3; resume from checkpoint on crash; no event loss
- **ClickHouse replication**: RF=2 minimum; reads can go to replica on primary failure
- **Redis counter recovery**: nightly sync job reconciles Redis total with ClickHouse aggregate
- **Flink exactly-once**: idempotent UPSERT in PostgreSQL sink; correct aggregates even on reprocess

---

## Security Considerations
- **PII in click events**: never store raw IP address; store SHA-256(IP + daily_salt) — GDPR compliant; supports deduplication without storing PII
- **Geo lookup**: IP → country mapping done at API gateway level before Kafka emit; raw IP never reaches analytics pipeline
- **Analytics access control**: URL owners only; enforce short_code ownership on every API call
- **Bot filtering**: detect bot user-agents at gateway; set device_type='bot'; exclude from default dashboard (but retain in raw for analysis)
- **Export access**: raw event exports are large (potentially GBs); rate limit export requests; require paid tier

---

## Observability
```
Kafka metrics:
  consumer_lag (per consumer group)    → alert if > 100K messages (~10s of events)
  producer_drop_rate                   → alert if > 0 (events lost before Kafka)

ClickHouse metrics:
  insert_rate (rows/sec)               → should track Kafka consumer rate
  insert_latency_p99                   → alert if > 1s (batching not working)
  query_latency_p99                    → alert if > 5s on dashboard queries

Flink metrics:
  checkpoint_duration_ms               → alert if > 60s (checkpointing too slow)
  records_processed_per_sec            → should track Kafka producer rate
  job_restarts_total                   → alert if > 0 (job instability)

Redis metrics:
  redis_counter_drift                  → nightly: compare Redis total vs ClickHouse total
                                         alert if drift > 5%

Business metrics (dashboard):
  clicks_per_short_code_per_min        → detect viral URLs in real-time
  top_10_short_codes_by_click_rate     → dashboard feature
  analytics_pipeline_freshness_seconds → time from click event to dashboard visibility
                                         target: < 300 seconds (5 min)
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Kafka broker down (1 of 3) | RF=3 absorbs; no disruption | RF=3 standard; alert if 2nd broker shows issues |
| Kafka producer buffer full | Click events dropped (analytics loss) | Producer configured to drop oldest; redirect unaffected |
| Flink job crashes | Aggregation stops; raw events safe in Kafka | Restart from S3 checkpoint; replay from Kafka |
| ClickHouse node down | Raw queries slower or fail | Replication; queries route to healthy node |
| Redis counter lost | Real-time totals reset to 0 | Nightly sync from ClickHouse; Redis is display-only |
| PostgreSQL down | Dashboard time-series unavailable | Fallback: serve from ClickHouse directly (slower but correct) |
| Consumer lag spike | Dashboard freshness > 5 min | Auto-scale Flink; alert on-call; add Kafka partitions |

---

## Chaos Testing
```
Experiment 1: Kill Kafka (simulate outage)
  Inject:    Stop all Kafka brokers
  Expected:  Redirects continue normally (producer buffer absorbs briefly; then drops)
  Expected:  Analytics events dropped (acceptable; analytics < redirect)
  Expected:  On Kafka recovery: producer reconnects; new events flow; old dropped events lost
  Red flag:  Redirect latency increases (Kafka producer is blocking the redirect path)
  Red flag:  Redirect returns 5xx (Kafka failure propagated to redirect handler)

Experiment 2: Kill Flink Job
  Inject:    Kill Flink job manager
  Expected:  Kafka consumer lag starts growing (events accumulating in Kafka)
  Expected:  Raw events still flowing to ClickHouse (separate consumer group)
  Expected:  On Flink restart: job resumes from last S3 checkpoint
  Expected:  Flink replays Kafka events from checkpoint offset; aggregates catch up
  Expected:  Dashboard shows stale data during outage; recovers within minutes of restart
  Red flag:  Events lost (Flink not checkpointing; consumer offset committed before processing)

Experiment 3: Hot URL Flood (1M clicks to one URL in 1 minute)
  Inject:    Send 1M click events with same short_code in 1 minute
  Expected:  All events land in same Kafka partition (partition by short_code)
  Expected:  ClickHouse consumer handles partition backlog; insert latency increases
  Expected:  Flink aggregates 1M clicks into 1-min window; single UPSERT to PostgreSQL
  Expected:  Redis INCR handles 1M/min easily (Redis can do 1M ops/sec)
  Red flag:  Kafka partition lag grows unbounded (consumer can't keep up)
  Mitigation: Sub-partition by hash(short_code + timestamp) % N for extremely hot URLs

Experiment 4: ClickHouse Disk Full
  Inject:    Fill ClickHouse disk to 95%
  Expected:  Alert fires at 80% disk usage
  Expected:  TTL deletion job runs (removes events > 90 days)
  Expected:  Insert errors at 100% disk; Kafka consumer lag grows
  Expected:  No redirect impact (Kafka buffers events)
  Red flag:  ClickHouse node crashes (instead of returning insert errors gracefully)
```

---

## Monthly Cost Estimate
```
Scale assumption: 10K events/sec, 10M short codes, 90-day raw retention

Kafka Cluster:
  3 × kafka.m5.large ($150/mo)                   = $450/mo
  Storage: 3.6 TB × $0.10/GB                     = $360/mo
  Total Kafka:                                    = $810/mo

ClickHouse Cluster:
  3 × r6g.2xlarge (64GB RAM, $500/mo)            = $1,500/mo
  Storage: ~2.5 TB compressed SSD × $0.10/GB     = $250/mo
  Total ClickHouse:                               = $1,750/mo

Flink (managed: Amazon Kinesis Data Analytics or self-managed):
  Self-managed: 3 × m5.xlarge ($150/mo)          = $450/mo

PostgreSQL (pre-aggregated stats):
  1 × db.r6g.large ($200/mo) + 1 read replica    = $400/mo

Redis (real-time counters):
  1 × cache.r6g.medium ($60/mo)                  = $60/mo
  (80 MB of counters → smallest node is fine)

Monitoring + networking:
                                                  = $300/mo
────────────────────────────────────────────────────────────
Total: ~$3,770/month

Cost drivers:
  ClickHouse:  46% of total (compute + storage)
  Kafka:       21% of total (brokers + storage)
  Flink:       12% of total

Optimisation levers:
  ClickHouse compression (7×): 15 TB raw → 2.1 TB stored → saves ~$1,290/mo vs uncompressed
  Cold tier: events > 30 days → S3 Parquet (Athena for queries) → saves ~$800/mo
  Reserved instances: ~30% discount on stable workloads
  Managed Flink: higher cost but lower ops burden
```

---

## Migration Story

### Current State
Click counts stored as `UPDATE urls SET click_count = click_count + 1 WHERE short_code = ?`. At 1K RPS: UPDATE lock contention on hot URLs causes redirect latency to spike to 200ms. No time-series. No geo data.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Decouple analytics write to Redis INCR (zero downtime)
  Action:
    Replace synchronous PostgreSQL UPDATE with async Redis INCR
    Redirect handler: INCR clicks:{short_code} (still in critical path but ~0.5ms vs 50ms)
    Background sync job: periodically flush Redis counters to PostgreSQL for backup
  Monitor:
    Redirect latency (should drop from 200ms to < 10ms)
    Redis INCR rate (should track redirect rate)
  Rollback:
    Re-enable synchronous PostgreSQL UPDATE; disable Redis INCR

Step 2: Add Kafka buffer (zero downtime)
  Action:
    Add async Kafka emit in redirect handler (alongside Redis INCR for now)
    Deploy Kafka cluster; verify events flow correctly
    Deploy ClickHouse cluster; deploy ClickHouse consumer
    Run ClickHouse consumer for 1 week; verify event counts match Redis
  Monitor:
    Kafka consumer lag (should be near-zero)
    ClickHouse event count vs Redis counter (drift should be < 1%)
  Rollback:
    Stop Kafka emit; remove Kafka consumer

Step 3: Add Flink aggregation (zero downtime, additive)
  Action:
    Deploy Flink job; subscribe to click-events Kafka topic
    Flink outputs to PostgreSQL hourly_stats and daily_country_stats
    Dashboard switches to PostgreSQL hourly_stats for time-series (faster than raw ClickHouse)
  Monitor:
    Flink checkpoint success rate
    Dashboard freshness (should be < 5 min)
    Flink consumer lag vs ClickHouse consumer lag
  Rollback:
    Stop Flink job; dashboard falls back to Redis for totals only

Step 4: Remove Redis from redirect critical path (zero downtime)
  Action:
    Move Redis INCR from redirect handler to Kafka consumer (consumer does the INCR)
    Redirect handler now only emits to Kafka (true fire-and-forget; no Redis in critical path)
    Verify: redirect latency drops by ~0.5ms (Redis RTT removed)
  Monitor:
    Redis counter accuracy (should still match ClickHouse counts within ~1%)
    Redirect latency (should be < 5ms now including Kafka emit)
  Rollback:
    Re-add Redis INCR directly in redirect handler

Step 5: Remove PostgreSQL click_count column (planned maintenance)
  Action:
    Dashboard now served entirely from Redis (total) + PostgreSQL hourly_stats (time-series)
       + ClickHouse (ad-hoc/deep queries) + PostgreSQL country_stats (geo)
    Drop click_count column from urls table
    Remove sync job
  Monitor:
    No regressions in dashboard data accuracy
  Rollback:
    Restore column with default value 0; re-enable sync from Redis
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → ClickHouse for raw click events (authoritative; 90-day retention).
  → PostgreSQL for pre-aggregated stats (authoritative for dashboard; rebuilt from ClickHouse).
  → Redis for real-time total (approximate; display-only; not used for billing).
  → If Redis and ClickHouse disagree: ClickHouse wins.

□ Consistency model?
  → Click recording: fire-and-forget (Kafka producer acks=1; no guarantee of delivery).
  → Raw events: at-least-once (Kafka consumer commits offset after ClickHouse insert).
  → Aggregates: eventually consistent (Flink 1-min window; < 5 min lag to dashboard).
  → Real-time total: approximate (Redis; may drift from ClickHouse total by < 1%).

□ Failure mode?
  → Kafka down: click events dropped; redirect unaffected.
  → Flink down: aggregation stops; events safe in Kafka; dashboard stale.
  → ClickHouse down: raw queries fail; dashboard falls back to PostgreSQL aggregates.
  → Redis down: real-time totals unavailable; fall back to PostgreSQL hourly_stats total.

□ Hot partitions?
  → Viral URL: all events with same short_code land in same Kafka partition.
  → Mitigation: sub-partition by hash(short_code + timestamp bucket) for extreme cases.
  → ClickHouse: data sorted by short_code → efficient point queries despite hot URL.

□ Cost driver?
  → ClickHouse compute + storage (46% of total).
  → Optimise: ClickHouse compression (7×); cold tier to S3 Parquet after 30 days.

□ Security risk?
  → PII: raw IP address in click events. Mitigation: hash IP before Kafka emit.
  → Analytics access: must enforce short_code ownership on every API call.
  → Export abuse: raw event exports can be GBs; rate limit + paid tier required.

□ Operational burden?
  → Flink job management: checkpoint monitoring, job restart on failure.
  → ClickHouse capacity planning: columnar store grows predictably but needs sizing.
  → Kafka partition rebalancing when adding brokers.
  → Nightly Redis reconciliation job (compare Redis total vs ClickHouse aggregate).

□ Migration path?
  → Sync PostgreSQL UPDATE → Redis INCR → Kafka buffer → Flink aggregation
  → Remove Redis from critical path → Drop PostgreSQL click_count column.
  → Each step independently rollbackable.
```

---

## Staff Engineer Discussion Points

**"How do you handle a URL that gets 1M clicks in 1 minute?"**
Kafka partitioned by short_code means all 1M events land in one partition. The ClickHouse consumer can handle this — bulk inserts are fast. Flink processes the full 1-min window and emits one aggregate. Redis INCR handles 1M ops/min (Redis does 1M ops/sec). The only concern is Kafka partition throughput — at 1M events/min × 200 bytes = 200 MB/min per partition. Kafka handles this, but you'd want to sub-partition that URL across 4 partitions during extreme events.

**"Is the Redis click count authoritative for billing?"**
No. Redis is display-only and approximate. For billing (if you charge per click), always query ClickHouse for the authoritative count. Redis may have drifted if it was restarted, if there were Redis failures, or if the nightly sync hasn't run yet. The nightly sync job reconciles Redis from ClickHouse precisely for this reason.

**"How do you replay analytics for a bug in the Flink aggregation logic?"**
Reset the Flink consumer group offset to the start of the affected period. Flink replays events from Kafka. The PostgreSQL UPSERT sink is idempotent (ON CONFLICT DO UPDATE) — reprocessing the same events produces the correct final aggregate. Kafka's 7-day retention limits replay to the last 7 days. Beyond that, replay from ClickHouse (export raw events as Kafka-compatible input).

**"How do you backfill a new metric you want to add (e.g. OS breakdown)?"**
Within 7 days of Kafka retention: add the new field to the Flink aggregation job and replay from Kafka. Beyond 7 days: the new metric doesn't exist in old events (it wasn't captured). Backfill options: (1) accept that the metric only has data from its introduction date, (2) use ClickHouse raw events to compute the metric retrospectively if the field existed in raw events. This is why capturing rich raw events (all fields you might ever need) in ClickHouse is worth the storage cost.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Write must not slow down critical path"      → Kafka fire-and-forget (not sync write)
  "Aggregate billions of rows in < 2s"          → ClickHouse columnar (not PostgreSQL row-based)
  "Real-time counter < 1s"                      → Redis INCR (not aggregation query)
  "Bug recovery on aggregation"                 → Kafka retention (replay capability)
  "Dashboard freshness < 5 min"                 → Flink streaming (not batch job)

DECISION TREE USAGE:
  Need event replay?                            → Kafka (not RabbitMQ; RabbitMQ deletes on consume)
  Analytics on time-series billions of rows?    → ClickHouse (columnar MergeTree)
  Real-time approximate count?                  → Redis INCR (not ClickHouse query per request)
  Freshness < 5 min?                            → Streaming aggregation (Flink/Spark Streaming)

PATTERNS REJECTED (and why):
  Sync write on redirect    → violates core constraint; 10–50ms latency added to redirect
  PostgreSQL for raw events → GROUP BY on 25B rows takes hours; columnar required
  No raw events (agg only)  → cannot replay on bug fix; raw events are the safety net
  Redis sorted sets         → prohibitive memory at 10K events/sec scale

SCC LENS:
  STATE:
    Raw events (authoritative): ClickHouse (append-only; 90-day TTL)
    Pre-aggregated stats (fast): PostgreSQL hourly_stats, daily_country_stats
    Real-time counter (approx):  Redis INCR (display-only; not billing source)
    In-flight buffer:            Kafka (7-day retention; replayable)

  COORDINATION:
    No coordination on write path: Kafka is fire-and-forget; redirect never waits
    Flink checkpointing: coordinates "where did I process up to" with S3 state
    Nightly reconciliation: syncs Redis approximation with ClickHouse truth
    Idempotent UPSERT in PostgreSQL: correct aggregates even on Flink replay

  CONCENTRATION:
    Viral URL: all events → same Kafka partition → same ClickHouse partition
    Mitigation: sub-partition by short_code + timestamp bucket for extreme cases
    Kafka itself: 64 partitions spread across 3 brokers → no single hotspot
    Redis: 10M keys × 8 bytes = 80 MB → never a memory concern

INTERVIEW ANSWER TRIGGER:
  "Design TinyURL analytics" →
    1. Spot the core constraint: redirect must NEVER wait for analytics
    2. Fire-and-forget via Kafka: producer.sendAsync(); no acks waiting
    3. Pipeline: Kafka → ClickHouse (raw) + Flink → PostgreSQL (aggregated) + Redis (realtime)
    4. Reject: sync write (latency), PostgreSQL for raw (aggregation speed), no raw events (no replay)
    5. SCC: State=ClickHouse+PostgreSQL+Redis, Coordination=Flink checkpoints,
            Concentration=viral URL → sub-partition
    6. Billing: always ClickHouse; never Redis (approximate)
```

---

*Next: Case Study 05 — API Gateway*
