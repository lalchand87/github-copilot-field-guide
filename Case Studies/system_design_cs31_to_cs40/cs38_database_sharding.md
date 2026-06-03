# Case Study 38 — Database Sharding and Partitioning

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Shard key selection, Cross-shard queries, Resharding complexity.

---

## Business Context

PostgreSQL can handle enormous workloads on a single server (hundreds of GB, thousands of TPS). But a payment platform processing 100B transactions/year eventually outgrows a single machine. Sharding distributes data across multiple database instances, each owning a subset of the data. The shard key determines which rows live where — and the wrong shard key choice is catastrophic (hot shards, cross-shard queries, impossible resharding).

**Business goals driving architecture:**
- Write throughput beyond single PostgreSQL limits (1M+ inserts/day)
- Horizontal scale: add shards as data grows
- Tenant isolation: enterprise customers on dedicated shards
- Cross-shard queries minimised (ideally zero in hot path)

---

## Recognition Framework

### Sharding vs Partitioning

```
Partitioning (single-DB):
  Data split across multiple tables or files WITHIN the same PostgreSQL instance
  PostgreSQL native: PARTITION BY RANGE(date) or PARTITION BY HASH(merchant_id)
  Benefits: query pruning (only scan relevant partition), maintenance (drop old partition)
  Limitation: still one DB server; no horizontal scale for writes
  
  Use when: data is large but write throughput fits in one server
             (e.g., 1TB payments table → partition by month → each partition 80GB)

Sharding (multi-DB):
  Data split across multiple SEPARATE database servers
  Application layer routes queries to correct shard
  Benefits: horizontal write scale; each shard handles fraction of total writes
  Complexity: cross-shard queries, resharding, distributed transactions
  
  Use when: single-server throughput is insufficient OR need tenant isolation

Hybrid (most real systems):
  Partitioning WITHIN each shard (best of both):
    payments table partitioned by month within each shard
    Query: shard router → correct shard → partition pruning within shard
```

### Shard Key Selection (Most Critical Decision)

```
Payment platform shard key options:

Option A: merchant_id (GOOD)
  All payments from merchant X go to shard K
  Queries: "all payments for merchant X" → single shard (efficient)
  Tenant isolation: put enterprise merchants on dedicated shards
  Problem: hot merchant (Amazon, Flipkart) gets 80% of traffic → hot shard
  Mitigation: hot merchant splitting (Amazon → multiple virtual shards)

Option B: payment_id (BAD for queries)
  Random distribution across shards
  Write distribution: even (good for write throughput)
  Queries: "all payments for user X" → fan-out to ALL shards (expensive)
  Use only when: writes dominate; no aggregation queries needed

Option C: user_id (MIXED)
  All payments from user X on same shard
  User-centric queries efficient
  Problem: power users (corporates) → hot shard
  Problem: cross-user queries (merchant totals) → fan-out

Option D: date (BAD for writes)
  All payments on 2024-01-15 go to shard K
  Problem: all writes today → one shard → 100% hot shard
  Good for: read-heavy analytics (each date's data on one shard)
  NEVER for current writes

CHOSEN for payments: merchant_id
  Rationale: most queries are merchant-scoped; tenant isolation is clean
  Hot merchant problem: solved by merchant_id hashing + shard weighting
  Cross-merchant analytics: handled by separate analytics DB (Case 18)
```

---

## Problem Statement

Shard payment platform's payments database by merchant_id. Consistent hashing for even distribution. Dedicated shards for enterprise merchants. Transparent shard routing in application.

---

## Shard Architecture

```
Shard configuration (example: 4 shards):
  Shard 0: merchant_ids whose hash % 4 == 0
  Shard 1: merchant_ids whose hash % 4 == 1
  Shard 2: merchant_ids whose hash % 4 == 2
  Shard 3: dedicated to enterprise merchant "Amazon" (too large for shared shard)

Shard registry (stored in control plane DB):
  merchant_id  → shard_id
  amazon_corp  → shard-3 (dedicated)
  flipkart     → shard-3 (dedicated)
  all others   → hash(merchant_id) % num_shards

Connection pool per shard:
  HikariCP: separate connection pool per shard (not one pool for all shards)
  Each pool: 10–50 connections to that shard's PostgreSQL
```

### Consistent Hashing for Shard Assignment
```java
public class ShardRouter {
    
    // Consistent hash ring: 150 virtual nodes per shard (reduces hot spots on resharding)
    private final ConsistentHashRing<Integer> ring;
    private final Map<String, Integer> dedicatedMerchants;  // from control plane DB
    private final Map<Integer, DataSource> shardDataSources;
    
    public DataSource getShardFor(String merchantId) {
        // Check dedicated shard first (enterprise merchants)
        if (dedicatedMerchants.containsKey(merchantId)) {
            return shardDataSources.get(dedicatedMerchants.get(merchantId));
        }
        
        // Consistent hash for regular merchants
        int shardId = ring.locate(merchantId);
        return shardDataSources.get(shardId);
    }
    
    public DataSource getShardForPayment(UUID paymentId) {
        // Can't route by payment_id alone (don't know merchant)
        // Payment table must include merchant_id: SELECT merchant_id FROM payments WHERE id=?
        // OR: encode shard_id in payment_id prefix
        String merchantId = extractMerchantFromPaymentId(paymentId);  // prefix encoding
        return getShardFor(merchantId);
    }
}
```

### Payment ID with Shard Encoding
```
Problem: given payment_id = "pay-abc-123", which shard is it on?
  Cannot route without merchant_id; need to look up or store it

Solution: encode shard information in payment_id
  Format: {shard_id_base62}-{timestamp_base62}-{random_base62}
  Example: "2A-K9XM-7R8Q" (shard 2A → shard 2)
  
  Benefits:
    Any service can extract shard from payment_id without DB lookup
    Globally unique (shard + timestamp + random)
    Human-readable prefix identifies shard for debugging
    
  Implementation:
    Payment created: ShardIdEncoder.encode(shardId) + "-" + timestampBase62 + "-" + randomBase62
    Routing: ShardIdEncoder.decode(paymentId.split("-")[0]) → shardId

Alternative: store shard mapping in Redis
  redis.set("pay-abc-123:shard", "2");  // set when payment created
  TTL: 90 days (hot period)
  Cold: look up from payment table (meta-lookup DB with all payment IDs)
```

---

## PostgreSQL Native Partitioning (Within Each Shard)

```sql
-- Within each shard: partition payments by month
-- (Query pruning: range queries by date only scan relevant partitions)

CREATE TABLE payments (
    payment_id    VARCHAR(32)  NOT NULL,
    merchant_id   VARCHAR(64)  NOT NULL,
    amount        BIGINT,
    status        VARCHAR(32),
    created_at    TIMESTAMPTZ  NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (automated by pg_partman or script)
CREATE TABLE payments_2024_01 PARTITION OF payments
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE payments_2024_02 PARTITION OF payments
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- etc.

-- Benefits:
-- 1. Query: WHERE created_at >= '2024-01-01' → scans only payments_2024_01
-- 2. Archival: DROP TABLE payments_2022_01 (drops old data in milliseconds)
-- 3. Maintenance: VACUUM payments_2024_01 (one partition at a time)
-- 4. Index per partition: smaller indexes = faster queries

-- pg_partman: automates partition creation for future months
-- SELECT partman.create_parent('public.payments', 'created_at', 'native', 'monthly');
```

---

## Cross-Shard Query Patterns

```
Query types and how to handle:

1. Single-merchant query: "all payments for merchant X"
   → Route to merchant X's shard → single shard query → fast
   This is 90% of queries; sharding by merchant_id optimises for this

2. Cross-merchant query: "total platform payments today"
   → Fan-out to ALL shards → merge results in application → slow
   These should NOT be in the payment hot path
   Solution: analytics DB (Case 18) handles these; not the transactional DB

3. Payment lookup by payment_id: "get payment pay-abc"
   → Decode shard from payment_id prefix → single shard query → fast
   (This is why shard encoding in payment_id matters)

4. User's payment history: "all payments for user U"
   → User payments span multiple merchants → multiple shards potentially
   Solution A: maintain user→merchant mapping; fan-out only to relevant shards
   Solution B: denormalize to user shard (different sharding dimension)
   Solution C: separate read replica with full data for user queries
   
   For payment platforms: option A (user transacts with few merchants → 1-3 shards)
```

---

## Resharding (Adding Shards)

```
Problem: 4 shards → 8 shards (need more capacity)
Simple hash (% 4 → % 8): half the data must move to different shards
  Moving data: writes to both old and new location → inconsistency window
  Downtime required for cut-over

Consistent hashing advantage:
  Adding a shard: only the data "adjacent" on the ring moves
  4 → 5 shards: ~20% of data moves (not 50%)
  Still requires data movement; just less of it

Practical resharding process:
  Step 1: Add new shard to ring (receive no traffic yet)
  Step 2: Background migration: copy data from old shard → new shard
  Step 3: Dual-write: new writes go to BOTH old and new shard
  Step 4: Read from new shard; verify data integrity
  Step 5: Stop writing to old shard; cut over completely
  Step 6: Clean up old shard data
  Duration: hours to days depending on data volume
  Downtime: near-zero with dual-write pattern

Avoid resharding with:
  Dedicated enterprise shards (isolate high-volume merchants → never need to move them)
  Over-sharding initially: 16 shards from day 1 (even if only 2 are active)
    Assign virtual shard IDs 0-15; physically map 4 virtual → 1 physical initially
    When scaling: split physical shard → 2 physical shards (no virtual → physical remapping)
```

---

## Observability
```
Per-shard metrics:
  shard_query_latency_p99 by {shard_id}        (detect hot shard)
  shard_write_throughput_tps by {shard_id}     (load distribution; alert on imbalance)
  shard_storage_bytes by {shard_id}            (capacity planning)
  cross_shard_query_count                      (should be near-zero in hot path)
  shard_rebalance_progress                     (during resharding)
  connection_pool_active by {shard_id}         (pool exhaustion detection)
```

---

## Architecture Review Checklist
```
□ Shard key must:
  → Evenly distribute writes (no hot shard)
  → Enable single-shard queries for most access patterns
  → Be immutable (cannot change merchant's shard after payment is created)
  → Handle hot keys (enterprise merchants) with dedicated shards

□ Never shard by:
  → Current date (all writes go to today's shard → 100% hot)
  → Sequential ID (hot shard = highest shard always)

□ Cross-shard queries:
  → Designed out of hot path (analytics uses separate DB)
  → Acceptable in: reporting, admin tools, batch jobs (not in payment critical path)
  → Never in: checkout, payment authorisation, balance check

□ Resharding plan exists before first shard:
  → Consistent hashing from day 1
  → Virtual shard IDs (allows physical split without remapping)
  → Dual-write pattern documented

□ SCC LENS:
  STATE: each shard is an independent PostgreSQL (authoritative for its keyspace)
  COORDINATION: shard router (application layer; consistent hash), shard registry (control plane)
  CONCENTRATION: hot merchant = hot shard → dedicated shard for high-volume merchants

□ vs Citus (PostgreSQL distributed):
  Citus: managed distributed PostgreSQL; handles sharding transparently
  Self-managed sharding: more control; lower abstraction cost
  For payments: Citus is excellent; reduces custom code significantly
  Trade-off: Citus version upgrade constraints; harder to customize per-shard config
```

---

*Next: Case Study 39 — Event Streaming vs Message Queuing*
