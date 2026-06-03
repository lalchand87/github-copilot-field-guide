# Case Study 03 — Rate Limiter

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Algorithm Comparison, Recognition Framework, Chaos Testing.

---

## Business Context

A rate limiter is infrastructure, not a product feature. Its business purpose is threefold: protect backend services from overload, enforce fair usage in a multi-tenant API, and detect and block abuse. For a payment platform, rate limiting also satisfies PCI-DSS requirements around brute-force protection on authentication endpoints.

The failure mode that hurts the business is not "rate limiter too aggressive" — it is "rate limiter takes down the payment API." **API availability always wins over rate limit accuracy.**

**Business goals driving architecture:**
- Fail-open: Redis downtime must not block API traffic
- Per-tenant limits enable tiered pricing (free: 100 req/min; paid: 10K req/min)
- Config changes must propagate without deployment
- Added latency must be < 5ms — limiter must not be the bottleneck
- Brute-force protection on auth endpoints (stricter limits than general API)

---

## Recognition Framework

### Signals from the Problem
```
- Multiple stateless gateway instances need shared counter state
- Counter must be checked AND incremented atomically (race condition if split)
- < 5ms latency budget — eliminates any solution requiring a DB round-trip
- Redis is the only store supporting atomic ops at 50K RPS with < 2ms latency
- Fail-open required: limiter failure must not equal API failure
- Per-client limits require per-key state (not global counters)
- Config changes without deployment → config in DB, cached in-process
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Contention | Multiple gateway instances competing to update the same client counter |
| Distributed Counters (Redis) | Shared atomic counter across stateless nodes; < 2ms round-trip |
| Redis Lua Scripts | Atomic check-and-increment; eliminates TOCTOU race condition |
| Fail-Open | Redis downtime must not block API traffic; availability > accuracy |
| In-Process Config Cache | Config reads must not hit DB on every request; 30s polling is fine |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| DB-based counters (PostgreSQL) | DB round-trip is 10–50ms; exceeds 5ms budget; cannot do atomic increment at 50K RPS |
| Sticky sessions (no shared state) | Limits bypassed by sending requests to different gateway instances |
| ZooKeeper / etcd for counters | Designed for config/leader election; not for high-frequency increment operations |
| Kafka for rate limiting | Async — cannot synchronously block a request in the critical path |
| In-process only (no Redis) | Breaks with horizontal scaling; each instance has independent counter |
| Fail-closed on Redis downtime | Redis instability would take down the payment API — unacceptable |

### Algorithm Comparison and Final Selection

```
Algorithm 1: Fixed Window Counter
  Implementation: INCR rate:{client}:{minute}; EXPIRE key 60
  Memory: 1 key per client per window (~100 bytes)
  Accuracy: Boundary burst problem
    Example: limit=100/min; client sends 100 at 11:59:50 and 100 at 12:00:01
    Both windows show 100 (within limit); but 200 requests in 2 seconds passed
  Verdict: REJECTED for strict enforcement; acceptable for rough limiting only

Algorithm 2: Sliding Window Log
  Implementation: ZADD rate:{client} timestamp; ZCOUNT (now-60s, now); ZREMRANGEBYSCORE
  Memory: 1 sorted set entry per request per client per window
    At 50K RPS × 10K clients = 500M sorted set entries in memory — prohibitive
  Accuracy: Exact — every request timestamped individually
  Verdict: REJECTED due to memory cost at scale

Algorithm 3: Token Bucket
  Implementation: Store (tokens, last_refill_time) per client; refill at rate R/sec
  Memory: 2 values per client (~50 bytes)
  Accuracy: Allows controlled bursting up to bucket capacity
  Complexity: Atomic Redis implementation requires Lua script with floating-point math
    (must atomically read tokens, compute refill, check, decrement, write back)
  Verdict: VALID alternative for burst-friendly APIs (e.g. human users);
           more complex Redis implementation than sliding window counter

Algorithm 4: Leaky Bucket
  Implementation: Request queue that drains at constant rate
  Adds latency: requests queue under load instead of failing fast
  Verdict: REJECTED for API gateway; adds latency; wrong shape for gateway limiting

Algorithm 5: Sliding Window Counter (CHOSEN)
  Implementation:
    Two keys: current window + previous window
    Estimated rate = prev_window_count × overlap_ratio + curr_window_count
    Single Lua script: read both keys, compute estimate, check limit, increment
  Memory: 2 keys per client (~200 bytes total)
  Accuracy: Within ~1% for smooth traffic; handles boundary burst correctly
  Complexity: Moderate — single Lua script, no floating-point
  Verdict: CHOSEN — best balance of accuracy, memory, and implementation simplicity

Why Sliding Window Counter beats Fixed Window:
  Fixed window allows 2× burst at boundary (as shown above)
  Sliding window counter eliminates this by accounting for prior window activity
  Formula: estimated = floor(prev_count × (window - elapsed) / window) + curr_count
  If prev_count=100 and we are 10s into current window:
    overlap_ratio = (60-10)/60 = 0.833
    estimated = floor(100 × 0.833) + curr_count = 83 + curr_count
    Limit of 100 means only 17 more requests allowed in current window
```

---

## Problem Statement

Distributed rate limiter in front of a payment API gateway. Enforces per-client request quotas across multiple stateless gateway instances. Returns HTTP 429 with `Retry-After` header when limit is exceeded.

---

## Functional Requirements
- Limit each client (by API key) to N requests per time window
- Configurable limits per client tier (free / paid / partner)
- Return 429 with `Retry-After` and `X-RateLimit-*` headers on breach
- Support burst (per-second) and sustained (per-minute) limits
- Config changes propagate without deployment (DB-backed, in-process cache)
- Special stricter limits for auth endpoints (brute-force protection)

## Non-Functional Requirements
- Latency added to request: **< 5ms p99**
- Availability: **99.99%** — limiter failure must not take down the API
- Accuracy: **within 5%** (soft enforcement; exact accuracy not required)
- Scale: **50,000 RPS** across all clients
- Distributed: all gateway instances share counter state via Redis

---

## Capacity Estimation
```
Throughput:
  50,000 RPS × 1 Redis Lua call per request = 50,000 Redis ops/sec
  Redis single node: handles 100K–500K ops/sec → single node sufficient initially
  Redis Cluster (3 shards): handles 300K–1.5M ops/sec → ample headroom

Memory per client (sliding window counter):
  2 keys × ~100 bytes each = 200 bytes per active client
  10,000 active clients = 2 MB total → trivially small

Config table size:
  10,000 client configs × 200 bytes = ~2 MB → fits in in-process cache easily
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Sliding Window Counter | Best accuracy/memory/complexity balance; no boundary burst |
| Redis Lua Scripts | Atomicity: check-and-increment in single operation |
| Fail-Open | Redis downtime → allow all requests; availability > accuracy |
| In-Process Config Cache | 30s refresh from DB; no DB hit on hot path |
| Per-Tier Config | Different limits per pricing tier; configurable without code change |

---

## Data Model

```sql
-- Config store (PostgreSQL, read-heavy, rarely written)
CREATE TABLE rate_limit_config (
  client_id      VARCHAR(64)  PRIMARY KEY,
  tier           VARCHAR(32)  NOT NULL,     -- free / paid / partner
  limit_per_min  INTEGER      NOT NULL,
  limit_per_sec  INTEGER,                   -- optional burst limit
  is_active      BOOLEAN      DEFAULT TRUE,
  updated_at     TIMESTAMP    DEFAULT NOW()
);

-- Tier defaults (fallback if client not in config)
CREATE TABLE rate_limit_tiers (
  tier           VARCHAR(32)  PRIMARY KEY,
  limit_per_min  INTEGER      NOT NULL,
  limit_per_sec  INTEGER
);

INSERT INTO rate_limit_tiers VALUES ('free', 100, 10);
INSERT INTO rate_limit_tiers VALUES ('paid', 10000, 200);
INSERT INTO rate_limit_tiers VALUES ('partner', 100000, 2000);
```

```
Redis key structure:
  rate:{client_id}:{window_start_epoch_seconds_truncated_to_minute}  → INTEGER
  TTL: 2 × window_size_seconds (keep prev window for sliding calculation)

  Example:
    rate:client-abc:28539520  → 47  (current window: 47 requests so far)
    rate:client-abc:28539519  → 100 (previous window: 100 requests)
    Both keys have TTL = 120s (2 × 60s window)
```

---

## API Design

```
Every inbound API request passes through rate limiter middleware transparently.

Response headers added to EVERY response (allowed and denied):
  X-RateLimit-Limit: 1000          (client's limit for this window)
  X-RateLimit-Remaining: 743       (requests remaining in current window)
  X-RateLimit-Reset: 1712345660    (Unix timestamp when window resets)
  X-RateLimit-Window: 60           (window size in seconds)

On limit exceeded:
  HTTP 429 Too Many Requests
  Retry-After: 37                  (seconds until next window)
  X-RateLimit-Remaining: 0
  Body: {
    "error": "rate_limit_exceeded",
    "message": "You have exceeded your rate limit of 1000 requests per minute.",
    "retry_after": 37,
    "limit": 1000,
    "window_seconds": 60
  }

Internal config management API:
  PUT /internal/v1/rate-limits/{client_id}
  Body: { "limit_per_min": 5000, "tier": "paid" }
  Response 200: updated config
  Effect: updates PostgreSQL; in-process cache refreshes within 30s

  GET /internal/v1/rate-limits/{client_id}/status
  Response 200: { "current_count": 47, "limit": 1000, "window_reset_in": 37 }
```

---

## High-Level Architecture

```
Inbound Request
  │
  ▼
API Gateway Instance (N stateless instances behind L4 LB)
  │
  ├── Step 1: Extract client_id
  │     From: Authorization header (API key) or JWT sub claim
  │     Validate key format before touching Redis (prevent injection)
  │
  ├── Step 2: Load limit config
  │     Check in-process HashMap (LRU, 30s TTL)
  │     On miss: SELECT from PostgreSQL → populate in-process cache
  │     Fallback: use tier default if client_id not in config table
  │
  ├── Step 3: Execute Redis Lua script (atomic check + increment)
  │     Keys: rate:{client_id}:{curr_window}, rate:{client_id}:{prev_window}
  │     Args: limit, current_timestamp, window_size
  │     Returns: {allowed: 0|1, estimated_count: N}
  │
  │     ├── ALLOWED → add X-RateLimit-* headers → forward to upstream API
  │     └── DENIED  → return 429 immediately (no upstream call)
  │
  └── If Redis call times out (> 5ms):
        → Fail-open: allow request
        → Increment local in-process counter (rough tracking only)
        → Emit metric: fail_open_activation_total++
        → Do NOT block; API availability > rate limit accuracy

Redis Cluster (shared state across all gateway instances)
  ├── Shard 0: clients A–F (by hash slot)
  ├── Shard 1: clients G–M
  └── Shard 2: clients N–Z

PostgreSQL (config only — never in hot request path)
  └── rate_limit_config table (read on cache miss; written on config change)
```

---

## Detailed Components

### Redis Lua Script (Sliding Window Counter, Fully Atomic)
```lua
-- KEYS[1]: rate:{client_id}:{curr_window_start}
-- KEYS[2]: rate:{client_id}:{prev_window_start}
-- ARGV[1]: limit (integer)
-- ARGV[2]: current_timestamp (Unix seconds)
-- ARGV[3]: window_size_seconds (integer, e.g. 60)
-- Returns: {allowed, estimated_count}
--   allowed = 1 (request permitted) or 0 (rate limited)

local key_curr   = KEYS[1]
local key_prev   = KEYS[2]
local limit      = tonumber(ARGV[1])
local now        = tonumber(ARGV[2])
local window     = tonumber(ARGV[3])

local prev_count = tonumber(redis.call('GET', key_prev) or 0)
local curr_count = tonumber(redis.call('GET', key_curr) or 0)

-- How far into the current window are we?
local elapsed    = now % window
-- What fraction of the previous window's requests count against us?
local overlap    = (window - elapsed) / window
-- Weighted estimate of requests in the sliding window
local estimated  = math.floor(prev_count * overlap) + curr_count

if estimated >= limit then
  -- Denied: do not increment; return current estimated count
  return {0, estimated}
end

-- Allowed: increment current window counter
redis.call('INCR', key_curr)
redis.call('EXPIRE', key_curr, window * 2)

return {1, estimated + 1}

-- Notes on atomicity:
-- This entire script executes as a single Redis command.
-- No other client can modify key_curr or key_prev between the GET and INCR.
-- This eliminates the TOCTOU (time-of-check-time-of-use) race condition
-- that would exist if we used separate GET + INCR commands.
```

### Fail-Open Implementation
```java
// In the gateway middleware:
public RateLimitResult checkRateLimit(String clientId, int limit) {
    try {
        // Set Redis timeout to 4ms (leaves 1ms budget for overhead)
        Future<RateLimitResult> future = redisExecutor.submit(
            () -> executeLuaScript(clientId, limit)
        );
        return future.get(4, TimeUnit.MILLISECONDS);

    } catch (TimeoutException e) {
        // Redis too slow → fail open
        metrics.increment("rate_limit.fail_open.total", tag("client", clientId));
        localCounter.increment(clientId); // rough local tracking
        return RateLimitResult.ALLOWED; // never block on Redis instability

    } catch (RedisConnectionException e) {
        // Redis unreachable → fail open
        metrics.increment("rate_limit.fail_open.total");
        return RateLimitResult.ALLOWED;
    }
}
```

### In-Process Config Cache
```java
// LRU cache with 30s TTL; refreshed on access miss
private final Cache<String, RateLimitConfig> configCache = Caffeine.newBuilder()
    .maximumSize(50_000)                         // 50K client configs in memory
    .expireAfterWrite(30, TimeUnit.SECONDS)      // stale config within 30s
    .build();

public RateLimitConfig getConfig(String clientId) {
    return configCache.get(clientId, id -> {
        // Cache miss: fetch from PostgreSQL
        RateLimitConfig config = configRepository.findById(id);
        if (config == null) {
            // Client not in config: use tier default
            config = tierDefaults.get("free");
        }
        return config;
    });
}
// Config change propagates to all instances within 30s (next cache expiry)
// No pub/sub needed: 30s eventual propagation is acceptable for rate limit configs
```

### Burst Limiting (Two-Level Checking)
```
For endpoints needing both per-second burst AND per-minute sustained limits:

Execute two Lua checks in sequence:
  1. Per-second check: rate:{client_id}:{curr_second}, window=1s, limit=limit_per_sec
  2. Per-minute check: rate:{client_id}:{curr_minute}, window=60s, limit=limit_per_min

Both must pass for request to proceed.
If either denies → 429 with appropriate Retry-After.

This uses 4 Redis key lookups (2 per Lua script × 2 scripts) → still < 5ms total.

Auth endpoint special handling:
  Stricter per-minute limit (e.g. 10 login attempts/min per IP)
  Separate key namespace: auth_rate:{ip_address}:{window}
  Longer lockout on repeated failures: exponential backoff (1min, 5min, 30min)
```

---

## Scaling Strategy
```
Phase 1 (0 → 5K RPS):
  Single Redis node (standalone)
  Fixed window counter (simpler implementation)
  In-process config HashMap

Phase 2 (5K → 50K RPS):
  Redis Sentinel (1 primary + 2 replicas + 3 sentinel nodes)
  Upgrade to sliding window counter (Lua script)
  Add fail-open logic (critical to add BEFORE Redis HA)

Phase 3 (50K → 500K RPS):
  Redis Cluster (3–6 shards, partitioned by client_id hash slot)
  Local in-process pre-check (token bucket) as L1 before Redis
    → Only hits Redis if local bucket is exhausted
    → Reduces Redis ops by ~80% for sustained-rate clients

Phase 4 (> 500K RPS or multi-region):
  Per-region Redis clusters; accept ±5% inaccuracy across regions
  Each region enforces its local limit; global enforcement is approximate
  Use case: global API with regional deployments (e.g. IN, US, EU)
  Trade: cross-region accuracy for latency (avoid cross-region Redis RTT of 50–150ms)
```

---

## Reliability Strategy
- **Redis HA**: Sentinel (3 nodes) or Cluster; automatic failover < 5s
- **Fail-open**: Redis downtime → allow requests through; emit metric; alert on-call
- **Circuit breaker**: Redis error rate > 5% in 10s → bypass rate limiting entirely; alert
- **Config durability**: PostgreSQL with WAL replication; stale in-process config (30s) is acceptable
- **No data loss concern**: rate limit counters are ephemeral (TTL 120s); losing them means temporary under-enforcement, not data corruption

---

## Security Considerations
- **Client ID extraction**: validate API key format with regex before Redis lookup — prevent key injection via crafted client_ids like `*` or `;FLUSHALL`
- **Key namespace isolation**: always prefix `rate:{client_id}` — client cannot control Redis key name
- **DDoS resilience**: rate limiter itself must handle 50K RPS without becoming bottleneck; Lua script is O(1)
- **Bypass prevention**: all traffic paths must route through rate limiter middleware; no backdoor routes
- **Auth endpoint hardening**: stricter limits (10 req/min per IP) with exponential lockout; protects against credential stuffing
- **HMAC-signed API keys**: `key = base64(client_id + "." + expiry + "." + HMAC(client_id + expiry, secret))`; validate signature before Redis lookup; prevents forged or spoofed client IDs

---

## Observability
```
Metrics (emit per request):
  rate_limit_allowed_total      (counter, labels: client_tier, endpoint)
  rate_limit_denied_total       (counter, labels: client_id, endpoint)
    → Spike in denied for one client: abuse detection
    → Spike across all clients: Redis issue or config error
  redis_lua_latency_p99         (target: < 2ms; alert: > 4ms)
  fail_open_activations_total   (counter; any value > 0 means Redis instability)
  config_cache_miss_rate        (should be low after warm-up; spike = config churn)
  rate_limit_config_staleness_seconds (gauge; how old the in-process config is)

Logs (on every 429 response):
  client_id, endpoint, limit, current_count, window, retry_after, request_id

Dashboards:
  - Top 10 most rate-limited clients (abuse detection)
  - Rate limit hit rate by tier (free vs paid vs partner)
  - Redis latency percentiles (P50, P95, P99)
  - Fail-open activation rate (should always be 0)

Alerts:
  redis_lua_latency_p99 > 4ms         → Redis under pressure; approaching fail-open threshold
  fail_open_activations_total > 0     → Redis instability; rate limiting not enforced
  rate_limit_denied spike for 1 client → possible abuse; check client_id
  rate_limit_denied spike for all clients → Redis misconfiguration or config error
  config_db_unreachable               → in-process config will go stale in 30s
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Redis primary fails | ~5s until Sentinel failover; fail-open during failover | Sentinel HA; fail-open logic; immediate alert |
| Redis latency spike (> 5ms) | Fail-open triggers; rate limiting unenforced | Circuit breaker; local in-process fallback |
| Config DB down | Stale in-process config used (up to 30s) | Alert; 30s is acceptable for config staleness |
| Window boundary burst (fixed window) | Up to 2× limit in 2 seconds | Use sliding window counter instead |
| Client ID spoofing | Attacker uses another client's quota | HMAC-signed API keys; signature validation before Redis |
| Hot client (1M RPS limit) | Single Redis key becomes bottleneck | Shard that client's counter across N sub-keys; sum them |
| Misconfigured limit (set to 0) | All requests for that client → 429 | Validation: reject config with limit < 1; alert on all-denied for a client |

---

## Chaos Testing
```
Experiment 1: Kill Redis Primary
  Inject:    Stop Redis primary node
  Expected:  Sentinel promotes replica within 5s
  Expected:  During failover: fail-open kicks in; all requests allowed
  Expected:  fail_open_activations_total increments during failover window
  Expected:  After failover: rate limiting resumes normally
  Red flag:  API starts returning 5xx (fail-closed logic wrong)
  Red flag:  Failover takes > 30s (Sentinel misconfigured)

Experiment 2: Redis Artificial Latency (10ms delay)
  Inject:    Add 10ms artificial latency to Redis via tc (traffic control)
  Expected:  Lua script calls time out at 4ms; fail-open activates
  Expected:  All requests pass through (no 429s during latency injection)
  Expected:  fail_open_activations_total climbs steadily
  Expected:  Alert fires within 1 minute
  Red flag:  API request latency increases by 10ms (timeout not working)

Experiment 3: Config DB Down
  Inject:    Stop PostgreSQL
  Expected:  In-process config cache serves stale config for 30s
  Expected:  Rate limiting continues normally with last-known limits
  Expected:  Alert fires; new clients get tier-default limits
  Expected:  On PostgreSQL recovery: cache refreshes on next miss
  Red flag:  Gateway crashes or rejects all traffic without config DB

Experiment 4: High-Traffic Single Client (10× Their Limit in 1 Second)
  Inject:    Send 10,000 requests/sec for client with 1,000/min limit
  Expected:  429s fire immediately after limit reached (within first second)
  Expected:  Other clients completely unaffected (different Redis keys)
  Expected:  X-RateLimit-Remaining drops to 0; Retry-After set correctly
  Red flag:  Other clients also receive 429s (Redis key collision or contention)

Experiment 5: Window Boundary Burst (Fixed Window Validation)
  Inject:    Send 100 requests at 11:59:55, then 100 at 12:00:01 (limit=100/min)
  With fixed window:  Both batches pass (200 requests in 6 seconds) — FAIL
  With sliding window: Second batch is partially blocked — PASS
  Expected:  Sliding window correctly limits burst at boundary
  Red flag:  200 requests pass when limit is 100 (using fixed window accidentally)
```

---

## Monthly Cost Estimate
```
Scale assumption: 50K RPS peak, 10K active clients, 4 gateway instances

Redis Sentinel (rate limit counters):
  1 primary + 2 replicas × cache.r6g.medium ($60/mo)   = $180/mo
  3 Sentinel nodes × cache.t3.micro ($15/mo)            = $45/mo

PostgreSQL (config only — tiny; minimal ops):
  db.t3.small (2 vCPU, 2 GB)                           = $30/mo

API Gateway instances (not rate limiter cost; context):
  4 × c5.large ($70/mo)                                = $280/mo

Monitoring (CloudWatch / Datadog for rate limit metrics):
                                                        = $50/mo
────────────────────────────────────────────────────────────────
Rate Limiter Infrastructure Total: ~$305/month

Context:
  Rate limiter is cheap: 8% of gateway total cost
  The Redis cluster is the entire cost of rate limiting

At 1M RPS (10× scale):
  Redis Cluster: 6 shards × 3 nodes × cache.r6g.large = $2,340/mo
  (vs. the payments API it protects: likely $50K+/month infrastructure)

Optimisation levers:
  Local in-process pre-check (Phase 3): reduces Redis ops by 80%
  → Redis Cluster can be downsized significantly
  → At 1M RPS with local pre-check: Redis Sentinel may still suffice
```

---

## Migration Story

### Current State
Single API instance with in-process HashMap rate limiting. Counts correct on one instance. Breaks immediately on second instance — each has independent counter; clients send to both instances and bypass limits.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Replace in-process HashMap with Redis INCR (zero downtime)
  Action:
    Deploy Redis standalone
    Replace in-process HashMap with Redis fixed window (INCR + EXPIRE)
    Deploy to 1 gateway instance first; verify 429s fire correctly
    Roll out to all instances one by one
  Verify:
    Rate limiting now consistent across all instances
    Client cannot bypass by hitting different instances
  Monitor:
    Redis latency (should be < 2ms)
    rate_limit_denied_total (should match expected pattern)
  Rollback:
    Revert code to in-process HashMap (single-instance only; not correct at scale)

Step 2: Add fail-open logic (zero downtime — DO THIS BEFORE HA UPGRADE)
  Action:
    Add Redis timeout (4ms) with fail-open fallback
    Add fail_open_activations_total metric
    Verify: kill Redis → API continues serving (no 429s)
    Verify: fail_open metric increments during Redis downtime
  This step is critical and must happen before Sentinel upgrade
  Rollback: remove timeout; restore blocking behaviour (dangerous; only in emergency)

Step 3: Upgrade to Sliding Window Counter Lua Script (zero downtime)
  Action:
    Replace INCR + EXPIRE with Lua script (sliding window counter)
    Deploy to 1 instance; verify no change in normal behaviour
    Verify: window boundary burst test now BLOCKED (see Experiment 5)
    Roll out to all instances
  Rollback:
    Revert Lua script to INCR + EXPIRE (loses sliding window accuracy)

Step 4: Upgrade Redis to Sentinel for HA (5-min maintenance window)
  Action:
    Deploy 3-node Sentinel cluster (1 primary + 2 replicas + 3 sentinels)
    Update Redis client connection to Sentinel endpoint
    Test failover: kill primary → verify < 5s automatic failover
    Verify: fail-open triggers during failover (Step 2 prerequisite)
  Rollback:
    Update connection string back to standalone Redis
    Standalone Redis still running; counters may be stale but functional

Step 5: Add per-tier config with DB-backed config (zero downtime)
  Action:
    Deploy rate_limit_config PostgreSQL table
    Populate with tier defaults
    Add in-process config cache (Caffeine, 30s TTL)
    Enable per-client overrides via management API
  Rollback:
    Remove config cache; revert to hardcoded tier limits in code
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Redis for rate limit counters (ephemeral; TTL 120s; loss = under-enforcement, not data loss).
  → PostgreSQL for per-client config (durable; in-process cached for 30s).
  → In-process cache is a read-through cache; PostgreSQL is always authoritative.

□ Consistency model?
  → Within a single Redis shard: strong (Lua script is atomic).
  → Across Redis shards (if Cluster): eventual (different clients on different shards).
  → Across regions (if multi-region): eventual (each region has its own Redis).
  → Config changes: eventual propagation within 30s across all gateway instances.

□ Failure mode?
  → Redis down → fail-open (allow all requests; accuracy lost; API available).
  → Config DB down → stale config used (30s); alert but no service impact.
  → Gateway instance crashes → LB removes it; other instances continue.
  → Verdict: always choose availability over rate limit accuracy.

□ Hot partitions?
  → High-traffic single client → one Redis shard gets hot.
  → Mitigation: local in-process pre-check (Phase 3) reduces Redis ops by 80%.
  → For very high-limit clients: shard counter across N sub-keys; sum them.

□ Cost driver?
  → Redis cluster (~74% of rate limiter infra cost).
  → Local pre-check (Phase 3) reduces Redis ops and allows downsizing.

□ Security risk?
  → Client ID injection into Redis key names.
    Mitigation: validate API key format; prefix with fixed namespace.
  → Client ID spoofing (fake another client's ID).
    Mitigation: HMAC-signed API keys; validate signature before any Redis call.
  → Bypass by sending to different gateway instances.
    Already solved: Redis is shared; all instances see same counter.

□ Operational burden?
  → Redis Sentinel: monitor failover events; size memory correctly.
  → Config table: low ops burden; rarely changes.
  → Lua script: must be correct; test boundary conditions exhaustively.

□ Migration path?
  → In-process HashMap → Redis INCR → Fail-open → Lua script → Sentinel → Per-tier config.
  → Fail-open (Step 2) must precede HA upgrade (Step 4) — critical ordering dependency.
```

---

## Staff Engineer Discussion Points

**"What if a client has a 1M RPS limit — does one Redis key become a hotspot?"**
Yes, a single Redis key at 1M increments/sec is a real hotspot. Two solutions: (1) Shard the counter across N sub-keys (`rate:{client_id}:shard:{hash(request_id) % N}`), then read all N keys in a pipeline to get the total count — this spreads the write load across N keys. (2) Local in-process pre-check: each gateway instance maintains a local token bucket for that client; only hits Redis when the local bucket is exhausted. At 4 gateway instances, each handles 250K RPS locally before touching Redis.

**"How do you handle a flash crowd against one specific endpoint?"**
Add a second rate limit dimension: per-endpoint in addition to per-client. Two Lua checks per request: `rate:{client_id}:{window}` (client limit) AND `rate:{endpoint_hash}:{window}` (endpoint limit). Both must pass. The endpoint limit protects against one client hammering a slow endpoint even within their overall quota.

**"What's the difference between rate limiting and throttling?"**
Rate limiting: hard cap, fail fast with 429 immediately when limit exceeded. The response is instantaneous. Throttling: soft control — slow down the client (add artificial delay, queue requests, use leaky bucket). Rate limiting protects the server by rejecting excess load. Throttling shapes the client's traffic pattern. For an API gateway, rate limiting is correct — clients should handle 429 and back off. Throttling is correct for async workloads where queuing is acceptable.

**"How do you prevent rate limit bypass via IP rotation or multiple API keys?"**
This moves beyond rate limiting into fraud detection territory. Rate limiting by API key is the primary mechanism. For IP-based attacks: add a separate IP-based rate limit layer (same Redis approach, key = `rate_ip:{ip}:{window}`). For distributed attacks using many API keys: you need anomaly detection (ML or rule-based) on request patterns across keys — beyond what a simple rate limiter can do. The rate limiter is one layer; fraud detection is another.

**"If Redis Sentinel failover takes 5 seconds and you're fail-open, what's the blast radius?"**
5 seconds × 50,000 RPS = 250,000 requests pass through without rate limiting. For a payment API, this means 250K uncontrolled requests in a 5-second window. Acceptable because: (1) legitimate traffic pattern rarely changes in 5 seconds; (2) the alternative (fail-closed) means 250K payment requests fail — that's a worse outcome. The real mitigation is keeping the Sentinel failover under 2s with aggressive timeouts, and having local in-process pre-checks as a secondary line of defence.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Multiple stateless nodes need shared counter"   → Contention + Distributed Counters (Redis)
  "Atomic check-and-increment required"            → Redis Lua Script (eliminates TOCTOU race)
  "Limiter failure must not take down API"         → Fail-Open pattern
  "Config must update without deployment"          → In-Process Config Cache (30s TTL from DB)

DECISION TREE USAGE:
  Need atomic ops at < 5ms?                        → Redis (not DB, not ZooKeeper)
  Redis down: allow or deny?                       → Fail-Open (availability > accuracy)
  Boundary burst problem?                          → Sliding window counter (not fixed window)
  Need burst + sustained limits?                   → Two-level check (per-second + per-minute)

ALGORITHM SELECTION:
  Fixed window:        Simple; boundary burst problem → REJECT for strict enforcement
  Token bucket:        Burst-friendly; complex Redis implementation → Use for human-facing APIs
  Sliding window log:  Exact; prohibitive memory at scale → REJECT for high-throughput
  Leaky bucket:        Adds latency; wrong shape for gateway → REJECT
  Sliding window counter: Accurate (~1%); low memory; simple Lua → CHOOSE for API gateway

SCC LENS:
  STATE:
    Rate limit counters: Redis (ephemeral; TTL 120s; loss = temporary under-enforcement)
    Config: PostgreSQL (authoritative) + in-process cache (read-through; 30s stale)
    No durable state needed: counters expire naturally; no backup/restore required

  COORDINATION:
    Atomicity: Lua script eliminates race condition — entire check+increment is one Redis op
    No coordination between gateway instances: Redis is the shared coordination point
    Config propagation: eventual (30s); no explicit pub/sub needed for config changes

  CONCENTRATION:
    Hot client: single Redis key at very high RPS → local pre-check (Phase 3) or key sharding
    Redis itself: all gateway instances funnel through it → must be sized correctly
    Mitigation: Redis Cluster shards by client_id; each shard handles subset of clients

INTERVIEW ANSWER TRIGGER:
  "Design a rate limiter" →
    1. Spot signals: shared state across stateless nodes, < 5ms, fail-open
    2. Algorithm: reject fixed window (boundary burst), choose sliding window counter
    3. Atomicity: Redis Lua script (not separate GET + INCR = race condition)
    4. Failure: fail-open (availability > accuracy for API gateway)
    5. SCC: State=Redis counters, Coordination=Lua atomicity, Concentration=hot client key
    6. Reject: DB counters (too slow), sticky sessions (bypass), Kafka (async)
```

---

*Next: Case Study 04 — TinyURL Analytics*
