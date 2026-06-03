# Case Study 25 — Advanced API Rate Limiter (Stripe/AWS Scale)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Multi-dimensional limits, Distributed counters, Cost-based throttling.

---

## Business Context

The basic rate limiter in Case 03 handles per-client request counts. At Stripe or AWS scale, rate limiting is a multi-dimensional problem: per API key, per IP, per endpoint, per method, per tenant, and per cost unit. A GET /charges is cheap; a GET /charges?expand[]=all is expensive. A customer doing 100 req/sec to a lightweight endpoint may be fine; 10 req/sec to a heavy endpoint may need throttling.

**Business goals driving architecture:**
- Protect API from resource exhaustion (not just request count)
- Cost-based throttling: expensive operations consume more quota
- Multi-tenant: isolate tenants; one misbehaving tenant cannot affect others
- Burst allowance: legitimate traffic has natural bursts; don't penalise normal usage
- Tiered limits: free, starter, business, enterprise have different allowances
- Developer experience: informative errors with retry guidance

---

## Recognition Framework

### Advanced Dimensions Beyond Case 03

```
Case 03 covered: per-client request count per minute
Case 25 adds:
  1. Per-endpoint limits (expensive endpoints have lower limits)
  2. Cost-based (each request has a "cost unit"; quota depletes faster for expensive ops)
  3. Per-IP limits (detect IP rotation attacks; shared IP for NAT)
  4. Concurrent request limits (max simultaneous in-flight, not just rate)
  5. Adaptive limits (auto-detect abuse patterns; tighten limits dynamically)
  6. Global burst + sustained (token bucket for burst; sliding window for sustained)
  7. Tenant isolation (business account can't affect free account limits)
```

### Cost-Based Throttling

```
Not all API calls are equal:
  GET /charges → reads 1 record → cost = 1 unit
  GET /charges?expand[]=customer&expand[]=payment_method → 3 DB reads → cost = 5 units
  POST /batch_charges → creates 100 charges → cost = 100 units
  GET /reports → aggregation query → cost = 50 units

Standard request counting would let a customer do 1,000 batch charges (cost=100K units)
when their limit should be 10,000 request-equivalents.

Cost-based quota:
  Instead of: "you can make 1,000 requests/min"
  Use:        "you have 10,000 quota units/min"
  Cost table: { GET_charge: 1, LIST_charges: 5, BATCH: 100, REPORT: 50, ... }
  On each request: deduct cost from quota; reject if quota exhausted

AWS API throttling uses this: EC2 DescribeInstances costs more than RunInstances 
in terms of throttling units because it's more expensive server-side.
```

### Token Bucket (Burst) + Sliding Window (Sustained) Combined

```
Token Bucket:
  Fills at rate R tokens/sec; max capacity C
  Each request consumes N tokens (or N cost units)
  Allows burst up to C tokens; sustained rate = R
  Good for: "allow short bursts; cap sustained throughput"

Sliding Window Counter:
  Count requests in last T seconds
  Good for: "never exceed N requests in any T-second window"

Combined (recommended for API platforms):
  Token Bucket: governs burst behaviour (short spikes OK up to burst_capacity)
  Sliding Window: governs sustained rate (never exceed sustained_limit in any window)
  Both must pass; either can reject

Example:
  Token bucket: 100 tokens; refills 10/sec
  Sliding window: max 500 req/min
  
  Scenario A: 100 req in 1 second → token bucket: ALLOW (100 ≤ burst); window: OK (100 < 500)
  Scenario B: 600 req in 1 minute evenly → token bucket: ALLOW (10/sec ≤ refill); window: BLOCK (600 > 500)
  Scenario C: 600 req in 1 second burst → token bucket: BLOCK after 100 (burst exhausted)
```

---

## Problem Statement

Advanced multi-dimensional API rate limiter. Handles per-key, per-IP, per-endpoint, and cost-based throttling. Combined token bucket + sliding window. Tenant isolation. Developer-friendly 429 responses with retry guidance.

---

## Functional Requirements
- Per-API-key quota (request count + cost units)
- Per-IP limits (detect shared NAT; IP rotation)
- Per-endpoint limits (expensive endpoints lower limit)
- Cost-based throttling (operations have different costs)
- Concurrent request limiting (max in-flight per key)
- Burst allowance via token bucket
- Sustained rate via sliding window
- Tier-based limits (free/starter/business/enterprise)
- Dynamic limit adjustment (abuse detection → temporary tighten)
- Graceful 429 with Retry-After, quota remaining, reset timestamp

## Non-Functional Requirements
- Rate limit check latency: **< 5ms** (must not dominate API latency)
- Accuracy: **within 5%** (approximate is acceptable; exact is not required)
- Scale: **1M API keys**, **500K RPS**
- Availability: fail-open on Redis unavailability (from Case 03)

---

## Capacity Estimation
```
Redis storage for rate limits:
  1M API keys × 3 counters each (req/min, cost/min, concurrent) = 3M Redis keys
  Each key: ~100 bytes → 300 MB → single Redis node is fine

  Per-endpoint counters: 100 endpoints × 1M keys = 100M potential keys
  But hot endpoints: only ~1K active keys per endpoint in practice
  → 100 endpoints × 1K active × 100 bytes = 10 MB → trivial

  Per-IP: 10M unique IPs × 1 counter = 10M keys × 50 bytes = 500 MB

Total Redis: ~1.3 GB → small; single Redis node for rate limiting

Throughput:
  500K RPS × 3 Lua script calls (key, IP, endpoint) = 1.5M Redis ops/sec
  Redis Cluster (3 nodes): handles 1.5M+ ops/sec → adequate
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Cost Units | Expensive operations consume more quota; fair resource allocation |
| Token Bucket + Sliding Window | Burst + sustained; complementary |
| Redis Lua (Atomic) | All checks and decrements in single atomic operation |
| Multi-Key Check (Pipeline) | API key + IP + endpoint checked in single Redis round-trip |
| Concurrent Request Tracking | Max in-flight prevents connection pool exhaustion |
| Tier Config (DB-backed cache) | Limits per tier; hot-reload without deployment |

---

## Data Model

```
Redis key structure:
  tb:{api_key}:req              → token bucket state: {tokens, last_refill_ts}
  sw:{api_key}:req:{minute}     → sliding window counter
  sw:{api_key}:cost:{minute}    → cost-based sliding window counter
  concurrent:{api_key}          → SET of in-flight request IDs (TTL auto-expires stale)
  sw_ip:{ip_hash}:{minute}      → per-IP sliding window
  sw_ep:{endpoint_hash}:{api_key}:{minute} → per-endpoint counter

Config (Redis + PostgreSQL backed):
  rate_limit_config:{api_key}   → JSON: tier, limits, burst_capacity, cost_table
```

```sql
-- Rate limit tier configuration (PostgreSQL; cached in-process 30s)
CREATE TABLE rate_limit_tiers (
  tier            VARCHAR(32) PRIMARY KEY,  -- free / starter / business / enterprise
  requests_per_min INTEGER,
  cost_units_per_min INTEGER,
  burst_capacity  INTEGER,
  max_concurrent  INTEGER
);

CREATE TABLE api_key_overrides (
  api_key         VARCHAR(64) PRIMARY KEY,
  tier            VARCHAR(32),
  custom_req_min  INTEGER,                  -- override tier defaults
  custom_cost_min INTEGER,
  is_blocked      BOOLEAN DEFAULT FALSE,    -- immediate hard block
  block_reason    VARCHAR(256)
);

-- Endpoint cost table
CREATE TABLE endpoint_costs (
  endpoint_pattern VARCHAR(128) PRIMARY KEY,  -- /v1/charges, /v1/charges/:id
  cost_units       INTEGER DEFAULT 1,
  has_own_limit    BOOLEAN DEFAULT FALSE,
  limit_per_min    INTEGER                    -- endpoint-specific limit if has_own_limit
);
```

---

## API Design

```
Every API response includes rate limit headers:
  X-RateLimit-Limit: 1000           (req/min for this tier)
  X-RateLimit-Remaining: 743        (remaining in current window)
  X-RateLimit-Reset: 1712345660     (epoch when window resets)
  X-RateLimit-Cost: 5               (cost units consumed by this request)
  X-RateLimit-Cost-Remaining: 8432  (cost units remaining)
  X-RateLimit-Concurrent: 3/10      (current/max in-flight)

On 429 Too Many Requests:
  {
    "error": {
      "type": "rate_limit_error",
      "message": "You have exceeded your rate limit.",
      "code": "rate_limit_exceeded",
      "limit_type": "cost_units",      // which limit was hit
      "retry_after": 37,               // seconds until quota refills
      "limit": 10000,
      "used": 10000,
      "reset_at": "2024-01-01T14:01:00Z",
      "doc_url": "https://docs.platform.com/rate-limits"
    }
  }

Rate limit management API (for platform admins):
  GET  /internal/v1/rate-limits/{api_key}           → current status
  PUT  /internal/v1/rate-limits/{api_key}           → update overrides
  POST /internal/v1/rate-limits/{api_key}/block     → emergency block
  POST /internal/v1/rate-limits/{api_key}/unblock
  GET  /internal/v1/rate-limits/{api_key}/history   → usage history
```

---

## High-Level Architecture

```
Inbound Request
  │
  ▼
Rate Limit Middleware (runs before route handler; same as Case 03)
  │
  ├── Step 1: Load API key config (in-process cache; 30s TTL from DB)
  │     Config: tier, limits, cost_table, is_blocked
  │     If is_blocked: return 403 immediately (no Redis needed)
  │
  ├── Step 2: Determine request cost
  │     cost = endpoint_costs[route_pattern]  (e.g. GET /charges/:id → cost 1)
  │     (from in-process cost table; reloaded on config change)
  │
  ├── Step 3: Execute composite Lua script (SINGLE Redis round-trip)
  │     Checks ALL limits atomically:
  │       a. IP rate limit (per-IP sliding window)
  │       b. API key request limit (token bucket)
  │       c. API key cost limit (sliding window, cost units)
  │       d. API key concurrent limit (SET cardinality)
  │     Returns: {allowed: bool, limit_hit: "ip"|"req"|"cost"|"concurrent"|null}
  │
  ├── Step 4a: If ALLOWED
  │     Track in-flight: SADD concurrent:{api_key} {request_id}; EXPIRE 60
  │     Execute request handler
  │     On completion: SREM concurrent:{api_key} {request_id}
  │     Add X-RateLimit-* headers to response
  │
  └── Step 4b: If RATE LIMITED
        Return 429 with Retry-After and detailed error
        Do NOT execute request handler
        Emit metric: rate_limit_denied{tier, limit_type, endpoint}
```

---

## Detailed Components

### Composite Lua Script (All Checks in One Round-Trip)
```lua
-- KEYS: [ip_key, tb_key, sw_cost_key, concurrent_key]
-- ARGV: [ip_limit, req_limit, burst_cap, refill_rate, cost_limit, cost, max_concurrent, now, window, req_id]

local ip_key        = KEYS[1]
local tb_key        = KEYS[2]
local sw_cost_key   = KEYS[3]
local concurrent_key = KEYS[4]

local ip_limit      = tonumber(ARGV[1])
local req_limit     = tonumber(ARGV[2])
local burst_cap     = tonumber(ARGV[3])
local refill_rate   = tonumber(ARGV[4])  -- tokens per second
local cost_limit    = tonumber(ARGV[5])
local cost          = tonumber(ARGV[6])
local max_conc      = tonumber(ARGV[7])
local now           = tonumber(ARGV[8])
local window        = tonumber(ARGV[9])  -- seconds (e.g. 60)
local req_id        = ARGV[10]

-- Check 1: IP limit (sliding window)
local ip_count = tonumber(redis.call('GET', ip_key) or 0)
if ip_count >= ip_limit then
    return {0, "ip", ip_count, ip_limit}
end

-- Check 2: Token bucket (burst check)
local tb_data = redis.call('HMGET', tb_key, 'tokens', 'last_ts')
local tokens = tonumber(tb_data[1] or burst_cap)
local last_ts = tonumber(tb_data[2] or now)

-- Refill tokens
local elapsed = now - last_ts
local new_tokens = math.min(burst_cap, tokens + elapsed * refill_rate)

if new_tokens < 1 then
    return {0, "burst", new_tokens, burst_cap}
end

-- Check 3: Cost limit (sliding window)
local cost_used = tonumber(redis.call('GET', sw_cost_key) or 0)
if cost_used + cost > cost_limit then
    return {0, "cost", cost_used, cost_limit}
end

-- Check 4: Concurrent limit
local conc = redis.call('SCARD', concurrent_key)
if conc >= max_conc then
    return {0, "concurrent", conc, max_conc}
end

-- All checks passed: apply changes
redis.call('INCR', ip_key)
redis.call('EXPIRE', ip_key, window)

redis.call('HMSET', tb_key, 'tokens', new_tokens - 1, 'last_ts', now)
redis.call('EXPIRE', tb_key, 3600)

redis.call('INCRBY', sw_cost_key, cost)
redis.call('EXPIRE', sw_cost_key, window)

-- Note: concurrent tracking done in application layer (SADD/SREM)
-- (Lua handles check; application handles add/remove for request lifecycle)

return {1, nil, new_tokens - 1, burst_cap}
-- {allowed, limit_hit, current_value, limit_value}
```

### Adaptive Rate Limiting (Abuse Detection)
```java
// Background thread analyzes patterns and temporarily tightens limits
public class AdaptiveRateLimiter {

    @Scheduled(fixedRate = 60_000)  // every minute
    public void detectAndAdaptAbusiveKeys() {
        // Find API keys with unusual patterns in last 10 minutes
        List<String> suspiciousKeys = metricsStore.findKeysWithPattern(
            Pattern.RAPID_ESCALATION,     // suddenly using 10× normal volume
            Pattern.ENDPOINT_HAMMERING,   // one endpoint > 80% of requests
            Pattern.ERROR_FARMING         // > 50% of requests returning 4xx
        );

        for (String apiKey : suspiciousKeys) {
            ApiKeyConfig config = configRepo.findById(apiKey);

            // Temporarily halve the limit for 10 minutes
            int tempLimit = config.getRequestsPerMin() / 2;
            redisConfig.setTemporaryOverride(apiKey, tempLimit, Duration.ofMinutes(10));

            // Alert security team for review
            securityOps.flagForReview(apiKey, "adaptive_rate_limit_triggered");

            // Metric
            metrics.increment("adaptive_rate_limit.triggered", tag("key", apiKey));
        }
    }
}
```

### Developer-Friendly 429 Response
```java
// Rate limit response includes everything developer needs to handle gracefully
public Response build429Response(RateLimitResult result, ApiKeyConfig config) {
    long retryAfterSeconds = computeRetryAfter(result);
    long resetAt = computeResetTimestamp(result);

    return Response.status(429)
        .header("Retry-After", retryAfterSeconds)
        .header("X-RateLimit-Limit", config.getRequestsPerMin())
        .header("X-RateLimit-Remaining", 0)
        .header("X-RateLimit-Reset", resetAt)
        .header("X-RateLimit-Limit-Type", result.getLimitHit())
        .entity(Map.of(
            "error", Map.of(
                "type", "rate_limit_error",
                "code", "rate_limit_exceeded",
                "message", String.format(
                    "You have exceeded your %s rate limit of %d per minute.",
                    result.getLimitHit(), result.getLimit()
                ),
                "retry_after", retryAfterSeconds,
                "reset_at", Instant.ofEpochSecond(resetAt).toString(),
                "limit_type", result.getLimitHit(),
                "doc_url", "https://docs.platform.com/rate-limits#" + result.getLimitHit()
            )
        ))
        .build();
}
```

---

## Scaling Strategy
```
Phase 1 (0 → 50K RPS):
  Single Redis node; per-key request count only
  Fixed tier limits; no cost-based throttling

Phase 2 (50K → 200K RPS):
  Add cost-based throttling (high-cost endpoints)
  Redis Sentinel for HA
  Composite Lua script (all checks in one round-trip)
  Per-IP limits (abuse detection)

Phase 3 (200K → 500K RPS):
  Redis Cluster (shard by api_key hash)
  Local in-process pre-check (L1 before Redis L2) → reduces Redis ops 80%
  Adaptive rate limiting (abuse detection)

Phase 4 (500K+ RPS):
  Edge rate limiting (Cloudflare Workers / CDN)
  Global rate limit state propagation across regions
  Accept cross-region inaccuracy (< 5% over-allowance across regions)
```

---

## Observability
```
Per-dimension metrics:
  rate_limit_allowed_total          (by tier, endpoint, limit_type)
  rate_limit_denied_total           (by api_key, limit_type, tier)
  rate_limit_cost_consumed          (cost units consumed; by tier, endpoint)
  concurrent_requests_high_water    (peak concurrent per api_key)
  adaptive_limit_activations        (abuse detection triggers)
  redis_lua_latency_p99             (target < 2ms)

Business metrics:
  top_10_api_keys_by_cost_units     (heaviest users)
  rate_limit_hit_rate_by_tier       (free tier should be highest)
  endpoint_cost_distribution        (actual vs expected cost unit consumption)

Developer experience metrics:
  retry_after_distribution          (how long developers wait on 429)
  429_to_success_retry_rate         (% of 429s that succeed on retry within Retry-After)
```

---

## Chaos Testing
```
Experiment 1: Cost Explosion Attack
  Inject:    Single API key calls expensive endpoint (cost=100) at 1K req/sec
  Expected:  Cost quota (10K/min) exhausted in 10 seconds (1K × 100 cost/req × 0.1min)
  Expected:  429 with Retry-After and "cost_units" limit_type
  Expected:  Regular cheap endpoints for this key: also blocked (same quota)
  Red flag:  Expensive calls don't consume extra cost units; quota not exhausted

Experiment 2: IP Rotation Attack
  Inject:    Same API key sends from 10K different IP addresses (VPN rotation)
  Expected:  Per-key limit still enforces; IP rotation doesn't help attacker
  Expected:  Per-IP limit: each IP gets blocked after IP_LIMIT requests
  Expected:  Total API key throughput still capped by key-level limit
  Red flag:  IP rotation bypasses rate limits (IP-only limiting without key limiting)

Experiment 3: Concurrent Connection Flooding
  Inject:    Open 50 concurrent connections from one API key (max_concurrent=10)
  Expected:  First 10: admitted; in concurrent tracking SET
  Expected:  11th onwards: 429 with "concurrent" limit_type
  Expected:  As requests complete and SREM fires: new requests admitted
  Red flag:  Concurrent count never decremented (SREM not called on completion)

Experiment 4: Redis Failover During High Traffic
  Inject:    Kill Redis primary during 100K RPS traffic
  Expected:  Sentinel failover in < 5s
  Expected:  During failover: fail-open → all requests pass through
  Expected:  fail_open_activations metric increments
  Expected:  After failover: rate limiting resumes; no data recovery needed (counters reset)
  Red flag:  Service returns 5xx instead of fail-open (wrong failure handling)
```

---

## Monthly Cost Estimate
```
Scale: 500K RPS, 1M API keys

Redis Cluster (3 shards × 3 nodes):
  9 × cache.r6g.large ($130/mo)                     = $1,170/mo
  (1.3 GB data fits easily; cluster for throughput, not memory)

Rate Limit API instances (embedded middleware; no separate service):
  Cost borne by API gateway instances (Case 05)
  Additional Redis round-trip: ~$200/mo marginal cost

Config DB (PostgreSQL; shared with existing):
  Marginal additional tables                        = $0 additional

Total incremental cost: ~$1,370/month
(Rate limiting is cheap; Redis is the entire cost)

vs. commercial alternatives:
  Kong Rate Limiting Advanced Plugin: ~$1,000/mo for 500K RPS
  AWS API Gateway throttling: $3.50/million requests × 43.2B/day ≈ way more
  Building in Redis: ~$1,370/mo → significantly cheaper at scale
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Redis: rate limit counters (ephemeral; loss = fail-open; acceptable).
  → PostgreSQL: tier config and overrides (durable; rarely changes).
  → In-process cache: 30s read-through from PostgreSQL (operational).

□ Consistency model?
  → Lua script: strongly consistent within a Redis shard (single-threaded ops).
  → Across Redis shards: different API keys are independent (no cross-shard counters).
  → On Redis failover: counters reset → brief over-allowance; acceptable.

□ Failure mode?
  → Redis down: fail-open (allow all); emit metric; alert.
  → Config DB down: in-process cache serves stale config (30s acceptable for limits).
  → Both down: all requests pass through; monitoring alerts immediately.

□ Multi-dimensional accuracy:
  → Approximate (sliding window ±5%); exact is not required for rate limiting.
  → Cost units: exact (synchronous deduction in Lua); cannot overspend.
  → Concurrent: exact (SCARD is atomic); strict enforcement needed for connection pools.

□ Developer experience:
  → Every 429 includes: Retry-After, limit type, reset timestamp, doc URL.
  → Gradual ramp: new API keys start at tier limits; no sudden blocks.
  → Quota visibility: X-RateLimit-Remaining on every response (not just 429).

□ SCC LENS:
  STATE: Redis (counters — ephemeral), PostgreSQL (config — durable)
  COORDINATION: Lua script (atomic multi-check), in-process cache (reduce Redis load)
  CONCENTRATION: Redis Cluster sharded by api_key; no cross-shard coordination needed
```

---

## Staff Engineer Discussion Points

**"How do you rate limit a SaaS platform where one customer is behind a corporate NAT (thousands of users share one IP)?"**
Don't use IP as the primary identifier — use the API key. IP rate limiting is supplementary (catch attacks from IPs with no valid key). For authenticated requests: the API key identifies the tenant/customer, not the IP. A corporate NAT with 10K employees all using one IP is fine if they're authenticated with a valid key and within their key-based quota. IP limits should only be applied to: unauthenticated requests, very low limits (e.g., 100 req/min per IP for public endpoints), and known malicious IP ranges (blocklist). The API key is the authoritative identity for rate limiting.

**"How do you handle rate limiting for webhooks (where you're the one sending, not receiving)?"**
Outbound rate limiting is the mirror problem: you need to ensure you don't overwhelm your customers' webhook endpoints. Pattern: per-destination URL token bucket (limit how fast you send to each webhook endpoint). If a customer's server returns 429 or 503: back off exponentially. Implement a separate outbound queue (Kafka) with a consumer that respects per-destination rate limits. This is exactly the Outbox Worker pattern from Case 21 — the webhook delivery service. Key difference from inbound: you control the retry timing; the customer controls the acceptance rate.

---

## Cheat Sheet Tie-in

```
COST-BASED THROTTLING:
  Problem: all requests counted equally → gaming possible (use only cheap endpoints)
  Solution: each endpoint has cost_units; quota measured in cost units not request count
  Implementation: Redis INCRBY cost (not INCR 1); quota = cost_limit/min

MULTI-DIMENSIONAL LIMITS (all checked in single Lua round-trip):
  1. Per-IP: IP reputation defence
  2. Token bucket: burst allowance (short spikes OK)
  3. Sliding window cost: sustained throttle on expensive ops
  4. Concurrent: connection pool protection (max in-flight)
  
  Order matters: cheapest check first (IP → fast reject); most expensive last (concurrent)

COMBINED TOKEN BUCKET + SLIDING WINDOW:
  Token bucket: allows burst up to capacity; refills over time
  Sliding window: never exceed sustained rate in any window
  Use both: covers both short spikes and sustained abuse

SCC LENS:
  STATE: Redis (counters — ephemeral; loss = fail-open)
  COORDINATION: Lua script atomically checks all dimensions; no race condition
  CONCENTRATION: hot API key → single Redis shard; local in-process L1 reduces shard load

vs CASE 03 (BASIC RATE LIMITER):
  Case 03: per-key request count; sliding window; single dimension
  Case 25: multi-dimensional; cost-based; burst+sustained; adaptive; concurrent
  Same Redis Lua principle; extended to 4 dimensions in one script
```

---

*Next: Case Study 26 — Multi-Tenant SaaS Architecture*
