# Case Study 10 — Audit Logging Service

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Immutability guarantees, Tiered Storage, Compliance requirements.

---

## Business Context

An audit log is legal evidence. For a payment platform, it answers: "Who did what, to which account, from which device, at what exact time — and can you prove that record was not altered?" PCI-DSS, GDPR, and RBI all require immutable audit trails for payment operations, admin actions, and data access. A tampered audit log is worse than no audit log — it creates liability.

The audit log is write-once, read-rarely. It is high-volume on writes (every payment, every auth event, every admin action). Reads are infrequent but non-negotiable when needed (incident investigation, compliance audit, legal discovery).

**Business goals driving architecture:**
- Every significant action recorded exactly once, in order, and immutably
- Tamper detection: if a log entry is altered, we can prove it
- Compliance: GDPR (personal data audit trail), PCI-DSS (payment audit), RBI (banking audit)
- Cost: 90-day hot storage; multi-year cold storage; tiered to manage cost
- Query: "show me all actions by user X in the last 30 days" in < 5 seconds

---

## Recognition Framework

### Signals from the Problem
```
- Write-once, append-only: no updates, no deletes (immutability requirement)
- High write volume (100K events/sec) but rare reads
- Compliance requires tamper detection (hash chain or WORM storage)
- Long retention (7–10 years) but tiered by recency
- Queries are ad-hoc and retrospective (who did what when)
- Write path must not slow down the producing service (fire-and-forget)
- Legal hold: specific events must be retained even beyond standard TTL
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Append-Only Log | Immutability requirement: no UPDATE or DELETE ever |
| Kafka (write buffer) | Decouple producing services from audit log persistence; replayable |
| ClickHouse (columnar) | Aggregation queries on billions of events: "all actions by user X in 30 days" |
| Hash Chain | Tamper detection: each entry's hash includes prior entry's hash |
| Tiered Storage (S3) | 90-day hot ClickHouse + cold S3 Parquet for 7-year retention |
| WORM (Write Once Read Many) | S3 Object Lock: legal guarantee of immutability |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| PostgreSQL for audit log | Row-based; GROUP BY on 10B rows takes hours; columns compress poorly; expensive at audit scale |
| Synchronous write in request path | Audit log write failure would fail the payment transaction; must decouple |
| Mutable log (allow corrections) | Corrections are themselves audit events; never modify existing entries |
| Single tier (hot forever) | 7-year hot storage is expensive; cold tier (S3 Parquet) is 20× cheaper |
| No tamper detection | Compliance requires ability to prove log integrity to auditors |

### Why Append-Only + Hash Chain?

```
Requirement: prove that audit log entry #4821 has not been altered since it was written.

Option A: Signed entries only
  Each entry signed with private key at write time
  Verify: check signature
  Problem: if attacker compromises private key, they can re-sign altered entries
  Also: doesn't prove ordering (could replay entries in different order)

Option B: Hash Chain (blockchain-lite, chosen)
  entry_n.hash = SHA-256(entry_n.data + entry_(n-1).hash)
  To verify entry_n: recompute hash from data + prior hash → if mismatch → tampered
  Properties:
    Altering any entry invalidates all subsequent hashes (chain breaks)
    Ordering is cryptographically proven (each entry includes proof of prior state)
    Attacker must recompute entire chain from alteration point to present → detectable
  Implementation:
    Chain anchored in a public ledger (e.g., Bitcoin OP_RETURN) every 24 hours
    External anchor proves chain state at that timestamp; impossible to retroactively alter

Option C: WORM storage only (S3 Object Lock)
  S3 Object Lock: once written, cannot be modified or deleted for retention period
  Problem: WORM proves immutability AFTER archiving, but pre-archive entries are mutable
  Solution: use BOTH hash chain (ongoing verification) AND WORM (archive guarantee)

Chosen approach: Hash chain for in-flight + recent entries; WORM (S3 Object Lock) for archived entries.
```

### Why ClickHouse for Audit Queries?

```
Typical audit query: "All payment actions by user u-12345 from January to March"
  PostgreSQL (row-based):
    SELECT * FROM audit_log WHERE actor_id='u-12345' AND action_time BETWEEN ...
    Table: 10B rows × 200 bytes = 2 TB
    Even with index on actor_id: index scan → random I/O on 2 TB → minutes
  
  ClickHouse (columnar, sorted by actor_id, action_time):
    Same query: reads only actor_id + action_time + action_type columns
    Data is sorted: range scan on actor_id is sequential I/O
    Compression: columnar data compresses 5–10× → reads ~200 MB instead of 2 TB
    Result: ~2 seconds for same query
    
ClickHouse is the correct choice for any query workload described as:
  "Complex WHERE clause on billions of rows" + "GROUP BY or aggregation"
```

---

## Problem Statement

Centralized audit logging service. Every significant action in the payment platform (payment events, auth events, admin actions, data access events) is recorded as an immutable, tamper-evident log entry. Queryable for compliance, incident investigation, and legal discovery.

---

## Functional Requirements
- Ingest audit events from all platform services (fire-and-forget)
- Guarantee: every ingested event is stored exactly once and never modified
- Tamper detection: cryptographic proof that log entries are unaltered
- Query: by actor (user/service), by resource (account/payment), by time range, by action type
- Retention: 90 days hot (ClickHouse); 7 years cold (S3 WORM)
- Legal hold: specific events exempt from TTL deletion
- Compliance report: generate audit report for given actor/time range

## Non-Functional Requirements
- Write throughput: **100,000 events/sec** (peak)
- Write latency added to producing service: **< 1ms** (async; fire-and-forget)
- Query latency: **< 5 seconds** for 30-day actor query on 10B events
- Availability: **99.99%** (cannot lose audit events)
- Immutability: **cryptographic guarantee** — tamper is detectable

---

## Capacity Estimation
```
Write volume:
  100,000 events/sec = 8.64B events/day
  Per event: ~500 bytes (actor, action, resource, metadata, hash, timestamp)
  Daily volume: 8.64B × 500 bytes = 4.32 TB/day (uncompressed)
  ClickHouse compression (10×): ~430 GB/day stored

Hot tier (90 days in ClickHouse):
  90 × 430 GB = ~38 TB in ClickHouse

Cold tier (7 years - 90 days in S3):
  (7×365 - 90) days × 430 GB/day ≈ 1 PB compressed S3 Parquet

Kafka buffer:
  100K events/sec × 500 bytes = 50 MB/sec inbound
  7-day Kafka retention: 50 MB/sec × 7 × 86400 = ~30 TB (before replication)
  RF=3: ~90 TB total Kafka storage

Hash chain overhead:
  Each entry adds 64 bytes (prior hash) + 64 bytes (own hash) = 128 bytes
  Overhead: 128 / 500 = ~26% overhead → acceptable for tamper detection
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Kafka (write buffer) | Decouples producers; 7-day replay; absorbs 100K events/sec bursts |
| ClickHouse (hot tier) | Columnar; append-only; 2s queries on 10B rows; partition by month |
| Hash Chain | Tamper detection; each entry cryptographically links to prior |
| S3 WORM (cold tier) | Legal immutability; 7-year retention; Object Lock |
| Tiered Storage | 90-day hot (ClickHouse) + cold (S3); 20× cost difference |
| Legal Hold | Flag exempts events from TTL; retained beyond standard policy |

---

## Data Model

```sql
-- ClickHouse audit log (hot tier — 90 days)
CREATE TABLE audit_events (
  event_id          UUID,
  event_time        DateTime64(3),          -- millisecond precision
  actor_type        LowCardinality(String), -- USER / SERVICE / SYSTEM
  actor_id          String,                 -- user_id, service name, or SYSTEM
  actor_ip          String,                 -- hashed: SHA-256(IP + daily_salt)
  action_type       LowCardinality(String), -- PAYMENT_CREATED / LOGIN / ADMIN_OVERRIDE / etc.
  resource_type     LowCardinality(String), -- PAYMENT / ACCOUNT / USER / CONFIG
  resource_id       String,                 -- payment_id, account_id, etc.
  outcome           LowCardinality(String), -- SUCCESS / FAILURE / DENIED
  metadata          String,                 -- JSON: additional context
  prev_event_hash   FixedString(64),        -- SHA-256 of previous event in chain
  event_hash        FixedString(64),        -- SHA-256(event data + prev_event_hash)
  sequence_num      UInt64,                 -- monotonically increasing; gaps indicate loss
  retention_class   LowCardinality(String), -- STANDARD / LEGAL_HOLD / SENSITIVE
  ingested_at       DateTime DEFAULT now()
) ENGINE = MergeTree()
  PARTITION BY toYYYYMM(event_time)          -- partition by month; drop entire months for TTL
  ORDER BY (actor_id, event_time)            -- primary sort: all actor events together
  TTL event_time + INTERVAL 90 DAY          -- auto-drop after 90 days (except LEGAL_HOLD)
  SETTINGS index_granularity = 8192;

-- PostgreSQL: metadata, legal holds, chain anchors
CREATE TABLE audit_chain_anchors (
  anchor_id      BIGSERIAL    PRIMARY KEY,
  chain_tip_hash CHAR(64)     NOT NULL,   -- hash of last event at anchor time
  sequence_num   BIGINT       NOT NULL,   -- last event's sequence number
  anchored_at    TIMESTAMP    DEFAULT NOW(),
  external_ref   TEXT                     -- Bitcoin txid, public ledger reference
);

CREATE TABLE legal_holds (
  hold_id        BIGSERIAL    PRIMARY KEY,
  resource_id    VARCHAR(128) NOT NULL,
  reason         TEXT         NOT NULL,
  requested_by   VARCHAR(64)  NOT NULL,
  created_at     TIMESTAMP    DEFAULT NOW(),
  expires_at     TIMESTAMP                 -- null = indefinite hold
);

CREATE TABLE audit_schemas (
  action_type    VARCHAR(64)  PRIMARY KEY,
  schema_version INTEGER,
  json_schema    JSONB,                    -- validate metadata field
  pii_fields     TEXT[],                   -- fields requiring masking for GDPR export
  retention_days INTEGER      DEFAULT 90
);
```

---

## API Design

```
Write API (called by all platform services — fire-and-forget):

POST /api/v1/audit/events
Body: {
  "actor_type": "USER",
  "actor_id": "u-12345",
  "action_type": "PAYMENT_CREATED",
  "resource_type": "PAYMENT",
  "resource_id": "pay-abc",
  "outcome": "SUCCESS",
  "metadata": {
    "amount": 50000,
    "currency": "INR",
    "merchant_id": "merch-xyz",
    "device_id": "d-abc",
    "session_id": "sess-xyz"
  }
}
Response 202: { "event_id": "evt-uuid", "accepted_at": "2024-01-01T14:00:00.123Z" }
(202 Accepted: event received; persistence is async)

Batch write (preferred for high-throughput services):
POST /api/v1/audit/events/batch
Body: { "events": [...up to 1000 events...] }
Response 202: { "accepted_count": 1000 }

Query API (compliance, investigation, reporting):
GET /api/v1/audit/events
Query params:
  actor_id=u-12345
  action_type=PAYMENT_CREATED,PAYMENT_FAILED
  resource_id=acct-abc
  from=2024-01-01T00:00:00Z
  to=2024-01-31T23:59:59Z
  limit=100 (default), offset=0
  include_pii=false (default; requires special permission)
Response 200: {
  "events": [...],
  "total": 4821,
  "next_offset": 100,
  "query_latency_ms": 1842
}

GET /api/v1/audit/events/{event_id}/verify
Response 200: {
  "event_id": "evt-uuid",
  "hash_valid": true,
  "chain_valid": true,   -- prev_event_hash matches previous event's hash
  "anchor_verified": true -- chain has been anchored to external ledger
}

POST /api/v1/audit/legal-hold
Body: { "resource_id": "acct-abc", "reason": "Regulatory investigation", "requested_by": "legal-team" }
Response 201: { "hold_id": "hold-123", "effect": "Events for acct-abc exempt from TTL" }

POST /api/v1/audit/reports/compliance
Body: { "actor_id": "u-12345", "from": "2024-01-01", "to": "2024-12-31", "format": "PDF" }
Response 202: { "report_id": "rpt-123" }
→ Async: generates report from ClickHouse + cold S3 → S3 presigned URL when ready
```

---

## High-Level Architecture

```
Producing Services (payment-service, auth-service, admin-service, ...)
  │
  ├── Emit audit event (fire-and-forget):
  │     auditClient.emit(event) → async Kafka producer
  │     → never blocks producing service
  │     → if Kafka buffer full: drop (rare) AND emit metric alert
  ▼
Kafka Topic: audit-events
  Partitions:  64 (partition by actor_id hash)
  Retention:   7 days (replay window for recovery)
  RF:          3 (no event loss on broker failure)
  Producer:    acks=1 (leader ack only; fire-and-forget)

Consumer Group 1: Audit Writer (hot path to ClickHouse)
  ├── Batch 10,000 events or 5 seconds (whichever first)
  ├── Hash chain computation:
  │     For each event in batch:
  │       event.sequence_num = atomic_counter.getAndIncrement()
  │       event.event_hash = SHA-256(event_data + prev_event_hash)
  │       prev_event_hash = event.event_hash
  ├── Bulk INSERT to ClickHouse
  ├── Commit Kafka offset AFTER successful ClickHouse insert
  └── Publish chain tip to Redis: SET audit:chain:tip {hash, sequence_num}

Consumer Group 2: Cold Archive Writer (to S3 WORM)
  ├── Accumulate 1-hour batches
  ├── Write as Parquet to S3: s3://audit-archive/{year}/{month}/{day}/{hour}.parquet
  ├── Apply S3 Object Lock (WORM): Retention = 7 years; mode = COMPLIANCE
  │   (COMPLIANCE mode: not even AWS can delete; overrides bucket owner)
  └── Write batch metadata to PostgreSQL (manifest for querying cold tier)

Audit Query Service:
  ├── Hot query (< 90 days): ClickHouse
  ├── Cold query (> 90 days): S3 Parquet via Athena (serverless query engine)
  ├── Cross-tier query: fan out to both; merge results
  └── Legal hold check: before any TTL deletion, check legal_holds table

Hash Chain Anchor Service (daily):
  ├── GET audit:chain:tip from Redis → current chain_tip_hash + sequence_num
  ├── Publish hash to Bitcoin blockchain (OP_RETURN) or public timestamp service
  ├── Record anchor in audit_chain_anchors table
  └── Verification: any auditor can verify chain integrity using public blockchain record
```

---

## Detailed Components

### Audit Event Emission SDK (Fire-and-Forget)
```java
// Embedded in every service; async; never blocks request thread
public class AuditClient {

    private final KafkaProducer<String, AuditEvent> producer;

    public void emit(AuditEvent event) {
        // Set event_id and timestamp server-side (not client-side)
        // to prevent event time manipulation
        event.setEventId(UUID.randomUUID().toString());
        event.setEventTime(Instant.now());

        ProducerRecord<String, AuditEvent> record =
            new ProducerRecord<>("audit-events",
                                 event.getActorId(),  // partition key: actor_id
                                 event);
        // sendAsync: fire-and-forget; callback for metrics only
        producer.send(record, (metadata, ex) -> {
            if (ex != null) {
                metrics.increment("audit.kafka.drop");
                // Log the event locally as fallback (disk buffer, not lost permanently)
                localFallbackBuffer.write(event);
            }
        });
        // Returns immediately; payment transaction continues
    }

    // Producer config (fire-and-forget for audit events):
    // acks=1       (leader ack; fastest; acceptable for audit events)
    // linger.ms=50 (batch for 50ms before sending; more efficient)
    // batch.size=32768 (32 KB batches)
    // compression.type=lz4 (fast compression; good ratio for JSON events)
    // NOTE: acks=1 (not acks=all) is acceptable because:
    //   - Kafka RF=3 means leader failure very rare
    //   - 7-day retention means we can replay if needed
    //   - The alternative (acks=all) adds 10–50ms latency to every audit emit
}
```

### Hash Chain Computation (Audit Writer)
```java
// Runs in Audit Writer consumer; computes chain before ClickHouse insert
public List<AuditEvent> computeHashChain(List<AuditEvent> batch, String prevHash) {
    String currentHash = prevHash;

    for (AuditEvent event : batch) {
        // Canonical serialization: deterministic field ordering (not JSON default)
        String canonicalData = toCanonicalJson(event);

        // Hash = SHA-256(event_data + prev_hash)
        String eventHash = sha256(canonicalData + currentHash);

        event.setPrevEventHash(currentHash);
        event.setEventHash(eventHash);

        currentHash = eventHash;
    }

    // Update chain tip in Redis (for next batch + anchor service)
    redis.set("audit:chain:tip", currentHash);
    redis.set("audit:chain:sequence", String.valueOf(batch.getLast().getSequenceNum()));

    return batch;
}

// Verification (any party can verify):
public boolean verifyEvent(AuditEvent event, String expectedPrevHash) {
    String canonicalData = toCanonicalJson(event);
    String recomputedHash = sha256(canonicalData + event.getPrevEventHash());
    boolean hashValid = recomputedHash.equals(event.getEventHash());
    boolean chainValid = event.getPrevEventHash().equals(expectedPrevHash);
    return hashValid && chainValid;
}

// Why canonical JSON? Regular JSON serialization is non-deterministic:
//   { "a": 1, "b": 2 } and { "b": 2, "a": 1 } are the same object
//   but SHA-256 of each string is DIFFERENT
// Canonical JSON: keys in lexicographic order; no extra whitespace
// Ensures: same event always produces same hash
```

### Tiered Storage and Querying
```
HOT TIER (ClickHouse, 90 days):
  Query: actor_id + time range → fast (sorted by actor_id, event_time)
  Retention: 30 → 90 days (ClickHouse TTL auto-drops partitions)
  Cost: ~$0.10/GB/month SSD; at 38 TB: ~$3,800/month

COLD TIER (S3 Parquet, 7 years):
  Files: hourly Parquet files, partitioned by year/month/day/hour
  S3 path: audit-archive/year=2024/month=01/day=01/hour=14/part-0001.parquet
  Hive partitioning: Athena can prune partitions for efficient query
  Query: AWS Athena (serverless SQL on S3 Parquet)
    SELECT * FROM audit_archive
    WHERE year='2024' AND month='01'        -- partition pruning: only scan Jan 2024
    AND actor_id = 'u-12345'
    Athena: scans only relevant Parquet files (partition pruning)
    Cost: $5/TB scanned; 30-day query on one user: ~10 GB scanned = $0.05
  Cost: S3 Standard-IA: $0.0125/GB/month; at 1 PB: ~$12,800/month

CROSS-TIER QUERY (< 90 days straddles hot/cold):
  Query service fans out:
    ClickHouse query for recent portion
    Athena query for older portion
  Merge results in application layer (both ordered by event_time)

LEGAL HOLD EFFECT ON TTL:
  ClickHouse TTL excludes events where retention_class = 'LEGAL_HOLD'
    TTL event_time + INTERVAL 90 DAY WHERE retention_class != 'LEGAL_HOLD'
  Cold tier: S3 Object Lock override not possible in COMPLIANCE mode
    Legal hold events: written to separate S3 prefix with indefinite Object Lock
```

### GDPR and PII Handling
```
Challenge: GDPR "right to erasure" vs audit log immutability.
These seem contradictory. Resolution:

Legal principle: audit logs for financial transactions are exempt from right to erasure
under EU GDPR Article 17(3)(b): "processing is necessary for compliance with a legal
obligation" AND Article 17(3)(e): "establishment, exercise or defence of legal claims."

For non-exempt PII (e.g., email address in metadata):
  Pseudonymization: store SHA-256(PII + pepper) instead of raw PII
  Encryption: encrypt PII field; "erasure" = delete encryption key
              encrypted blob becomes unreadable (cryptographic erasure)
  Separate PII store: store PII in separate deletable DB; audit log holds reference ID
    On erasure request: delete from PII store; audit log entry has null PII but intact chain

For compliance reports including PII:
  Reporter must have GDPR-justified purpose
  Report generation includes PII masking for unauthorized viewers
  Access to include_pii=true requires separate authorization
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K events/sec):
  Direct write to PostgreSQL audit_log table
  No Kafka; synchronous in-request-path write
  Works but: adds 10ms to every audited operation

Phase 2 (1K → 10K events/sec):
  Kafka buffer + async ClickHouse write
  Remove audit write from request path
  Batch inserts to ClickHouse (1,000 events per insert)

Phase 3 (10K → 100K events/sec):
  ClickHouse cluster (3+ nodes; sharded by actor_id)
  Hash chain computation in dedicated consumer
  S3 cold archive for events > 90 days

Phase 4 (100K+ events/sec):
  ClickHouse cluster with more shards
  Kafka partitions scaled to 128
  Separate ClickHouse clusters by event type (payment events vs auth events)

Phase 5 (multi-region):
  Regional audit services; each region handles local events
  Cross-region replication for global compliance reporting
  Central cold archive (S3) with cross-region replication
```

---

## Reliability Strategy
- **Kafka RF=3**: no events lost on single broker failure; 7-day replay window
- **Local fallback buffer**: if Kafka full, SDK writes to local disk; periodic flush to Kafka
- **ClickHouse replication**: RF=2; reads continue on primary failure
- **Cold archive**: S3 WORM; 99.999999999% durability; Object Lock prevents accidental deletion
- **Sequence gap detection**: if sequence_num jumps, alert (possible event loss between producer and consumer)
- **Chain verification**: nightly job replays entire chain; alerts on first broken link

---

## Security Considerations
- **Immutability**: no UPDATE or DELETE on audit_events table; ClickHouse and S3 WORM enforce this
- **Access control**: write API open to internal services only (VPC + mTLS); query API requires compliance role
- **PII in events**: raw IPs hashed; actor details pseudonymized per GDPR schema
- **Hash chain anchoring**: public ledger anchoring every 24h prevents retroactive chain rewriting
- **Audit log for audit service itself**: meta-audit — who queried the audit log and when
- **Kafka topic ACLs**: only audit-writer consumers can read; producers cannot consume (no self-replay attack)
- **S3 Object Lock COMPLIANCE mode**: not even account root can delete; protects against insider threats

---

## Observability
```
Ingest metrics:
  audit_events_ingested_total          (counter; per action_type)
  audit_kafka_producer_drop_rate       (alert if > 0; events lost before Kafka)
  kafka_consumer_lag_audit_writer      (alert if > 100K events; ~1s of events)
  clickhouse_insert_latency_p99        (target < 500ms for batch of 10K)
  hash_chain_computation_latency_ms    (should be < 100ms per batch)

Integrity metrics:
  chain_verification_result            (nightly; SUCCESS or BROKEN_AT_SEQ_N)
  sequence_gap_detected                (alert immediately; indicates event loss)
  chain_anchor_last_published          (alert if > 25 hours since last anchor)

Query metrics:
  audit_query_latency_p99              (target < 5s for 30-day query)
  audit_query_clickhouse_vs_athena     (hot vs cold tier query distribution)
  legal_hold_count                     (gauge; number of active legal holds)

Storage metrics:
  clickhouse_storage_gb                (alert at 80% of provisioned; add nodes)
  s3_worm_object_count                 (cold tier growth rate)
  cold_archive_write_latency           (alert if S3 writes failing)

Alerts:
  audit_kafka_producer_drop_rate > 0  → Immediate: events being lost
  kafka_consumer_lag > 100K           → Consumer falling behind; scale ClickHouse writes
  chain_verification_failed           → CRITICAL: audit log tampered or corrupted
  sequence_gap_detected               → CRITICAL: event loss between producer and consumer
  chain_anchor_last_published > 25h   → Anchoring service failure
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Kafka broker failure (1 of 3) | RF=3 absorbs; no impact | RF=3 standard; auto-rebalance |
| Audit Writer crashes | Consumer lag grows; Kafka buffers events | Restart consumer; replay from Kafka from last committed offset |
| ClickHouse node down | Query latency increases; inserts to remaining nodes | RF=2 replication; reads continue; alert |
| S3 Object Lock conflict | Cannot delete archived event (intended) | COMPLIANCE mode is intentional; legal team process for legitimate exceptions |
| Hash chain corruption detected | Critical integrity failure | Alert immediately; halt new writes; investigate; restore from Kafka replay |
| Kafka topic full (retention exceeded) | Events older than 7 days purged from Kafka replay | Accept: ClickHouse is authoritative after insert; Kafka is a temporary buffer |
| Producer service event drop | Event not emitted (SDK drops) | Local fallback buffer; periodic flush; sequence gap detection catches this |
| Legal hold missed (event deleted before hold) | Compliance risk | Legal hold must be created before event TTL; process enforcement |

---

## Chaos Testing
```
Experiment 1: Audit Writer Consumer Crash During Batch
  Inject:    Kill Audit Writer process mid-batch (during hash chain computation)
  Expected:  Kafka offset was NOT committed (commit happens after ClickHouse insert)
  Expected:  On restart: consumer replays from last committed offset
  Expected:  Duplicate events may be produced (at-least-once)
  Expected:  ClickHouse INSERT is idempotent for same event_id → dedup on replay
  Red flag:  Events lost (offset committed before ClickHouse insert)
  Red flag:  Hash chain broken after recovery (sequence numbers out of order)

Experiment 2: ClickHouse Node Failure During Insert
  Inject:    Kill ClickHouse primary node mid-batch insert
  Expected:  Insert fails; consumer does NOT commit Kafka offset
  Expected:  Consumer reconnects to ClickHouse replica (promoted to primary)
  Expected:  Retry batch insert on new primary
  Expected:  No event loss; possible duplicate (handled by dedup on event_id)
  Red flag:  Consumer commits offset before ClickHouse confirmation → event lost

Experiment 3: Hash Chain Tamper Detection
  Inject:    Manually UPDATE one audit event row in ClickHouse (change metadata field)
  Expected:  Nightly chain verification job detects hash mismatch at that event
  Expected:  Alert fires: "chain_verification_failed at sequence_num N"
  Expected:  External anchor proves chain state at last anchor is inconsistent with current
  Red flag:  Tampered event passes verification (hash chain not being verified)

Experiment 4: S3 Object Lock Protection Test
  Inject:    Attempt to delete a WORM-locked S3 object via AWS CLI (as account root)
  Expected:  S3 returns: "Object is locked and cannot be deleted"
  Expected:  Even bucket owner, IAM admin, and AWS support cannot delete
  Expected:  Object remains until Object Lock retention period expires
  Red flag:  Object deleted despite Object Lock (misconfigured lock mode)
  Note:      GOVERNANCE mode allows admin deletion; must use COMPLIANCE mode

Experiment 5: Legal Hold Effectiveness
  Inject:    Create legal hold on resource_id=acct-abc; wait for 90-day TTL to fire
  Expected:  ClickHouse TTL job skips events with retention_class='LEGAL_HOLD'
  Expected:  Events for acct-abc remain in ClickHouse after 90 days
  Expected:  Events without legal hold for same time period are correctly deleted
  Red flag:  Legal hold events deleted by TTL (WHERE clause not filtering correctly)
```

---

## Monthly Cost Estimate
```
Scale assumption: 100K events/sec, 90-day hot tier, 7-year cold tier

Kafka Cluster:
  3 × kafka.m5.2xlarge ($300/mo)               = $900/mo
  Storage: 90 TB × $0.10/GB                    = $9,000/mo (Kafka 7-day retention + RF3)
  Total Kafka:                                  = $9,900/mo

ClickHouse Cluster (hot tier):
  3 × r6g.4xlarge (128 GB RAM, $800/mo)        = $2,400/mo
  Storage: 38 TB SSD × $0.10/GB               = $3,800/mo
  Total ClickHouse:                            = $6,200/mo

S3 (cold tier, 7 years):
  Year 1: 430 GB/day × 365 days ≈ 157 TB × $0.0125/GB = $1,963/mo
  Year 7: ~1 PB × $0.0125/GB = $12,800/mo (growing)
  S3 Object Lock: no additional cost
  Year 1 cold tier:                            = $1,963/mo

Audit Writer instances:
  4 × m5.xlarge ($150/mo)                      = $600/mo

Query Service + Athena:
  Athena: $5/TB scanned; ~10 TB/month queries  = $50/mo
  2 × t3.large ($60/mo) for query API          = $120/mo

PostgreSQL (metadata + holds):
  db.t3.medium                                 = $70/mo

Monitoring + networking:
                                               = $500/mo
────────────────────────────────────────────────────────────
Total Year 1: ~$19,353/month

Cost drivers:
  Kafka storage (51%): 7-day retention × RF3 = largest cost
  ClickHouse (32%): compute + SSD storage

Optimisation levers:
  Kafka retention: reduce to 3 days (saves ~$4,500/mo)
    Risk: smaller replay window for recovery
  ClickHouse compression: current estimate assumes 10× compression; if lower, costs more
  S3 Intelligent-Tiering: auto-moves cold data to cheaper tier → saves ~20% on S3
  Kafka on Graviton2 + S3 Express One Zone for Kafka: ~15% cheaper
  Hot tier only for last 30 days; 30–90 days on ClickHouse-Glacier (cheaper storage class)

Realistic optimised cost:
  3-day Kafka + S3 Intelligent-Tiering + Graviton2: ~$12,000/month Year 1
```

---

## Migration Story

### Current State
Audit events written synchronously to PostgreSQL `audit_log` table. At 10K events/sec: DB CPU at 90%, p99 latency 80ms added to every audited operation. No tamper detection. No cold storage. Compliance team cannot run ad-hoc queries without DBA help.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Decouple audit write via Kafka (zero downtime)
  Action:
    Add async Kafka emit in SDK (alongside synchronous PostgreSQL write — dual write)
    Verify: Kafka consumer receives all events; count matches PostgreSQL count
    Once verified: make Kafka the primary write; PostgreSQL becomes backup
    After 1 week stable: remove synchronous PostgreSQL write from hot path
  Monitor:
    Kafka consumer lag (should be near-zero)
    DB CPU (should drop significantly once sync write removed)
    Event count parity: Kafka events vs old PostgreSQL events
  Rollback:
    Re-enable synchronous PostgreSQL write; disable Kafka path

Step 2: Add ClickHouse as write destination (zero downtime)
  Action:
    Deploy ClickHouse cluster; deploy Audit Writer consumer
    Consumer: reads from Kafka → batches → inserts to ClickHouse
    Dual-write: events go to both PostgreSQL (existing) and ClickHouse (new)
    Run ClickHouse for 30 days; verify query results match PostgreSQL
  Monitor:
    Event count parity between ClickHouse and PostgreSQL
    ClickHouse query latency (should be dramatically faster than PostgreSQL)
  Rollback:
    Stop ClickHouse consumer; PostgreSQL remains authoritative

Step 3: Add hash chain computation (zero downtime)
  Action:
    Audit Writer now computes hash chain on each batch before ClickHouse insert
    Backfill: compute hashes for existing ClickHouse events (batch job)
    Verify: chain verification job runs on backfilled data; confirms integrity
  Monitor:
    Hash chain verification result (nightly job)
    chain_anchor_last_published (once anchoring is deployed)
  Rollback:
    Disable hash chain computation; ClickHouse events have null hashes
    (Tamper detection lost; audit functionality preserved)

Step 4: Add S3 cold archive (zero downtime, additive)
  Action:
    Deploy Cold Archive Writer consumer
    Consumer: accumulates 1-hour batches → Parquet → S3 Object Lock
    Configure Athena for cold tier queries
    Test: verify cold tier query returns correct events for 3-month-old data
  Monitor:
    Cold archive write success rate
    S3 Object Lock applied correctly (verify via S3 GetObjectLegalHold)
  Rollback:
    Stop Cold Archive Writer; cold events not archived (risk: hot tier TTL eventually deletes)

Step 5: Drop PostgreSQL audit_log (planned, after compliance sign-off)
  Action:
    Compliance team signs off: ClickHouse + S3 meets all regulatory requirements
    Archive PostgreSQL audit_log → S3 (historical preservation)
    Drop PostgreSQL audit_log table; remove dual-write path
  Monitor:
    No compliance gaps after PostgreSQL removal
  Rollback:
    Restore PostgreSQL from archived S3; resume dual-write
    (Irreversible once dropped; get sign-off first)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → ClickHouse for hot tier (0–90 days): append-only; authoritative for recent events.
  → S3 WORM for cold tier (90+ days): immutable; authoritative for historical events.
  → Kafka: temporary buffer (7 days); NOT authoritative; events may replay.
  → PostgreSQL: metadata only (legal holds, chain anchors, schemas).

□ Consistency model?
  → Write: fire-and-forget to Kafka; at-least-once delivery to ClickHouse.
  → Duplicate events possible (consumer crash + replay); deduplicated on event_id.
  → Hash chain: eventual consistency (computed by consumer after DB write, not at emit time).
  → Cold archive: eventual (archived within 1 hour of event_time; S3 is always slightly behind).

□ Failure mode?
  → Kafka consumer crash → replay from last committed offset; at-least-once; dedup by event_id.
  → ClickHouse node down → replication absorbs; queries slower; inserts continue.
  → S3 outage → cold archive writes queue in consumer; hot tier unaffected.
  → Hash chain broken → CRITICAL alert; halt investigation; restore from Kafka replay.

□ Hot partitions?
  → High-frequency actor (service account making millions of events) → hot ClickHouse partition.
  → ClickHouse ORDER BY (actor_id, event_time): hot actor → sequential scans on same part.
  → Mitigation: ClickHouse handles high write rate per partition well (MergeTree is write-optimized).

□ Cost driver?
  → Kafka storage (51%): 7-day retention × RF3 is expensive at 100K events/sec.
  → Optimise: reduce retention to 3 days (saves ~$4,500/mo); accept smaller replay window.

□ Security risk?
  → Audit log itself targeted by insider (DBA deletes inconvenient events).
    Mitigation: ClickHouse has no DELETE for MergeTree tables (TTL only); WORM for archive.
  → Hash chain key compromised.
    Mitigation: external public ledger anchor (Bitcoin OP_RETURN) doesn't require key trust.
  → Compliance reporter reads PII they shouldn't.
    Mitigation: include_pii=true requires separate auth; default response masks PII fields.

□ Migration path?
  → Synchronous PostgreSQL → Kafka async → ClickHouse dual-write → Hash chain
    → S3 cold archive → Drop PostgreSQL.
  → Compliance sign-off required before final step.
```

---

## Staff Engineer Discussion Points

**"How do you handle 'right to erasure' (GDPR) for an immutable audit log?"**
The apparent conflict resolves legally and technically. Legally: financial transaction audit logs are exempt from erasure under GDPR Article 17(3)(b) and (e) — compliance obligations and legal claims take precedence. Technically for non-exempt PII: use cryptographic erasure — encrypt the PII field with a per-user key; to "erase," delete the key. The encrypted blob remains in the audit log (chain integrity preserved) but is unreadable (functionally erased). A separate PII store holds the actual values linked by reference ID; deleting from the PII store is straightforward. The audit entry remains intact; only the sensitive values become inaccessible.

**"What's the difference between your audit log and an event sourcing log?"**
Event sourcing log: the authoritative state store for a domain aggregate. You reconstruct current state by replaying events. It's a first-class persistence mechanism — you query it to get current state. Mutation is the norm: you're constantly writing new events to advance state. Audit log: a record of what happened, for compliance and investigation. You never reconstruct state from it. It's secondary evidence, not primary storage. The audit log is append-only by policy; event sourcing is append-only by design (events are immutable facts). If you conflate them, you end up with an event sourcing store that has compliance retention requirements (7 years of domain events) — expensive and complex.

**"How do you ensure no events are silently dropped between emit and ClickHouse?"**
Sequence numbers. Each event gets a monotonically increasing sequence number assigned by the Audit Writer consumer (not the emitting service). A gap detection job runs every minute: SELECT sequence_num FROM audit_events ORDER BY sequence_num → detect gaps. A gap means an event was in Kafka but the consumer crashed before committing the offset, or an event was dropped at the Kafka producer. Also: the local fallback buffer in the SDK catches SDK-level drops. The two monitors together (sequence gaps + Kafka producer drop metric) detect event loss at both boundaries.

**"How would you design this to support real-time security alerting?"**
Add a third Kafka consumer group: the Security Consumer. This consumer reads the same audit-events topic in real-time and evaluates streaming CEP (Complex Event Processing) rules: "three failed login attempts in 60 seconds from same IP → alert," "admin action outside business hours → alert," "payment amount > $10,000 → alert." Flink is the right engine for stateful streaming rules with time windows. This consumer is entirely separate from the Audit Writer — adding real-time alerting doesn't touch the audit log storage path. The audit log is evidence; the security alerting is action.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Cannot modify or delete log entries"              → Append-Only (ClickHouse MergeTree; S3 WORM)
  "Prove log was not tampered"                       → Hash Chain + external anchor
  "100K events/sec; never slow down producers"       → Kafka fire-and-forget
  "Query 10B events in < 5 seconds"                  → ClickHouse columnar
  "7-year retention at reasonable cost"              → Tiered storage (ClickHouse → S3 Parquet)
  "Legal hold: exempt from TTL"                      → retention_class field + TTL WHERE clause

IMMUTABILITY GUARANTEES BY TIER:
  In-flight (Kafka):     No immutability; consumer has not persisted yet
  Hot (ClickHouse):      No UPDATE/DELETE in MergeTree; TTL only (policy-based)
  Cold (S3 WORM):        S3 Object Lock COMPLIANCE mode; even root cannot delete
  Chain integrity:       SHA-256 hash chain + public ledger anchor (cryptographic)

HASH CHAIN MECHANICS:
  entry_n.hash = SHA-256(canonical_json(entry_n) + entry_(n-1).hash)
  Alter entry_k → entry_k.hash changes → entry_(k+1).hash invalid → chain breaks
  External anchor (Bitcoin OP_RETURN every 24h): proves chain state at that timestamp
  Attacker cannot retroactively recompute chain without being detected

PATTERNS REJECTED (and why):
  Synchronous write in request path → adds 10–50ms to every payment; must decouple
  PostgreSQL for raw events → GROUP BY 10B rows takes hours; columnar required
  Mutable log ("allow corrections") → corrections ARE audit events; never modify existing
  Single tier forever → 7-year hot storage at $6,200/mo; cold tier at $1,963/mo = 3× savings

SCC LENS:
  STATE:
    In-flight: Kafka (7-day buffer; replayable; not authoritative)
    Hot: ClickHouse (authoritative; 90 days; append-only; hash chain)
    Cold: S3 WORM Parquet (authoritative; 7 years; cryptographically immutable)
    Meta: PostgreSQL (legal holds, chain anchors, schemas — not audit events themselves)

  COORDINATION:
    Sequence numbers: Audit Writer assigns; gap detection verifies no events lost
    Hash chain: Audit Writer serializes event hash computation (cannot parallelize across batches)
    Legal hold check: before any TTL deletion, check legal_holds table (coordination point)
    Chain anchoring: daily job publishes to external ledger (timestamp the chain state)

  CONCENTRATION:
    Kafka: all 100K events/sec funnel through it → must be sized correctly (64 partitions)
    Audit Writer: single consumer group serializes hash chain → parallelization limited
    Mitigation: shard by actor_id → multiple independent chains (shard_id + sequence within shard)

IAM PLATFORM CONNECTION:
  Every IAM event is an audit event:
    "User u-12345 authenticated via OIDC from device d-abc" → AUTHENTICATION event
    "OAuth token issued to client-xyz for scopes payments:write" → TOKEN_ISSUED event
    "Admin overrode session expiry for user u-12345" → ADMIN_OVERRIDE event
    "OPA policy denied access to payment resource" → ACCESS_DENIED event
  PCI-DSS requires: all access to cardholder data must be auditable
  ForgeRock AM: all authentication and authorization events → audit log
  OPA decision log → audit event (who requested, what policy decided, what context)

INTERVIEW ANSWER TRIGGER:
  "Design an audit logging service" →
    1. Append-only: no UPDATE or DELETE ever; hash chain for tamper detection
    2. Decouple via Kafka: producers fire-and-forget; audit log never slows payment path
    3. ClickHouse: columnar for fast aggregation queries on billions of events
    4. Tiered storage: 90-day ClickHouse (hot) → 7-year S3 WORM (cold)
    5. Compliance: hash chain + external anchor; GDPR cryptographic erasure
    6. SCC: State=ClickHouse+S3+Kafka; Coordination=sequence gaps+hash chain;
            Concentration=Kafka 64 partitions; Audit Writer serializes hash computation
    7. IAM tie-in: every auth event, token issuance, policy decision → audit event
```

---

*Part 1 Complete: Case Studies 01–10 (Beginner Track)*
*Next: Part 2 — Intermediate (Case Studies 11–20)*
