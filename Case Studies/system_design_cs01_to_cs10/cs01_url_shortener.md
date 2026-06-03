# Case Study 01 — URL Shortener

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Recognition Framework, Rejected Patterns, Migration Story.

---

## Business Context

A URL shortener monetises in two ways: free redirects (volume, ad-supported) and paid analytics (click tracking, geo, custom domains). The product promise is: **your link never breaks**.

**Business goals driving architecture:**
- Redirect success rate > 99.99% — the primary SLO
- Analytics is a premium upsell — must never slow the free redirect path
- A slow write (creating a short URL) is acceptable. A failed redirect is not.
- Availability > consistency for analytics (stale click count is acceptable)

---

## Recognition Framework

### Signals from the Problem
```
- 100:1 read/write ratio — redirects vastly outnumber creations
- Simple lookup: short_code → long_url (pure key-value access)
- Records are tiny (~200 bytes per URL)
- Global uniqueness required for short codes
- Analytics must not block the redirect critical path
- Redirects are globally cacheable (same short_code → same long_url)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Read Scaling (Cache-Aside) | 100:1 ratio — DB alone cannot serve 115K reads/sec |
| ID Generation (Snowflake + Base62) | Short codes must be unique, URL-safe, non-guessable |
| CDN / Edge Redirect | Redirects are stateless and globally cacheable — serve from edge |
| Event Streaming (Kafka) | Analytics must decouple from redirect path entirely |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| CQRS | Overkill — read model is just a cache; no complex projection needed |
| Event Sourcing | URL state is not a sequence of events; it is a simple record |
| Saga / 2PC | No distributed transaction needed here |
| Horizontal DB sharding | PostgreSQL + read replicas handles this scale |
| Write-through cache | Adds latency to write path; write volume is low; cache-aside suffices |
| Random short code generation | Requires collision check on every write; Snowflake avoids this |

### Why Not Stronger Consistency for Analytics?
Seeing "14,203 clicks" vs "14,207 clicks" causes no business harm — eventual consistency is correct here. The redirect itself (short_code → long_url) uses strong consistency because returning the wrong URL or a 404 breaks the product promise.

### Why 302 and Not 301?
- **301 (permanent):** browser caches redirect → fewer server hits → analytics blind
- **302 (temporary):** every click hits server → full analytics visibility
- Choose 302 when analytics matter. Choose 301 for maximum CDN offload.

---

## Problem Statement

Users submit a long URL and receive a short alias (e.g. `short.ly/aB3xQ`). Clicking the short URL redirects to the original. Analytics tracks clicks per URL.

---

## Functional Requirements
- Generate unique short URL from long URL
- Redirect short URL → original URL
- Optional: custom aliases, expiry on short URLs
- Analytics: total clicks, geo breakdown, referrer, device type (async, < 5 min lag)
- Optional: delete a short URL (owner only)

## Non-Functional Requirements
- Read/write ratio: **100:1**
- Redirect latency: **< 10ms p99**
- Availability: **99.99%** — broken redirect is a product failure
- Consistency: **strong** for redirect resolution; **eventual** for analytics
- Durability: short URLs must never silently disappear

---

## Capacity Estimation
```
Writes:
  100M new URLs/day = ~1,200 writes/sec

Reads (redirects):
  10B redirects/day = ~115,000 reads/sec

Storage:
  Per URL record: ~500 bytes (long URL avg 200 chars + metadata)
  100M URLs/day × 365 × 500 bytes = 18 TB/year

Cache sizing:
  20% of URLs drive 80% of traffic (Pareto principle)
  Top 20M URLs × 500 bytes = 10 GB → fits in a single Redis cluster node

Bandwidth:
  Read:  115,000 RPS × 500 bytes = ~57 MB/s (mostly CDN cache hits)
  Write: 1,200 RPS × 500 bytes   = ~600 KB/s
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Read Scaling | 100:1 ratio — cache is non-negotiable |
| ID Generation | Unique, URL-safe, non-guessable short codes |
| Cache-Aside | Redirect path must never hit DB on hot URLs |
| Event Streaming | Analytics decoupled from redirect via Kafka |
| CDN | Public redirects are cacheable at edge globally |

---

## Data Model

```sql
-- Core URL records
CREATE TABLE urls (
  short_code   VARCHAR(8)   PRIMARY KEY,
  long_url     TEXT         NOT NULL,
  user_id      BIGINT,
  created_at   TIMESTAMP    DEFAULT NOW(),
  expires_at   TIMESTAMP,
  is_active    BOOLEAN      DEFAULT TRUE
);

-- Unique index to support idempotent creation
CREATE UNIQUE INDEX idx_urls_long_url_user ON urls(user_id, long_url);

-- Click events: append-only, never update
-- Stored in ClickHouse for analytics, not PostgreSQL
-- Schema shown here for reference:
-- short_code, clicked_at, country_code, referrer, device_type, session_id
```

**Key decisions:**
- `short_code` is the primary key — all redirects are point lookups by code
- Click events live in ClickHouse (columnar), not PostgreSQL (row-based)
- Separation of analytics from URL records is mandatory — different access patterns

---

## API Design

```
POST /api/v1/urls
Body:     { "long_url": "https://...", "custom_alias": "myBrand", "expires_in_days": 30 }
Response: 201 { "short_url": "https://short.ly/aB3xQ", "short_code": "aB3xQ" }

GET /{short_code}
Response: 302 Location: https://original-long-url.com
Response: 404 { "error": "not_found" }
Response: 410 { "error": "expired" }

GET /api/v1/urls/{short_code}/stats
Response: 200 {
  "total_clicks": 14203,
  "clicks_by_day": [...],
  "top_countries": [...],
  "top_referrers": [...]
}

DELETE /api/v1/urls/{short_code}
Response: 204
```

---

## High-Level Architecture

```
Client
  │
  ▼
CDN (caches 302 responses for hot short_codes, TTL = 1h)
  │
  ▼
Load Balancer
  │
  ▼
API Service (stateless, horizontally scaled)
  │                    │
  ▼                    ▼
Redis Cache        PostgreSQL
(short_code        (primary +
 → long_url,        read replicas)
 TTL 24h)

On every redirect: async emit to Kafka (fire-and-forget)
  │
  ▼
Kafka Topic: click-events
  │
  ├── ClickHouse Writer (batch insert every 5s)
  ├── Flink Aggregator → PostgreSQL hourly_stats
  └── Redis INCR clicks:{short_code} (real-time counter)

ID Generation:
  Snowflake ID (64-bit) → Base62 encode → 7-char short_code
  62^7 = 3.5 trillion combinations → lasts ~92,000 years at 1,200 writes/sec
```

---

## Detailed Components

### Short Code Generation
```
1. Generate 64-bit Snowflake ID
     Structure: 41-bit timestamp | 10-bit machine ID | 12-bit sequence
     Monotonically increasing; no central coordinator needed

2. Base62-encode the integer
     Characters: [0-9][a-z][A-Z] = 62 chars
     7 chars = 62^7 = 3.5 trillion unique codes

3. Write to PostgreSQL

Why not random strings?
  → Requires SELECT before INSERT to check collision — 2 DB ops per write
  → Under high load, collision probability increases non-linearly
  → Snowflake: single INSERT, guaranteed unique

Why not UUID?
  → 36 chars — too long for a "short" URL
  → UUIDs are not sortable (v4) — bad for DB index performance
```

### Redirect Flow (Critical Path)
```
1. Client → GET /aB3xQ
2. CDN HIT   → return 302 immediately (< 1ms, no server involved)
   CDN MISS  → API Service
3. Redis HIT → return 302 (< 2ms round-trip)
   Redis MISS → query PostgreSQL read replica
4. Check is_active and expires_at
   → expired: return 410
   → active: populate Redis (TTL 24h), return 302
5. Async (non-blocking): emit click event to Kafka
   Redirect does NOT wait for Kafka ack
```

### Cache Strategy
```
Pattern:   Cache-Aside (lazy population)
Key:       url:{short_code} → long_url
TTL:       24h (or match URL expiry if shorter)
Eviction:  LRU — cold URLs fall out naturally
On delete: explicit DEL url:{short_code} to invalidate immediately
On expiry: background TTL + background sweep job
```

### Thundering Herd on Cold Cache
```
Problem: Popular URL expires → 10K concurrent requests all miss Redis
         → all hit PostgreSQL simultaneously → DB overload

Solutions applied:
  1. Staggered TTL: TTL = 86400 + random(0, 3600)
     Spreads expiry events; avoids synchronized stampede

  2. Local in-process cache (Caffeine) in API pods
     Top 10K codes cached in-process; never hit Redis
     Evicted on LRU; serves as L1 before Redis (L2)
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K RPS):
  Single API instance + PostgreSQL + no cache
  Works fine; PostgreSQL handles this easily

Phase 2 (1K → 10K RPS):
  Add Redis cache-aside
  Cache hit rate 80%+ within 1 hour of warm-up
  DB load drops dramatically

Phase 3 (10K → 100K RPS):
  Horizontal API scaling (stateless → easy)
  Redis Cluster (shard by short_code CRC16 hash)
  PostgreSQL read replicas for cache misses
  Decouple analytics to Kafka + ClickHouse

Phase 4 (100K+ RPS):
  CDN caching for redirect responses (major traffic handled at edge)
  Local Caffeine cache in API pods for top-N codes
  Multi-region deployments; geo-routing via anycast DNS

Phase 5 (global, multi-region):
  Active-active with eventual consistency between regions
  Regional Redis clusters; accept ~100ms cross-region replication lag
  Short codes created in one region visible globally within 100ms
```

---

## Reliability Strategy
- **Redis failure**: fall through to PostgreSQL — latency degrades, service stays up
- **DB failure**: serve stale cache; circuit breaker on DB calls; 503 for new short codes
- **Idempotent writes**: same (user_id, long_url) → same short_code (unique index + UPSERT)
- **Expiry**: background job sweeps `expires_at < NOW()` every 5 minutes; removes cache entries
- **Health checks**: liveness + readiness probes on all pods; Redis Sentinel for HA

---

## Security Considerations
- **Rate limit writes**: 100 URL creations/hour per API key (prevent bulk phishing link generation)
- **Malicious URL scanning**: check long_url against Google Safe Browsing API before creating
- **Custom alias blocklist**: reserve `api`, `admin`, `health`, `static`, `robots.txt`
- **SSRF prevention**: validate long_url scheme (http/https only); do not follow redirects internally
- **Analytics PII**: hash IP before storing in ClickHouse; GDPR compliance on geo data retention
- **Enumeration prevention**: Snowflake IDs are non-sequential (timestamp + machine + sequence)

---

## Observability
```
Metrics (emit from every API instance):
  redirect_latency_p50/p99    → target < 10ms p99
  cache_hit_rate              → target > 95%
  cdn_offload_rate            → % of redirects served at edge (no server hit)
  url_creation_rate           → writes/sec
  db_fallback_rate            → redirects that reached PostgreSQL (Redis miss)

Logs (structured JSON, every redirect):
  short_code, latency_ms, cache_tier (cdn/redis/db/miss),
  status_code, request_id, timestamp

Traces:
  Full distributed trace on redirect path: CDN → API → Redis → DB
  Propagate X-Request-ID to all downstream systems

Alerts:
  cache_hit_rate < 90%        → Redis issue or traffic pattern shift
  redirect_p99 > 50ms         → DB fallback happening too frequently
  error_rate > 1%             → investigate immediately
  cdn_offload_rate < 50%      → CDN misconfiguration or TTL too short
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Redis primary down | All redirects fall to DB; latency ~50ms | Redis Sentinel failover < 5s; DB replicas absorb load |
| DB primary down | Writes fail (503); reads from cache only | RDS Multi-AZ auto-failover; cache serves existing short codes |
| Short code collision | Two URLs get same code (shouldn't happen with Snowflake) | DB unique constraint catches it; retry with new Snowflake ID |
| Viral URL overwhelms Redis | Single Redis node CPU spikes to 100% | Local Caffeine cache in API pods; CDN caches the 302 |
| Kafka consumer lag | Click counts delayed in dashboard | Analytics is async — acceptable; alert on lag > 5 min |
| CDN misconfiguration | All redirects miss CDN; API + Redis overloaded | CDN config as code; staged rollout; rollback in < 2 min |

---

## Chaos Testing
```
Experiment 1: Kill Redis Primary
  Inject:    Stop Redis primary node
  Expected:  Sentinel promotes replica within 5s
  Expected:  Redirect latency increases to ~50ms (DB fallback); no 5xx errors
  Red flag:  5xx spike → fail-closed logic wrong; should fall through to DB

Experiment 2: Kill DB Primary
  Inject:    Stop PostgreSQL primary
  Expected:  Cache serves all recent short codes (zero errors for hot codes)
  Expected:  New short code creation fails with 503 (write path needs primary)
  Expected:  DB failover promotes replica within 30s; writes resume
  Red flag:  All redirects fail → cache not being used

Experiment 3: Kill Kafka (analytics pipeline)
  Inject:    Stop Kafka cluster
  Expected:  Redirects continue normally (Kafka emit is fire-and-forget)
  Expected:  Analytics events queue in producer buffer; delivered after recovery
  Red flag:  Redirect latency increases → Kafka emit is blocking the critical path

Experiment 4: Viral URL Thundering Herd
  Inject:    Expire a hot short_code from Redis; fire 10K concurrent requests
  Expected:  First request misses → populates Redis; remaining requests hit Redis
  Expected:  DB receives at most 1–2 queries (with mutex lock)
  Red flag:  10K DB queries simultaneously → thundering herd not mitigated
```

---

## Monthly Cost Estimate
```
Scale assumption: 115K redirects/sec peak, 20M DAU, 100M URLs stored

API Service:
  8 × t3.large ($67/mo)                        = $536/mo
  (stateless; scales horizontally)

Redis Cluster:
  6 × cache.r6g.large ($130/mo)                = $780/mo
  (10 GB hot URLs comfortably; 3 primary + 3 replica shards)

PostgreSQL:
  1 primary + 2 read replicas × db.r6g.large   = $600/mo

CDN (CloudFront):
  ~148 TB/month egress × $0.008/GB             = $1,184/mo
  (57 MB/s × 86,400s × 30 days; most served from edge)

Kafka (analytics pipeline):
  3 × kafka.m5.large ($150/mo)                 = $450/mo

ClickHouse (analytics storage):
  3 × r6g.xlarge ($300/mo)                     = $900/mo
  Storage: ~5 TB compressed × $0.10/GB         = $500/mo

Monitoring, networking, misc:
                                                = $500/mo
──────────────────────────────────────────────────────────
Total: ~$5,450/month at 115K RPS peak

Cost drivers:
  CDN egress:      22% of total
  ClickHouse:      26% of total
  Redis:           14% of total

Optimisation levers:
  - Longer CDN TTLs for hot short codes → reduces Redis + API load significantly
  - ClickHouse MergeTree compression (5–10×) → halves storage cost
  - Reserved instances for stable baseline → ~30% discount on EC2/ElastiCache
```

---

## Migration Story

### Current State
Single PostgreSQL server. All redirects hit the DB. Works fine at 1K RPS. At 10K RPS, DB CPU is at 90% and redirect latency is 80ms p99.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Add Redis cache-aside (zero downtime)
  Action:   Deploy Redis Standalone; update API to check Redis before DB
  Monitor:  Cache hit rate climbs from 0% to ~80% within 1 hour as cache warms
  Impact:   DB CPU drops from 90% to ~20%; redirect latency drops to ~5ms
  Rollback: Remove Redis check in code; re-deploy; DB handles all traffic

Step 2: Add PostgreSQL read replicas (zero downtime)
  Action:   Provision 2 read replicas; route all SELECT to replicas (round-robin)
  Monitor:  Primary CPU drops ~60% (all writes; no reads)
  Impact:   Write path (URL creation) and DB failover both improved
  Rollback: Route all reads back to primary

Step 3: Add CDN for redirect caching (zero downtime)
  Action:   Put CloudFront in front of API; cache 302 responses (TTL 1h)
  Monitor:  CDN offload rate climbs; API RPS drops; Redis load drops
  Impact:   Majority of redirects now served from edge; cost drops
  Rollback: Remove CloudFront from DNS CNAME; traffic returns to API directly

Step 4: Decouple analytics to Kafka (zero downtime)
  Action:   Add async Kafka emit in redirect handler (non-blocking)
            Deploy Kafka + ClickHouse + Flink consumer
            Migrate existing click data from PostgreSQL to ClickHouse
            Switch dashboard queries to ClickHouse
            Remove synchronous click INSERT from redirect path
  Monitor:  Redirect latency drops further (no synchronous analytics write)
  Impact:   Analytics scalable to billions of events; redirect path clean
  Rollback: Re-enable synchronous PostgreSQL INSERT; stop Kafka emit

Step 5: Upgrade Redis to Sentinel for HA (5-min maintenance window)
  Action:   Deploy 3-node Sentinel; update client to Sentinel endpoint
  Test:     Kill primary → verify auto-failover in < 5s
  Rollback: Point client back to standalone Redis
```

**Key lesson:** Each step solves a specific bottleneck. You never need to do all steps at once. Start with the bottleneck that's actually hurting you today.

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL for URL records. Redis is a cache. ClickHouse for analytics events.
  → If Redis and DB disagree, DB wins. Cache is always expendable.

□ Consistency model?
  → Redirect resolution: strong (Redis miss falls to DB; DB is authoritative).
  → Analytics: eventual (Kafka → ClickHouse; 5-min lag acceptable).
  → Cache TTL means redirects can be stale by up to 24h after URL update.

□ Failure mode?
  → Redis down → DB fallback (latency degrades; service stays up).
  → DB down → cache serves existing short codes; new URL creation fails.
  → Kafka down → redirects unaffected; analytics delayed until recovery.

□ Hot partitions?
  → Viral URL → hot Redis key → hot CDN edge node.
  → Mitigation: local Caffeine cache in API pods (L1); CDN caches 302 (L0).
  → Redis key is a single value (GET/SET); no write contention.

□ Cost driver?
  → CDN egress (22%) and ClickHouse (26%).
  → Optimise: longer CDN TTLs; ClickHouse compression.

□ Security risk?
  → Phishing short links. Mitigation: Google Safe Browsing check on every URL creation.
  → Enumeration of short codes. Mitigation: Snowflake IDs (non-sequential) + redirect rate limiting.

□ Operational burden?
  → Redis Cluster: shard rebalancing when adding nodes.
  → ClickHouse: capacity planning; partition management.
  → Expiry worker: must be monitored; heartbeat alert if it stops.

□ Migration path?
  → PostgreSQL only → Redis → Read replicas → CDN → Kafka analytics.
  → Each step independently rollbackable. Test each in staging first.
```

---

## Staff Engineer Discussion Points

**"How do you handle a viral URL getting 10M clicks in 10 minutes?"**
Two layers: (1) CDN caches the 302 at edge — most clicks never reach your servers. (2) Local Caffeine cache in API pods for top-N codes — eliminates Redis network RTT for the hottest codes. Redis alone can't absorb 10M clicks/min even if it's fast enough per-op, because the network bandwidth becomes the bottleneck.

**"What if Redis and DB are both slow simultaneously?"**
Circuit breaker on DB calls. If both Redis and DB are slow, stop queuing requests (which makes it worse). Return 503 with `Retry-After` header. It's better to fail fast than to queue-pile and cause cascading failure. The circuit breaker trips at 50% error rate over 10 seconds.

**"How would you support multi-tenancy (each customer has their own domain)?"**
Tenant-aware routing at the load balancer level. The short URL becomes `customer.short.ly/aB3xQ` not `short.ly/aB3xQ`. Composite key in DB: `(tenant_id, short_code)`. Redis key: `url:{tenant_id}:{short_code}`. CDN configured per-tenant domain. The core architecture doesn't change; tenant isolation is an additive concern.

**"How do you prevent short code enumeration attacks?"**
Snowflake IDs are non-sequential — an attacker can't guess adjacent short codes by incrementing. Add rate limiting on the redirect path (e.g. 1000 req/min per IP) to prevent bulk crawling. For sensitive short codes, add an additional secret token requirement.

**"What's your strategy if ClickHouse analytics data is wrong (bug in Flink aggregation)?"**
This is why Kafka has 7-day retention. Fix the Flink job, reset the consumer group offset to the start of the affected period, and replay. Raw events in Kafka are the source of truth for analytics. Pre-aggregated PostgreSQL hourly_stats can be rebuilt from ClickHouse. This is the entire value of the Lambda architecture here.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Reads dominate (100:1)"         → Read Scaling: Cache-Aside + CDN
  "Need globally unique short IDs" → ID Generation: Snowflake ID + Base62 encode
  "Analytics must not block reads" → Event Streaming: Kafka fire-and-forget
  "Fast lookup globally"           → CDN caching at edge

DECISION TREE USAGE:
  Need analytics replay?           → Kafka (not RabbitMQ — need replay on bug fix)
  Analytics on billions of rows?   → ClickHouse columnar (not PostgreSQL row-based)
  Cache hit rate critical?         → LRU eviction + staggered TTL

PATTERNS REJECTED (and why):
  CQRS          → read model is just a cache; no complex projection
  Event Sourcing → URL state is a record, not an event sequence
  Write-through  → write volume low; cache-aside sufficient; don't add write latency

SCC LENS:
  STATE:
    Source of truth: long_url record in PostgreSQL
    Hot cache: short_code → long_url in Redis (ephemeral; expendable)
    Analytics: click events in ClickHouse (append-only; columnar)

  COORDINATION:
    short_code uniqueness: DB unique constraint (single source of truth)
    Cache invalidation: explicit DEL on delete; TTL expiry for natural cleanup
    No coordination needed between API instances (all stateless)

  CONCENTRATION:
    Viral short_code → hot Redis key → local Caffeine cache in API pods
    CDN absorbs bulk of traffic for hot codes (never reaches Redis or DB)
    Write path is low volume — not a concentration concern

INTERVIEW ANSWER TRIGGER:
  "Design a URL shortener" →
    1. State pattern signals: 100:1 read/write, tiny records, key-value lookup
    2. Select: Cache-Aside + ID Generation + CDN + Kafka analytics
    3. Reject with reason: CQRS (no projection), Event Sourcing (not state events)
    4. SCC: State=PostgreSQL+Redis, Coordination=unique constraint, Concentration=viral URL
    5. Discuss tradeoffs: 302 vs 301, eventual vs strong for analytics
```

---

*Next: Case Study 02 — Pastebin*
