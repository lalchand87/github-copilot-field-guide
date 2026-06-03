# Case Study 06 — Distributed Cache

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Invalidation Strategy Selection, Consistent Hashing, Thundering Herd.

---

## Business Context

A distributed cache exists to solve one problem: your database cannot serve your read load at the required latency, and cache RAM is cheaper than database replicas. The hidden cost is operational complexity: cache invalidation, thundering herd, cache poisoning, and the implicit consistency contract you are making with every user who reads cached data.

The cache is never the source of truth. The database is. Cache loss is always acceptable. **If you design your system so that cache loss causes data corruption, you have a bug — not a cache problem.**

**Business goals driving architecture:**
- Reduce DB read load by > 90% — primary metric for cache success
- Sub-millisecond read latency for cacheable data
- Cache miss must never cause a user-visible error — fall through to DB gracefully
- Multi-tenant: services share cache infrastructure but must not collide on keys
- Staleness tolerance is per-data-type — define explicitly, not implicitly

---

## Recognition Framework

### Signals from the Problem
```
- DB read load exceeds capacity at required latency
- Most reads are for the same hot records (Pareto: 20% of records drive 80% of reads)
- Records are small enough to hold in memory (< 1 MB per record)
- Stale reads tolerable for seconds to minutes (not zero tolerance)
- Multiple stateless app servers need shared cache state
- Adding DB replicas is more expensive than cache nodes (RAM vs disk I/O)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Consistent Hashing | Scale cluster without mass key remapping on node add/remove |
| Replication (per shard) | HA — primary failure must not cause cache shard to be unavailable |
| Cache-Aside (Lazy Load) | Standard read pattern; simple; no write-path coupling |
| LRU Eviction | Cold keys fall out naturally; hot keys stay; appropriate default |
| TTL on all keys | Safety net — even correct data must eventually expire |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Simple modulo sharding (key % N) | Adding 1 node remaps ~50% of keys → mass DB stampede on rebalance |
| Write-through by default | Adds latency to every write; unnecessary if writes are infrequent |
| Write-behind (write-back) by default | Risk of data loss if cache node crashes before persisting to DB |
| LFU eviction by default | More complex; marginal benefit over LRU unless access is highly skewed |
| No TTL on any key | Without TTL: stale data lives forever after DB update; silent corruption |
| CDC-driven invalidation for all data | Overkill for most cases; adds Debezium + Kafka + consumer to ops burden |

### Invalidation Strategy Selection — The Hardest Problem in Caching

```
Strategy 1: TTL Only (simplest)
  How:    Every SET includes TTL; key auto-expires; no explicit invalidation
  Stale window: up to TTL duration
  Use when: staleness for TTL duration is acceptable
  Example: user profile (1h TTL fine), product catalog (5min TTL fine)
  Avoid:  financial balances, inventory counts, auth sessions
  Ops cost: zero (TTL is automatic)

Strategy 2: Cache-Aside with TTL (lazy population)
  How:    On miss: read DB → write cache with TTL
          On DB write: do NOT update cache; wait for TTL to expire
  Stale window: up to TTL duration after DB write
  Use when: read >> write; stale window acceptable
  Problem: thundering herd on TTL expiry of popular key
  Ops cost: low

Strategy 3: Write-Through (zero stale window)
  How:    On DB write → also write cache immediately (same transaction or dual-write)
  Stale window: zero (cache always matches DB)
  Use when: reads >> writes; cache reads must always be fresh
  Problem: adds latency to writes; write failure = cache inconsistency
  Example: session store (Case 07), auth tokens
  Ops cost: medium (write path complexity)

Strategy 4: Event-Driven Invalidation (CDC)
  How:    DB change → Debezium captures WAL event → Kafka → cache consumer → DEL key
  Stale window: ~100ms (event propagation time)
  Use when: near-zero stale tolerance AND write-through latency is unacceptable
  Example: financial balance, inventory count
  Ops cost: high (Debezium + Kafka + consumer + monitoring)
  Use only when: staleness causes real business harm (overcharge, oversell)

Strategy 5: Key-Based Versioning (cache-busting)
  How:    Cache key includes a version: user:123:v5 (v5 = latest version in DB)
          On DB write: increment version in DB; new reads use new key (miss → warm)
          Old key expires naturally via TTL
  Stale window: zero (new key = new data; old key = old data; never mix)
  Use when: you need fresh reads but write-through latency is unacceptable
  Example: product details with frequent updates
  Ops cost: medium (version management in DB)

Decision matrix:
  Staleness > 1 min ok?       → TTL only
  Staleness 1–60s ok?         → Cache-aside with short TTL
  Zero stale tolerance?       → Write-through OR Event-driven (CDC)
  Financial / inventory data? → Event-driven (CDC) — real business harm if stale
  Auth sessions?              → Write-through (from Case 07)
```

---

## Problem Statement

Distributed in-memory cache layer for applications. Supports GET/SET/DELETE with TTL. Handles cache invalidation. Scales to a cluster of 10 nodes (320 GB total capacity).

---

## Functional Requirements
- GET key → value (null if missing or expired)
- SET key value TTL
- DELETE key (explicit invalidation)
- Multi-type values: String, Hash, List, Set
- TTL-based automatic expiration
- LRU eviction when memory is full
- Consistent distribution across N nodes (minimal remapping on node change)

## Non-Functional Requirements
- Latency: **< 1ms p99** for GET/SET
- Availability: **99.99%** (cache node failure must not take down application)
- Consistency: **eventual** — stale reads tolerable (per-key TTL defines staleness window)
- Memory: 32 GB per node; 10 nodes = 320 GB total
- Scale: ~1 billion keys across cluster

---

## Capacity Estimation
```
Cluster capacity:
  10 nodes × 32 GB = 320 GB
  Average key+value: 320 bytes → ~1 billion keys (before replication)

Throughput:
  Redis single node: 100K–500K ops/sec
  10 shards × 100K ops/sec = 1M+ ops/sec cluster-wide

Replication topology:
  Each shard: 1 primary + 2 replicas
  10 shards × 3 nodes = 30 physical nodes total

Memory accounting per node:
  Data: 32 GB
  Redis overhead: ~200 bytes per key (metadata, expiry, hash table)
  At 100M keys per shard: 100M × 200 bytes = 20 GB overhead → leaves 12 GB for values
  Practical: size nodes for 60–70% of expected key count (leave room for overhead)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Consistent Hashing | Minimal key remapping on topology change |
| Per-Shard Replication | HA: primary failure → replica promoted; shard stays available |
| Cache-Aside | Standard application-level caching; no write-path coupling |
| LRU Eviction | Cold keys evicted; hot keys survive memory pressure |
| TTL on all keys | Safety net; mandatory for every key; no exceptions |

---

## Data Model

```
Redis Cluster slot assignment:
  16,384 hash slots total
  HASH_SLOT = CRC16(key) mod 16384
  Shard 0: slots 0–1637    (primary + 2 replicas)
  Shard 1: slots 1638–3276 (primary + 2 replicas)
  ...
  Shard 9: slots 14746–16383 (primary + 2 replicas)

Application key design (mandatory conventions):
  Pattern:   {service}:{type}:{id}
  Examples:
    payments:account:acct-123          → account data, TTL 5min
    iam:session:sess-abc               → session data, TTL 15min (Case 07)
    catalog:product:prod-456           → product details, TTL 30min
    rate:client-xyz:1712345600         → rate limit counter, TTL 120s (Case 03)

Key namespacing rule:
  EVERY key must start with {service}: prefix
  Reason: prevents cross-service key collision in shared cluster
  Enforcement: validate in client library; reject keys without prefix

Multi-key operations:
  MGET: client library routes each key to correct shard; parallelises fetches
  Transactions: only within single shard (Redis MULTI/EXEC requires same slot)
  Cross-shard atomic ops: not supported — design to avoid or use DB instead
```

---

## API Design

```java
// Application-facing client library interface (hides cluster topology)

// Basic operations
String value = cache.get("iam:session:sess-abc");
// Returns: value string or null if missing/expired

cache.set("iam:session:sess-abc", sessionJson, Duration.ofMinutes(15));
// TTL is mandatory — library enforces this

cache.delete("iam:session:sess-abc");
// Explicit invalidation — synchronous; returns after DEL confirmed

// Bulk operations (client library parallelises across shards)
Map<String, String> values = cache.mget(List.of("k1", "k2", "k3"));
// Returns: map of key → value (null for missing keys)

// Hash operations (sub-field access; avoids fetching entire object)
cache.hset("payments:account:acct-123", "balance", "10000");
String balance = cache.hget("payments:account:acct-123", "balance");
// More efficient than getting entire account JSON for single field access

// Atomic increment (for counters — e.g. rate limiter from Case 03)
long newValue = cache.incr("rate:client-xyz:1712345600");
cache.expire("rate:client-xyz:1712345600", Duration.ofSeconds(120));
```

---

## High-Level Architecture

```
Application Instances (payment-service, user-service, etc.)
  │
  ▼
Cache Client Library (embedded in each application)
  ├── Cluster topology map (fetched from any node; refreshed every 1s)
  ├── CRC16 hash slot calculator → routes each key to correct shard
  ├── Connection pool per shard (10 connections per shard per app instance)
  ├── MOVED/ASK redirect handling (transparent on topology change)
  └── Circuit breaker per shard (fail gracefully on shard unavailability)

Cache Cluster (30 nodes across 3 AZs)
  ├── Shard 0: Primary (AZ-A) + Replica (AZ-B) + Replica (AZ-C)
  ├── Shard 1: Primary (AZ-B) + Replica (AZ-C) + Replica (AZ-A)
  ├── ...
  └── Shard 9: Primary (AZ-C) + Replica (AZ-A) + Replica (AZ-B)

Write path:  Key → client hashes to shard → primary node only
Read path:   Key → client hashes to shard → primary (strong) or replica (eventual)
Failover:    Primary fails → Redis Cluster promotes replica in < 5s automatically
             Client receives MOVED redirect → reconnects to new primary transparently

Application DB (PostgreSQL / DynamoDB)
  └── Source of truth; always consulted on cache miss
```

---

## Detailed Components

### Consistent Hashing — Why It Matters

```
Problem with simple modulo hashing (key % N nodes):
  10 nodes → add 1 node → 11 nodes
  Every key: hash(key) % 10 → hash(key) % 11
  For most keys: remainder changes → key maps to different node
  Result: ~91% of keys map to a different node on 10→11 node expansion
  Impact: 91% cache miss rate → 91% of traffic hits DB → DB overload

Redis Cluster uses hash slots (16,384 slots, not consistent hashing ring):
  HASH_SLOT = CRC16(key) mod 16384
  Slots are assigned to shards; shards are assigned to nodes
  On node addition: move only some slots (and their keys) to the new node
  Result: ~1/N of keys remap (for N→N+1 nodes: ~9% remap on 10→11 expansion)

Classic consistent hashing (ring-based):
  Place nodes on a ring (hash space 0 to 2^32)
  Each key maps to nearest clockwise node on ring
  On node addition: only keys between new node and its predecessor remap
  Virtual nodes (vnodes): 150 virtual positions per physical node
    Improves load distribution; avoids ring hotspots from uneven placement

Redis Cluster's slot-based approach achieves the same goal with simpler implementation.
```

### Thundering Herd — Detection and Prevention

```
Problem:
  Popular key "catalog:product:prod-456" expires (TTL reached)
  1,000 app instances simultaneously query cache → all get MISS
  All 1,000 instances simultaneously query PostgreSQL
  PostgreSQL: 1,000 unexpected concurrent queries → CPU spike → timeout → cascade

Detection:
  Monitor: sudden spike in cache miss rate + DB query rate at the same time
  Alert:   db_queries_per_sec > 2× baseline while cache_hit_rate drops suddenly

Prevention strategies (choose based on data type):

  1. Staggered TTL (easiest; no code change)
     Instead of: cache.set(key, value, Duration.ofMinutes(30))
     Use:        cache.set(key, value, Duration.ofMinutes(30).plus(randomSeconds(300)))
     Effect:     Keys expire at different times; stampede spreads over 5-minute window
     Best for:   High-volume read keys (product catalog, user profiles)

  2. Background Refresh (proactive; no miss window)
     Track key creation time in cache; when TTL is 90% elapsed, trigger background refresh
     Thread fetches fresh data from DB and updates cache before expiry
     Effect:     Key never actually expires for active users
     Best for:   Top-N hot keys that are almost always read
     Risk:       Background thread pool must be sized correctly; don't refresh every key

  3. Probabilistic Early Expiry (simple; language-agnostic)
     p(refresh) = exp(-delta × beta × log(ttl_remaining))
     As TTL approaches 0: probability of early refresh increases
     Effect:     Some requests trigger early refresh; rest see current value
     Best for:   Medium-traffic keys; no coordination needed

  4. Mutex / Distributed Lock (correct; adds complexity)
     On cache miss:
       Try to acquire lock: SET lock:{key} 1 NX EX 30
       If acquired: fetch from DB, populate cache, release lock
       If not acquired: wait briefly (10ms) and retry from cache
     Effect:     Only 1 request hits DB regardless of concurrency
     Best for:   Low-tolerance for DB spike; financial data; expensive DB queries
     Risk:       Lock acquisition failure handling; lock expiry timing

Recommendation:
  Use staggered TTL as default for all keys.
  Add mutex lock for keys with > 1K concurrent readers AND expensive DB queries.
  Background refresh only for the top-50 most-read keys (manually identified).
```

### Eviction Policies — Choosing the Right One

```
Redis eviction policies (set via maxmemory-policy):

allkeys-lru:
  Evict any key using LRU algorithm
  Use when: all keys are cacheable; OK to evict anything when memory full
  Best for: general-purpose caching

volatile-lru:
  Evict only keys WITH TTL set, using LRU
  Keys without TTL are never evicted (even if memory full)
  Use when: some keys must NEVER be evicted (e.g. rate limit counters with TTL)
  Risk: if all keys have TTL, same as allkeys-lru; if no keys have TTL, NO eviction → OOM

allkeys-lfu:
  Evict least frequently used key
  Use when: access pattern is highly skewed (celebrity problem)
  Example: 1K product keys but 1 product is viewed 10,000× more than others
  LFU keeps that hot key; LRU might evict it if it wasn't accessed recently

noeviction:
  Return error on SET when memory full; never evict
  Use when: data must never be lost (e.g. rate limit counters — losing a counter means
            allowing a burst that should have been blocked)
  Use with maxmemory set to prevent OOM kill

Recommendation per use case:
  General caching (product, user profile): allkeys-lru
  Rate limit counters:                     noeviction (counters must never disappear)
  Session store:                           volatile-lru (sessions have TTL; configs don't)
  Mixed workload:                          volatile-lru with noeviction fallback
```

---

## Scaling Strategy
```
Phase 1 (0 → 10K ops/sec):
  Single Redis node (standalone; no replication)
  All data on one node; simple; fine for development and small prod

Phase 2 (10K → 100K ops/sec):
  Redis Sentinel: 1 primary + 2 replicas + 3 sentinel nodes
  HA: automatic failover if primary dies
  Reads can go to replicas (eventual consistency) for read-heavy workloads

Phase 3 (100K → 1M ops/sec):
  Redis Cluster: 6–10 shards; horizontal scaling
  Client library handles routing (CRC16 hash slot)
  In-memory data split across shards; more RAM available

Phase 4 (> 1M ops/sec or > 320 GB):
  Larger nodes (64 GB or 128 GB per shard)
  OR more shards (16–32 shards)
  Local in-process cache (Caffeine) as L1 before Redis as L2
    Top-N hot keys cached in process → eliminates Redis network RTT entirely
    L1 hit rate for hot keys: ~99%; reduces Redis ops by 50–80%

Phase 5 (multi-region):
  Regional Redis clusters (one per region)
  Accept cross-region cache inconsistency (different regions may have different cache state)
  DB is always consistent; cache is regional approximation
  Async DB replication feeds regional caches
```

---

## Reliability Strategy
- **Shard replication**: 1 primary + 2 replicas per shard; automatic failover in < 5s
- **Application resilience**: client library catches Redis errors; falls through to DB
- **Cold start**: after full cache loss (restart), DB handles all reads; cache warms naturally over minutes
- **No persistence by default**: cache is ephemeral; persistence (RDB/AOF) adds write overhead; enable only if warm-up time matters
- **Memory monitoring**: alert at 80% memory usage; add nodes before hitting 100% (eviction starts)

---

## Security Considerations
- **Network isolation**: Redis nodes in private subnet; only app servers (by security group) can reach port 6379
- **AUTH**: Redis `requirepass` or ACLs (Redis 6+) with command-level permissions
- **TLS**: Redis 6+ supports TLS in-transit; at-rest encryption via OS-level disk encryption
- **Key namespace isolation**: `{service}:` prefix mandatory; client library enforces
- **No sensitive data unencrypted**: if caching JWTs, payment tokens, or PII — encrypt value before SET; decrypt after GET
- **ACL example**:
  `ACL SETUSER payment-service on >password ~payments:* +GET +SET +DEL +INCR +EXPIRE`
  payment-service can only access keys starting with `payments:`; cannot FLUSHALL

---

## Observability
```
Per-node metrics (emit from Redis Exporter):
  redis_memory_used_bytes           (alert if > 80% of maxmemory)
  redis_evicted_keys_total          (high eviction = undersized cache)
  redis_keyspace_hits_total         → cache_hit_rate = hits / (hits + misses)
  redis_keyspace_misses_total
  redis_connected_clients           (alert if approaching maxclients)
  redis_replication_lag_bytes       (replica lag; alert if > 1 MB)
  redis_instantaneous_ops_per_sec   (ops/sec per node)

Application-level metrics:
  cache_hit_rate by key_prefix      (target: > 90%; alert: < 75%)
  cache_miss_db_fallback_latency    (latency when DB fallback triggered)
  cache_set_success_rate            (alert if < 100%; Redis accepting writes)
  cache_eviction_rate               (alert if > 0% consistently)
  thundering_herd_detected          (spike in miss rate + DB query rate simultaneously)

Dashboards:
  - Cache hit rate by service and key type
  - Memory usage per shard (with eviction rate overlay)
  - Replication lag per shard
  - Cache miss DB fallback latency percentiles
  - Top 20 most-accessed keys (detect hotspots)

Alerts:
  cache_hit_rate < 75%            → cache undersized or TTL too short
  redis_evicted_keys_total > 1%   → add nodes or increase memory
  redis_primary_down              → Cluster/Sentinel auto-failover + page on-call
  replication_lag > 1 MB          → replica falling behind; investigate
  redis_memory_used > 80%         → provision additional nodes
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Primary node fails | ~5s downtime for that shard's writes | Redis Cluster auto-promotes replica; client reconnects |
| All replicas for a shard fail | That shard primary handles reads; no HA for that shard | Alert; provision replacement replica; accept single-node risk temporarily |
| Network partition (split brain) | Majority partition stays primary; minority becomes read-only | Redis Cluster majority-vote prevents split brain |
| Memory full (no eviction headroom) | SET operations fail (if noeviction) or keys evicted (if LRU) | Alert at 80%; provision nodes before 100% |
| Thundering herd | DB spike on hot key expiry | Staggered TTL + mutex lock for high-traffic keys |
| Cache poisoning (wrong value stored) | Applications read stale/wrong data | Short TTL bounds damage duration; explicit invalidation on known-bad write |
| Redis version upgrade | Brief unavailability during rolling upgrade | Rolling upgrade: one node at a time; replicas first, then primaries |

---

## Chaos Testing
```
Experiment 1: Kill Primary Node for One Shard
  Inject:    Forcefully terminate one primary Redis node
  Expected:  Redis Cluster promotes a replica within 5s
  Expected:  Client receives MOVED error → reconnects to new primary transparently
  Expected:  ~5s elevated error rate for that shard's keys; then recovery
  Expected:  App falls back to DB during the 5s window; no data loss
  Red flag:  Failover takes > 30s (Sentinel/Cluster misconfigured)
  Red flag:  Data loss after failover (replication lag was too large at time of failure)

Experiment 2: Fill Cache to 100% Memory (eviction test)
  Inject:    Load cache past maxmemory threshold
  Expected:  LRU eviction kicks in; oldest/coldest keys removed
  Expected:  Cache hit rate drops briefly as evicted keys miss
  Expected:  App falls back to DB for evicted keys and re-populates cache
  Expected:  No application errors; service degrades gracefully
  Red flag:  OOM kill (eviction not configured; noeviction policy)
  Red flag:  Application returns errors instead of falling back to DB

Experiment 3: Thundering Herd Simulation
  Inject:    Expire a hot key (EXPIREAT immediately); fire 1,000 concurrent requests
  With staggered TTL:  Expiries spread; no simultaneous miss possible
  Without mitigation:  1,000 DB queries simultaneously
  Expected:  < 5 DB queries hit for any single key expiry (mutex or stagger)
  Red flag:  DB query rate spikes to 1,000× normal on a single key expiry

Experiment 4: Cluster Topology Change (node addition)
  Inject:    Add a new shard (2 new nodes) to the cluster; trigger slot rebalancing
  Expected:  Redis migrates ~10% of slots to new shard (cluster rebalance)
  Expected:  During migration: MOVED errors handled transparently by client
  Expected:  Cache hit rate drops slightly during rebalance (migrated keys miss temporarily)
  Expected:  No application errors; fallback to DB for any cache miss
  Red flag:  Mass cache miss causes DB overload during rebalance
  Mitigation: Rebalance gradually (migrate 1 slot at a time); do during low-traffic window

Experiment 5: Redis AUTH Credential Rotation
  Inject:    Rotate Redis password; update secrets manager
  Expected:  New connections use new password (from secrets manager refresh)
  Expected:  Existing connections continue until naturally recycled
  Expected:  Zero authentication errors if client library handles AUTH correctly
  Red flag:  All Redis connections fail on password rotation (no graceful transition)
```

---

## Monthly Cost Estimate
```
Scale assumption: 10 shards × 3 nodes = 30 nodes × 32 GB RAM

Redis Cluster (ElastiCache for Redis):
  30 × cache.r6g.xlarge (32 GB, $260/mo)        = $7,800/mo

Networking (cross-AZ replication traffic):
  3 AZs; replication data: ~$200/mo

Monitoring (Redis Exporter + CloudWatch):
                                                  = $150/mo
────────────────────────────────────────────────────────────
Total: ~$8,150/month for 320 GB cache cluster

Context (cost justification):
  Equivalent PostgreSQL read replicas (10 × db.r6g.2xlarge): ~$4,000/mo
  BUT: PostgreSQL read replica can serve ~5K QPS at ~5ms latency
  Redis Cluster serves ~1M ops/sec at ~0.5ms latency
  These are different capabilities; cache is not a replacement for read replicas

Optimisation levers:
  Local in-process Caffeine cache (L1 before Redis):
    Top-10K hot keys cached in-process → 50-80% Redis ops reduction
    Redis Cluster can be scaled down to 5 shards → saves ~$3,900/mo
    Net: Caffeine is free; Redis savings ~$3,900/mo

  Reserved instances:
    1-year reservation: ~40% discount on ElastiCache
    Saves: ~$3,120/mo (at full cluster)

  Graviton2 nodes (cache.r7g.xlarge):
    ~15% better price/performance vs r6g
    Saves: ~$1,170/mo

Realistic optimised cost:
  5 shards (with L1 cache) × reserved × Graviton2: ~$2,600/mo
```

---

## Migration Story

### Current State
All reads go directly to PostgreSQL. At 10K RPS, DB CPU is 70% and p99 latency is 50ms. No cache layer at all.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Deploy Redis standalone + cache-aside (zero downtime)
  Action:
    Deploy Redis single node (not HA yet; validate before committing to HA)
    Add cache-aside to read path: check Redis first; on miss: query DB + SET cache
    Add TTL to every SET (start with conservative TTL: 5 min for all data types)
    Deploy to 1 service first (e.g. user-service); validate hit rate and latency
  Monitor:
    Redis hit rate (expect 0% on day 1; climbs to 70%+ within hours as cache warms)
    DB CPU (should drop as hit rate climbs)
    Application latency (should drop from ~50ms to < 5ms for cache hits)
  Rollback:
    Remove Redis check from read path; re-deploy
    PostgreSQL handles all reads (back to pre-migration state)

Step 2: Add explicit cache invalidation on writes (zero downtime)
  Action:
    On every DB write: DEL the corresponding cache key(s)
    This prevents stale reads after write (was relying purely on TTL before)
    Test: write a record → verify cache key is deleted → verify next read is fresh
  Monitor:
    Cache hit rate should stay the same or improve (explicit DEL is more correct than TTL-only)
    Stale read rate (monitor via spot checks: compare cache value vs DB value)
  Rollback:
    Remove explicit DEL calls; fall back to TTL-only invalidation

Step 3: Add TTL tuning per data type (zero downtime)
  Action:
    Review which data types have strict freshness requirements
    sessions: 15-min TTL, write-through (from Case 07)
    product_catalog: 30-min TTL, TTL-only (eventual is fine)
    account_balance: 30-second TTL or CDC-driven invalidation
    Implement per-type TTL in client library
  Monitor:
    Stale reads per data type (track "cache value ≠ DB value" events)
    DB query rate per data type (validate TTL changes have expected effect)
  Rollback:
    Revert to uniform TTL (5 min); re-deploy

Step 4: Upgrade to Redis Sentinel for HA (5-min maintenance window)
  Action:
    Deploy Sentinel setup (1 primary + 2 replicas + 3 sentinel nodes)
    Update client connection to Sentinel endpoint
    Failover test: kill primary → verify < 5s failover; app continues
  Monitor:
    Redis primary availability
    Replication lag (should be < 10ms)
  Rollback:
    Update connection string back to standalone Redis (still running)

Step 5: Upgrade to Redis Cluster for scale (rolling, zero downtime)
  Action:
    Deploy Redis Cluster alongside Sentinel
    Warm Cluster: start with 10% of read traffic; cache-aside warms naturally
    Gradually shift to 100% (10% → 25% → 50% → 100%)
    Decommission Sentinel after full migration
  Monitor:
    Cluster slot routing correctness
    MOVED error handling in client (transparent to application)
    Cache hit rate during migration (expect temporary dip as new cluster warms)
  Rollback:
    Shift traffic back to Sentinel; decommission Cluster
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL (or primary DB) is always authoritative.
  → Redis is a cache. Cache loss is always acceptable — app falls back to DB.
  → If Redis and DB disagree on any value: DB wins; Redis is wrong.

□ Consistency model?
  → Cache-aside with TTL: eventual (up to TTL duration stale after DB write).
  → Write-through: strong (cache always matches DB after write).
  → CDC-driven: near-real-time (~100ms after DB commit).
  → Choose per data type; never apply one model to all data.

□ Failure mode?
  → Primary shard fails → auto-failover in < 5s; app falls back to DB during window.
  → All Redis unavailable → app falls back to DB for all reads; latency degrades.
  → Cache full → LRU eviction; hit rate drops; DB absorbs miss traffic.
  → In all cases: application must never throw an uncaught exception on cache failure.

□ Hot partitions?
  → Hot key on one shard (celebrity problem).
  → Mitigation: local Caffeine L1 cache for top-N keys (eliminates Redis RTT).
  → OR: key sharding (cache key:shard:{0..N}; app reads all N; takes first non-null).

□ Cost driver?
  → RAM cost ($8,150/mo for 320 GB unoptimised).
  → L1 local cache reduces Redis ops by 80% → allows 50% fewer shards → ~$4,000/mo savings.

□ Security risk?
  → Cross-service key collision. Mitigation: mandatory service: key prefix enforced by client library.
  → PII or financial data in cache unencrypted. Mitigation: encrypt value before SET; decrypt after GET.
  → Unlimited key creation (cache flooding attack). Mitigation: key format validation in client library.

□ Operational burden?
  → Redis Cluster management: slot rebalancing when adding nodes.
  → Eviction monitoring: if eviction rate > 0%, cache is undersized.
  → TTL hygiene: audit periodically; stale forever keys are silent bugs.
  → Thundering herd: implement staggered TTL as default; mutex for hot keys.

□ Migration path?
  → No cache → Cache-aside (1 service) → Explicit invalidation → TTL tuning
  → Sentinel (HA) → Cluster (scale).
  → Each step independently rollbackable.
```

---

## Staff Engineer Discussion Points

**"Cache is returning wrong (stale) data — what do you do?"**
First: define what "wrong" means for this data type. For product price: 5 minutes stale is acceptable. For account balance: 30 seconds stale might already be too long. The answer is: define the staleness contract explicitly for every data type before you start caching. If you discover the contract was violated, the fix is to shorten TTL or switch to write-through/CDC for that type. Then add monitoring: periodically sample 1% of cache reads and compare to DB value to detect staleness violations proactively.

**"Cache penetration — key that never exists in DB?"**
Classic attack: send requests for `user:999999999999` (user that doesn't exist). On every request: Redis miss → DB query (finds nothing) → Redis not populated → next request: same miss → DB again. At scale: attacker can flood DB with queries for non-existent keys. Fix: cache the null result with a short TTL (e.g. `SET user:999999999999 "NULL" EX 60`). Next request hits Redis, gets "NULL", returns 404 without touching DB. Add rate limiting on non-existent key misses as a secondary defence.

**"What's the difference between a cache and a session store?"**
A cache is best-effort, expendable, and holds copies of data that exists in a DB. Cache loss → application falls back to DB → no data loss. A session store is authoritative for session state. If Redis is the session store (Case 07), session loss means users are logged out — that's user-visible data loss, not just latency degradation. Treat session stores with the same durability requirements as a primary DB. Different reliability requirements, different failure modes, different HA topologies.

**"How would you design a multi-tenant cache?"**
Option A: Key namespacing (shared cluster, key prefix = tenant ID).
  `tenant-123:product:prod-456` vs `tenant-456:product:prod-456`
  Cost: cheaper (shared infrastructure). Risk: noisy neighbour (one tenant's hot keys crowd out another's).

Option B: Dedicated Redis instance per tenant.
  Full isolation: tenant-123 gets its own Redis; tenant-456 gets another.
  Cost: expensive at scale (100 tenants = 100 Redis clusters). Justified for enterprise tier.

Option C: Dedicated shard per tenant (Redis Cluster slot assignment).
  Assign a range of hash slots to each tenant's keys.
  Moderate isolation without full cluster per tenant.
  Suitable for medium-scale multi-tenancy.

Recommendation: Start with A (namespacing); move high-value tenants to B (dedicated) when noisy neighbour becomes a documented problem.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "DB can't handle read load at required latency"     → Distributed Cache (Redis)
  "Add nodes without mass key remapping"              → Consistent Hashing / Redis Cluster slots
  "HA: primary node failure must not lose shard"      → Per-shard replication + auto-failover
  "Stale reads after DB write"                        → Write-through / CDC invalidation / shorter TTL
  "1,000 concurrent misses on hot key expiry"         → Thundering Herd → staggered TTL + mutex

INVALIDATION DECISION TREE:
  Staleness > 1 min acceptable?   → TTL only (simplest)
  Staleness 1–60s acceptable?     → Cache-aside + short TTL
  Near-zero stale required?       → Write-through (low-write data) OR CDC (any write rate)
  Financial / inventory data?     → CDC (real business harm if stale)
  Auth sessions?                  → Write-through (Case 07)

EVICTION POLICY SELECTION:
  General caching?                → allkeys-lru
  Rate limit counters (must not lose)? → noeviction + explicit TTL
  Mixed: some critical, some not? → volatile-lru (only evict keys with TTL set)

PATTERNS REJECTED (and why):
  Modulo sharding     → 50% key remapping on node add; DB stampede
  Write-behind default → data loss on cache crash before DB persist
  No TTL on any key   → stale data lives forever; silent correctness bug
  CDC for all data    → high ops cost; only justified for near-zero stale tolerance

SCC LENS:
  STATE:
    DB: source of truth for all data; never loses data
    Redis: ephemeral copy of DB state; loss always acceptable
    L1 (Caffeine): process-local ephemeral copy; eliminates Redis RTT for hot keys
    Three-tier state: L1 (< 0.1ms) → L2 Redis (< 1ms) → DB (< 10ms)

  COORDINATION:
    Invalidation is the hardest problem: who invalidates, when, how
    Cache-aside: no coordination on write (TTL handles staleness eventually)
    Write-through: write path must update both DB and cache atomically (or accept inconsistency window)
    CDC: DB is the coordinator; Debezium captures every change; cache listens
    Thundering herd: mutex coordinates "who fetches from DB on miss"

  CONCENTRATION:
    Hot key on one shard → concentration of reads on one Redis node
    Mitigation: L1 local cache eliminates the Redis RTT for hot keys entirely
    Redis Cluster: 16,384 slots across N shards; relatively even distribution
    Noisy tenant: one tenant's hot keys evict other tenants' data (shared cluster)

INTERVIEW ANSWER TRIGGER:
  "Design a distributed cache" →
    1. Spot signals: DB read overload, hot records, latency requirement
    2. Consistent hashing (not modulo) for cluster topology
    3. Invalidation strategy: define per data type (not one-size-fits-all)
    4. Thundering herd: staggered TTL + mutex for hot keys
    5. Eviction: allkeys-lru for general; noeviction for counters
    6. SCC: State=Redis (copy), DB (truth); Coordination=invalidation strategy;
            Concentration=hot key → L1 local cache
```

---

*Next: Case Study 07 — Session Store*
