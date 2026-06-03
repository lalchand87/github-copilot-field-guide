# Case Study 37 — Caching Strategies (Advanced)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Cache invalidation strategies, Write-through vs write-behind, Stampede prevention.

---

## Business Context

Caching is the single most powerful performance optimisation in distributed systems — and the single most common source of subtle bugs. "There are only two hard things in computer science: cache invalidation and naming things." For a payment platform, stale account balances cause overselling; stale fraud signals miss fraud; stale VPA cache blocks payments. Getting caching right is getting correctness and performance simultaneously.

**Business goals driving architecture:**
- Balance reads in checkout: < 10ms (currently goes to DB: 50ms)
- Merchant config reads: thousands/sec; rarely changes; should be sub-millisecond
- VPA resolution: < 5ms (today: 200ms per NPCI call)
- Never serve stale balance data for payment authorisation
- Cache invalidation that doesn't cause thundering herd on DB

---

## Recognition Framework

### Cache Strategies (When to Use Which)

```
Cache-Aside (Lazy Loading):
  Read: check cache → miss → read DB → populate cache → return
  Write: write DB → (do NOT write cache; let it expire or invalidate)
  
  Pros: only cache what's actually read; no cache pollution
  Cons: cache miss = 2× latency (cache + DB); thundering herd on cold start
  Use for: most cases; default choice; read-heavy with acceptable miss penalty

Write-Through:
  Write: write cache → write DB (both synchronously)
  Read: check cache → almost always hit (just wrote it)
  
  Pros: cache always fresh; no stale reads
  Cons: write penalty (must write cache AND DB); unused writes waste cache memory
  Use for: data that's written and immediately read; session data; user preferences

Write-Behind (Write-Back):
  Write: write cache only → async write to DB (in background)
  Read: from cache (always fresh if recently written)
  
  Pros: write latency = cache write only (fast); DB writes batched
  Cons: data loss if cache fails before async write to DB; consistency risk
  Use for: analytics counters; view counts; non-critical writes; NEVER for payments

Read-Through:
  Same as Cache-Aside but cache library handles DB read on miss
  Application talks only to cache; cache calls DB on miss
  Use for: when cache library supports it (Redis with Lua scripts)

Refresh-Ahead (Proactive):
  Background job refreshes cache before TTL expires
  Application always hits cache (no cold start)
  Use for: expensive-to-compute data; predictable access patterns
```

---

## Problem Statement

Advanced caching layer for a payment platform. Multiple data types with different freshness requirements. Stampede prevention. Cache invalidation on write. Multi-level caching.

---

## Caching Tiers per Data Type

```
Data Type               | Strategy        | TTL      | Why
------------------------|-----------------|----------|------------------------------------------
Account balance         | Cache-Aside     | 5 seconds| Freshness critical; short TTL tolerable
Merchant config         | Write-Through   | 1 hour   | Rarely changes; must be consistent
VPA → bank mapping      | Cache-Aside     | 1 hour   | NPCI call is expensive (200ms)
JWT JWKS keys           | Refresh-Ahead   | 5 min    | Key rotation is rare; never miss
User fraud features     | Write-Behind    | 10 min   | Velocity counters; DB write can lag
Card network status     | Refresh-Ahead   | 30 sec   | External status; stale is risky
Exchange rates          | Refresh-Ahead   | 1 min    | External; predictable access
Payment history page    | Cache-Aside     | 1 min    | DB query expensive; slight staleness OK
```

---

## Multi-Level Cache (L1 + L2)

```
Problem: Redis (L2 cache) is fast (1ms) but not fast enough for 50K TPS × many lookups
Solution: in-process L1 cache (< 0.1ms) + Redis L2 (< 1ms) + DB (50ms)

                    L1 (in-process)    L2 (Redis)    L3 (DB)
Merchant config:    1000ms TTL         1h TTL        authoritative
JWT JWKS:           5min TTL           5min TTL      OIDC provider
VPA resolution:     10min TTL          1h TTL        NPCI API

Cache hierarchy hit path:
  Request → Check L1 (in-process map) → hit: < 0.1ms → return
  Request → Check L1 → miss → check L2 (Redis) → hit: < 1ms → populate L1 → return
  Request → Check L1 → miss → check L2 → miss → call DB/API → populate L2 → populate L1 → return
```

```java
// Caffeine (L1 in-process) + Redis (L2) cache implementation
@Component
public class MerchantConfigCache {
    
    // L1: in-process, Caffeine (fast, bounded, eviction-aware)
    private final Cache<String, MerchantConfig> l1Cache = Caffeine.newBuilder()
        .maximumSize(10_000)           // evict LRU when > 10K entries
        .expireAfterWrite(1, TimeUnit.MINUTES)  // shorter than L2 to force refresh
        .recordStats()                 // for monitoring hit rate
        .build();
    
    @Autowired
    private RedisTemplate<String, MerchantConfig> redis;
    
    @Autowired
    private MerchantConfigRepository repo;
    
    public MerchantConfig get(String merchantId) {
        // L1 hit
        MerchantConfig config = l1Cache.getIfPresent(merchantId);
        if (config != null) return config;
        
        // L2 hit
        String redisKey = "merchant:config:" + merchantId;
        config = redis.opsForValue().get(redisKey);
        if (config != null) {
            l1Cache.put(merchantId, config);  // populate L1 from L2
            return config;
        }
        
        // L3 miss: DB query
        config = repo.findById(merchantId).orElseThrow();
        redis.opsForValue().set(redisKey, config, Duration.ofHours(1));  // populate L2
        l1Cache.put(merchantId, config);  // populate L1
        return config;
    }
    
    public void invalidate(String merchantId) {
        l1Cache.invalidate(merchantId);
        redis.delete("merchant:config:" + merchantId);
        // DB update happened before this call (write-through to DB first; then invalidate cache)
    }
}
```

---

## Cache Invalidation Patterns

### Write-Through (Cache + DB in Same Transaction)
```java
@Transactional
public void updateMerchantConfig(String merchantId, MerchantConfigUpdate update) {
    // 1. Write to DB
    merchantRepo.update(merchantId, update);
    
    // 2. Write to cache (same transaction context)
    MerchantConfig updated = merchantRepo.findById(merchantId).orElseThrow();
    redis.opsForValue().set("merchant:config:" + merchantId, updated, Duration.ofHours(1));
    
    // 3. Invalidate all L1 caches across all pods (local only is not enough)
    // Use Redis Pub/Sub to broadcast invalidation
    redis.convertAndSend("cache-invalidations", 
        new CacheInvalidation("merchant:config", merchantId));
    
    // Each pod subscribes to this channel and evicts from its L1 cache
}
```

### Cache-Aside with Stampede Prevention (Probabilistic Early Expiry)
```java
// Problem: TTL expires → 1000 concurrent requests all miss → 1000 DB queries
// Solution: probabilistic early expiry (some requests refresh before expiry)

public Balance getBalance(String accountId) {
    String key = "balance:" + accountId;
    CachedValue<Balance> cached = redis.get(key, CachedValue.class);
    
    if (cached != null) {
        // Probabilistic early refresh: as expiry approaches, randomly refresh
        double remainingFraction = cached.remainingTTL() / cached.originalTTL();
        double refreshProbability = Math.exp(-BETA * Math.log(remainingFraction));
        //   BETA = 1.0: standard; higher = more aggressive early refresh
        //   When 50% TTL remaining: low probability; when 5% remaining: high probability
        
        if (Math.random() < refreshProbability) {
            // Refresh this entry (only this request does it; others still hit cache)
            // Background refresh: don't block current request
            CompletableFuture.runAsync(() -> refreshBalance(accountId));
        }
        return cached.getValue();
    }
    
    // True cache miss: use lock to prevent stampede
    return refreshBalance(accountId);
}

private Balance refreshBalance(String accountId) {
    String lockKey = "lock:balance:" + accountId;
    // SET NX: only one request fetches from DB
    if (redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(5))) {
        try {
            Balance balance = ledgerService.getBalance(accountId);
            redis.set("balance:" + accountId, 
                      new CachedValue<>(balance, Duration.ofSeconds(5)), 
                      Duration.ofSeconds(5));
            return balance;
        } finally {
            redis.delete(lockKey);
        }
    } else {
        // Another request has the lock; wait briefly and retry from cache
        Thread.sleep(50);
        return getBalance(accountId);  // will hit cache populated by lock holder
    }
}
```

### Write-Behind for Fraud Feature Store
```java
// Fraud velocity counters: write-behind is acceptable (small inconsistency window)
// DB is not in the hot path; Redis is the authoritative source for fraud signals

@Component
public class FraudFeatureStore {
    
    private final RedisTemplate<String, String> redis;
    
    // Track in-flight writes to avoid duplicates on crash recovery
    private final Queue<FeatureUpdate> pendingWrites = new ConcurrentLinkedQueue<>();
    
    public void recordTransaction(String userId, long amount, String merchantCategory) {
        // Write to Redis immediately (fraud engine reads from here)
        String key = "fraud:user:" + userId;
        redis.opsForHash().increment(key, "txn_count_5min", 1);
        redis.opsForHash().increment(key, "total_amount_5min", amount);
        redis.expire(key, Duration.ofMinutes(10));
        
        // Queue async write to PostgreSQL (for historical analysis + retraining)
        pendingWrites.add(new FeatureUpdate(userId, amount, merchantCategory));
    }
    
    @Scheduled(fixedRate = 1000)  // flush every second
    public void flushToDB() {
        List<FeatureUpdate> batch = drainQueue(pendingWrites, 1000);
        if (!batch.isEmpty()) {
            // Batch upsert: count/amount per user per minute bucket
            featureRepo.batchUpsert(batch);
        }
    }
    
    // Recovery: on startup, if pendingWrites queue has entries (crashed without flushing)
    // → replay from Redis to DB (Redis persists via AOF; safe)
}
```

---

## Cache Consistency for Balance (Critical)

```
Account balance: must be fresh at payment authorisation time.
  Stale balance → authorise payment → insufficient funds → payment fails at capture → bad UX
  OR: stale balance → authorise insufficient funds → overdraft → financial loss

Strategy: short TTL (5 seconds) + cache miss is acceptable
  For payment authorisation: BYPASS CACHE (read directly from DB)
  Cache is ONLY for: balance display in app (informational; 5-second staleness OK)
  
  Code:
    GET /balance (display to user) → read from cache (5s TTL) → fast
    POST /payments/authorise       → read from DB (bypass cache) → accurate
  
  This is the key insight: different use cases need different consistency levels
    UI balance display: eventual consistency (5s old) is fine
    Payment authorisation: strong consistency (always from DB) is required
  
  Never use cache for financial decisions; use cache for user-facing display only
```

---

## Observability
```
Cache metrics:
  cache_hit_rate by {cache_name}          (target > 90% for hot data; alert if < 70%)
  cache_miss_rate by {cache_name}
  cache_stampede_events_total             (lock contention; indicates TTL expiry storm)
  l1_eviction_rate                        (L1 cache size too small; increase maximumSize)
  redis_memory_usage_bytes                (alert if > 80% of maxmemory)
  cache_invalidation_lag_ms              (how long between DB write and cache invalidation)
  stale_reads_detected                   (if you have a way to detect: validation check)

Alerts:
  cache_hit_rate < 70%: DB under unusual load (verify DB latency hasn't spiked)
  redis_memory > 80%:   reduce TTLs or increase Redis cluster size
  stampede_events > 10/min: TTL too short for traffic pattern; increase TTL or add jitter
```

---

## Architecture Review Checklist
```
□ Different data types get different strategies:
  → Balance (critical): cache-aside + short TTL + bypass cache for authorisation
  → Config (stable): write-through; long TTL; L1 in-process
  → Velocity counters (write-heavy): write-behind; Redis as primary store

□ Stampede prevention:
  → Probabilistic early expiry OR mutex lock (SET NX) on cache miss
  → Never let N requests all hit DB simultaneously on TTL expiry

□ Cache invalidation is consistent:
  → Write-through: cache updated in same transaction as DB write
  → Cache-aside: invalidate on write; never write-without-invalidate
  → L1 invalidation: Redis Pub/Sub broadcast to all pods

□ Cache coherence across pods:
  → L1 caches are per-pod; stale L1 after DB update
  → Redis Pub/Sub invalidation channel: all pods subscribe; evict L1 on message
  → TTL on L1 as safety net: even without explicit invalidation, L1 expires eventually

□ Financial correctness:
  → Account balance for AUTHORISATION: NEVER from cache (always DB read)
  → Account balance for DISPLAY: cache OK (5s staleness acceptable)
  → Ledger writes: NEVER write-behind (must be synchronous; financial integrity)

□ SCC LENS:
  STATE: DB (authoritative), Redis L2 (shared cache), in-process L1 (pod-local cache)
  COORDINATION: cache invalidation via Redis Pub/Sub; TTL as safety net
  CONCENTRATION: Redis L2 is the shared hot path for all pods → cluster + eviction policy
```

---

*Next: Case Study 38 — Database Sharding and Partitioning*
