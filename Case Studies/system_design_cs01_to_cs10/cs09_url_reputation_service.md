# Case Study 09 — URL Reputation Service

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Bloom Filter mechanics, Tiered Lookup, False Positive handling.

---

## Business Context

A URL reputation service answers one question: "Is this URL safe to click?" It sits in the critical path of payment confirmation emails, notification links, and user-generated content. A false negative (letting a malicious URL through) enables phishing. A false positive (blocking a legitimate URL) breaks user experience. The service must be fast enough to not add perceptible latency to page loads.

**Business goals driving architecture:**
- Block phishing and malware URLs before users click them
- Sub-millisecond check for known-good URLs (vast majority of traffic)
- Fresh threat intelligence: newly registered phishing domains must be blocked within minutes
- No single external dependency that can take down link-checking
- False positives (blocking legitimate URLs) are costly — must be low and recoverable

---

## Recognition Framework

### Signals from the Problem
```
- 99%+ of URLs are known-good → fast path for negatives is critical
- Reputation data is probabilistic → some false positives acceptable
- Dataset is large (billions of URLs) but check is binary → probabilistic filter fits
- Newly malicious URLs must propagate fast → streaming update required
- Multiple data sources (internal + external feeds) → aggregation layer needed
- URL check is in critical path → must not add > 5ms to page load
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Bloom Filter | 99% of checks are "not malicious" — fast negative answer with near-zero memory |
| Tiered Lookup | Bloom → Redis → DB — short-circuit on first conclusive answer |
| Write-Behind Cache | Async DB persist from Redis; URL reputation checks are fire-and-query |
| Streaming Threat Feed | New malicious URLs propagate fast (< 1 min) via Kafka |
| Probabilistic Data Structure | Billions of URLs; exact set membership is too expensive in RAM |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Hash set of all malicious URLs in memory | 1B malicious URLs × 16 bytes = 16 GB per instance; too expensive |
| DB lookup on every check | 5–50ms per check; unacceptable for link-check in page load |
| External API only (Google Safe Browsing) | Single external dependency; network hop; rate limits; adds latency |
| Full URL match (no normalization) | `http://evil.com/path` and `https://evil.com/path` treated as different; bypass trivial |

### Why Bloom Filter as L1?

```
Problem: 1 billion known-malicious URLs; check must be < 0.1ms; memory must be < 1 GB

Option A: Hash Set (exact)
  Memory: 1B entries × 16 bytes hash = 16 GB
  Lookup: O(1), exact — no false positives or negatives
  Verdict: 16 GB per instance is prohibitive

Option B: Bloom Filter (probabilistic)
  Memory: optimal at 1B entries, 1% FPR = ~1.2 GB
          optimal at 1B entries, 0.1% FPR = ~1.8 GB
  Lookup: O(k) hash computations — < 0.1ms
  False negative rate: 0% (never misses a known-bad URL)
  False positive rate: 1% (occasionally flags a good URL as potentially bad)
  Verdict: 1.8 GB for 0.1% FPR — dramatically cheaper; false negatives are impossible

Why false negatives are impossible in a Bloom Filter:
  Bloom Filter only produces false POSITIVEs — it says "maybe in set" when not
  It NEVER produces false NEGATIVEs — it never says "not in set" when it is
  For URL reputation: a false negative (missing a bad URL) is a security failure
  A false positive (flagging a good URL) triggers L2 lookup to disambiguate → not a problem

The tiers exist because of false positives:
  L1: Bloom says "maybe malicious" → can't trust it alone
  L2: Redis exact check → conclusive for hot URLs
  L3: DB exact check → conclusive for cold URLs
  If L2 and L3 both miss → URL is not in malicious set (Bloom false positive confirmed)
```

---

## Problem Statement

URL reputation service used to classify URLs as safe or malicious. Integrated into email links, notification links, and user-generated content rendering. Must return a verdict in < 5ms for 99% of checks.

---

## Functional Requirements
- check(url) → {verdict: SAFE | MALICIOUS | UNKNOWN, confidence: float, category: string}
- Ingest malicious URL feeds (external + internal threat intel)
- Report false positives (users can flag incorrect verdicts)
- Admin: manually override verdict for a specific URL
- Purge URLs from reputation list (legal takedown, false positive confirmed)

## Non-Functional Requirements
- Latency: **< 1ms** for known-safe URLs (Bloom negative)
- Latency: **< 5ms p99** for all other URLs
- Throughput: **50,000 checks/sec**
- False negative rate: **0%** (never miss a known-malicious URL)
- False positive rate: **< 0.1%** (1 in 1,000 safe URLs incorrectly flagged)
- Freshness: malicious URL added to feeds → blocked within **60 seconds**

---

## Capacity Estimation
```
Dataset:
  Malicious URLs: ~1 billion (conservative; includes all-time historical)
  Active malicious URLs (last 90 days): ~50 million
  New malicious URLs added/day: ~1 million (threat feeds)

Bloom Filter sizing:
  50M active URLs, 0.1% FPR:
    bits = -n × ln(p) / (ln(2))^2 = -50M × ln(0.001) / 0.48 ≈ 720M bits = ~90 MB
  1B total URLs (historical), 0.1% FPR: ~1.8 GB
  Use two filters: active (90 MB, fast) + historical (1.8 GB, fallback)

Redis hot cache:
  Top 1M recently-seen "maybe malicious" URLs (Bloom positives):
  Each entry: URL hash (32 bytes) + verdict (10 bytes) = ~50 bytes
  1M × 50 bytes = 50 MB → trivially small

Throughput:
  50,000 checks/sec × ~3 hash operations per Bloom check = 150,000 hash ops/sec
  Pure CPU; easily handled by 2–4 cores per instance

Threat feed ingestion:
  1M new URLs/day = ~12 URLs/sec → trivially low for write path
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Bloom Filter (two-tier) | < 0.1ms negative answer; zero false negatives; 90 MB for 50M URLs |
| Tiered Lookup (L1→L2→L3) | Short-circuit on first conclusive answer; DB only for cold misses |
| Kafka (threat feed ingestion) | Async fan-out to all service instances; replayable |
| Redis (verdict cache) | Fast L2 lookup for Bloom positives; TTL-managed |
| URL Normalization | `http://evil.com` ≡ `https://evil.com` — canonicalize before any hash |

---

## Data Model

```sql
-- Malicious URL registry (PostgreSQL — authoritative)
CREATE TABLE url_verdicts (
  url_hash       CHAR(64)      PRIMARY KEY,  -- SHA-256 of normalized URL
  url_pattern    TEXT,                        -- original URL or domain pattern
  verdict        VARCHAR(16)   NOT NULL,      -- MALICIOUS / SAFE_OVERRIDE / UNKNOWN
  category       VARCHAR(64),                 -- PHISHING / MALWARE / SPAM / C2 / etc.
  confidence     DECIMAL(3,2), -- 0.00–1.00
  source         VARCHAR(64),                 -- GOOGLE_SAFE_BROWSING / INTERNAL / MANUAL
  first_seen_at  TIMESTAMP     DEFAULT NOW(),
  last_seen_at   TIMESTAMP     DEFAULT NOW(),
  expires_at     TIMESTAMP,                   -- NULL = permanent; set for temporary blocks
  added_by       VARCHAR(64),
  notes          TEXT
);

CREATE INDEX idx_url_verdicts_pattern ON url_verdicts(url_pattern);
CREATE INDEX idx_url_verdicts_expires ON url_verdicts(expires_at)
  WHERE expires_at IS NOT NULL;

-- False positive reports
CREATE TABLE false_positive_reports (
  id             BIGSERIAL     PRIMARY KEY,
  url_hash       CHAR(64),
  reported_by    VARCHAR(64),
  reported_at    TIMESTAMP     DEFAULT NOW(),
  resolved       BOOLEAN       DEFAULT FALSE,
  resolution     TEXT
);
```

```
Redis key structure:
  verdict:{url_hash}           → JSON: {verdict, category, confidence, ttl_source}
  TTL: 5 minutes for MALICIOUS (re-check frequently as threats evolve)
       1 hour for SAFE_OVERRIDE (stable)
       30 seconds for UNKNOWN (re-check DB quickly for new intel)

Bloom Filter (in-process, loaded at startup):
  Active filter:    built from url_verdicts WHERE verdict='MALICIOUS' AND (expires_at IS NULL OR expires_at > NOW())
  Rebuilt every:    60 seconds (picks up new malicious URLs from Kafka ingestion)
  Rebuild strategy: build new filter in background → atomic pointer swap → old filter GC'd
```

---

## API Design

```
POST /api/v1/reputation/check
Body: { "url": "https://suspicious-site.com/login?redirect=http://evil.com" }
Response 200: {
  "url": "https://suspicious-site.com/login?...",
  "normalized_url": "https://suspicious-site.com/login",
  "verdict": "MALICIOUS",
  "category": "PHISHING",
  "confidence": 0.97,
  "checked_at": "2024-01-01T14:00:00Z",
  "source": "GOOGLE_SAFE_BROWSING",
  "check_latency_ms": 2.1
}

POST /api/v1/reputation/check/batch
Body: { "urls": ["url1", "url2", ..., "url100"] }
Response 200: { "results": [...] }
(batch: more efficient for email scanning with many links)

POST /api/v1/reputation/false-positive
Body: {
  "url": "https://legitimate-bank.com/secure-login",
  "reason": "This is our own domain; incorrectly flagged"
}
Response 202: { "report_id": "fp-123", "status": "under_review" }

PUT /api/v1/reputation/override
Body: {
  "url": "https://confirmed-phishing.com",
  "verdict": "MALICIOUS",
  "category": "PHISHING",
  "confidence": 1.0,
  "added_by": "security-team"
}
Response 200: propagated to all instances within 60s

GET /api/v1/reputation/stats
Response 200: {
  "bloom_filter_size_mb": 92,
  "bloom_fpr": 0.001,
  "redis_cache_hit_rate": 0.73,
  "checks_per_second": 42000,
  "malicious_url_count": 49823421
}
```

---

## High-Level Architecture

```
Client (email service, link renderer, content scanner)
  │
  ▼
URL Reputation API (N stateless instances)
  │
  ├── Step 1: URL Normalization
  │     Lowercase scheme + host
  │     Remove tracking params (utm_*, fbclid, gclid)
  │     Decode percent-encoding
  │     Remove default ports (http:80, https:443)
  │     Resolve redirects (follow up to 3 hops; detect redirect chains)
  │     Compute SHA-256(normalized_url) → url_hash
  │
  ├── Step 2: L1 — Bloom Filter check (in-process; ~0.1ms)
  │     bloom.contains(url_hash)
  │     MISS (definitely safe): return SAFE immediately → done in < 1ms
  │     HIT (maybe malicious): proceed to L2
  │
  ├── Step 3: L2 — Redis exact check (~1ms)
  │     GET verdict:{url_hash}
  │     HIT: return cached verdict → done in < 2ms
  │     MISS: proceed to L3
  │
  ├── Step 4: L3 — PostgreSQL exact check (~5ms)
  │     SELECT * FROM url_verdicts WHERE url_hash = ?
  │     HIT (MALICIOUS): populate Redis cache → return verdict
  │     MISS: Bloom false positive confirmed → return SAFE (after Redis population)
  │
  └── External check (optional; only for UNKNOWN or low-confidence):
        Query Google Safe Browsing API, VirusTotal, etc.
        Async: do not block response; return UNKNOWN; update DB when result arrives
        Use for newly-seen URLs not yet in any feed

Threat Feed Ingestion Pipeline (separate service):
  External feeds (GSB, VirusTotal, internal threat intel):
    → Kafka topic: malicious-url-feeds
    → Consumer: normalize URL → compute hash → UPSERT url_verdicts
    → After DB write: PUBLISH to Redis channel "new-malicious-urls"
    → All API instances subscribe: update Bloom filter + delete Redis cache key

Bloom Filter Rebuild (every 60 seconds in each API instance):
  Background thread:
    SELECT url_hash FROM url_verdicts WHERE verdict = 'MALICIOUS'
    Build new BloomFilter in memory
    Atomic pointer swap: old_filter = current_filter; current_filter = new_filter
    Old filter GC'd after in-flight requests drain
```

---

## Detailed Components

### URL Normalization (Critical for Correctness)
```java
public String normalizeUrl(String rawUrl) {
    try {
        URI uri = new URI(rawUrl).normalize();

        String scheme = uri.getScheme().toLowerCase();
        String host = uri.getHost().toLowerCase();
        int port = uri.getPort();

        // Remove default ports
        if ((scheme.equals("http") && port == 80) ||
            (scheme.equals("https") && port == 443)) {
            port = -1;
        }

        // Remove tracking parameters
        String query = removeTrackingParams(uri.getQuery());

        // Remove fragment (# anchor; server never sees it)
        URI normalized = new URI(scheme, null, host, port,
                                 uri.getPath(), query, null);
        return normalized.toString().toLowerCase();

    } catch (URISyntaxException e) {
        // Malformed URL: treat as suspicious
        return rawUrl.toLowerCase().trim();
    }
}

private String removeTrackingParams(String query) {
    // Remove: utm_source, utm_medium, utm_campaign, fbclid, gclid, etc.
    Set<String> trackingParams = Set.of("utm_source", "utm_medium",
        "utm_campaign", "utm_term", "utm_content", "fbclid", "gclid");
    // parse query → filter out tracking → rebuild
    ...
}

// Why normalization is critical:
// http://Evil.COM/PATH → https://evil.com/path (same site, different form)
// https://evil.com/login?utm_source=email → https://evil.com/login (same destination)
// https://evil.com:443/login → https://evil.com/login (default port)
// Without normalization: attacker adds ?utm_source=random to bypass Bloom Filter
```

### Bloom Filter Implementation
```java
public class UrlBloomFilter {

    private final BitSet bitSet;
    private final int[] seeds;   // k different hash seeds
    private final int bitCount;

    // Optimal k (number of hash functions):
    //   k = (m/n) × ln(2)
    //   For m=720M bits, n=50M elements: k ≈ 10 hash functions
    //   More hash functions = lower FPR but slower (linear in k)

    public boolean mightContain(String urlHash) {
        for (int seed : seeds) {
            int position = Math.abs(murmur3(urlHash, seed)) % bitCount;
            if (!bitSet.get(position)) {
                return false;  // Definite MISS: at least one bit is 0
            }
        }
        return true;  // All bits are 1: probably in set (could be false positive)
    }

    public void add(String urlHash) {
        for (int seed : seeds) {
            int position = Math.abs(murmur3(urlHash, seed)) % bitCount;
            bitSet.set(position);
        }
    }

    // Cannot remove from a standard Bloom Filter (bit is shared by multiple entries)
    // For deletions: use Counting Bloom Filter or rebuild periodically
    // We rebuild every 60s → deletion handled by exclusion from rebuild query
}

// Memory math:
//   50M URLs, 0.1% FPR → m = 718M bits = ~90 MB
//   This is 90 MB per API instance for instant check of 50M malicious URLs
//   Redis: 50M keys × 50 bytes = 2.5 GB (28× more memory for same data)
//   PostgreSQL: 50M rows → must index → much larger; much slower
//   Bloom Filter is the right tool for this exact problem
```

### Tiered Lookup with Latency Budget
```
Each tier has a latency budget; exceeding it → skip to next tier or return UNKNOWN:

L1: Bloom Filter
  Latency: ~0.1ms (pure CPU; bitset reads)
  Outcome:
    MISS (bit not set) → return SAFE immediately (100% confidence; no false negatives)
    HIT (all bits set) → proceed to L2 (may be false positive)

L2: Redis Cache
  Latency: ~1ms (single GET)
  Timeout: 3ms (if Redis slow → skip to L3 directly)
  Outcome:
    HIT (MALICIOUS) → return MALICIOUS (cached verified verdict)
    HIT (SAFE_OVERRIDE) → return SAFE (admin override)
    MISS → proceed to L3

L3: PostgreSQL
  Latency: ~5ms (primary key lookup)
  Timeout: 10ms (if DB slow → return UNKNOWN and async-check)
  Outcome:
    HIT (MALICIOUS) → populate Redis → return MALICIOUS
    MISS → Bloom false positive confirmed → populate Redis (SAFE, 1h TTL) → return SAFE

External API (async, not in hot path):
  Google Safe Browsing, VirusTotal
  Called async for UNKNOWN verdicts
  Result updates DB + triggers Redis invalidation + Bloom rebuild
```

### Bloom Filter Rebuild Strategy
```
Why rebuild instead of incremental add?
  Standard Bloom Filters cannot delete entries (removing a bit affects other entries)
  Option A: Counting Bloom Filter (allows deletion, uses more memory)
  Option B: Periodic rebuild from DB (chosen)

Rebuild every 60 seconds:
  1. Background thread: SELECT url_hash FROM url_verdicts
     WHERE verdict = 'MALICIOUS' AND (expires_at IS NULL OR expires_at > NOW())
  2. Build new BloomFilter in memory (~90 MB allocation)
  3. Populate from result set (50M rows at ~100K rows/sec = 500 seconds)
     → Too slow! Use:
     Alternative: READ BINARY bloom filter from S3 (pre-built by ingestion pipeline)
     Ingestion pipeline rebuilds filter on every 1K new URLs → uploads to S3
     API instances download new filter from S3 every 60s (~90 MB download; fast)
  4. Atomic pointer swap: currentFilter.set(newFilter) (AtomicReference)
  5. In-flight requests using old filter drain; then old filter is GC'd

Freshness: new malicious URL → in DB → ingestion pipeline rebuilds filter → S3 upload
→ all API instances download within 60s → within 60s of DB write, URL is blocked
This meets the "blocked within 60 seconds" SLO.
```

---

## Scaling Strategy
```
Phase 1 (0 → 5K checks/sec):
  Single API instance + PostgreSQL
  No Bloom Filter; no Redis; direct DB lookup on every check
  Acceptable at low volume; 5ms per check is fine

Phase 2 (5K → 20K checks/sec):
  Add Redis verdict cache
  DB lookup only on cache miss
  Redis hit rate climbs to 80%+ for hot URLs

Phase 3 (20K → 100K checks/sec):
  Add Bloom Filter (in-process)
  99%+ of checks return in < 0.1ms (Bloom negative)
  Redis and DB only for Bloom positives (rare)
  Scale horizontally: multiple stateless API instances

Phase 4 (100K+ checks/sec or multi-region):
  Bloom Filter snapshot to S3 (shared across all instances)
  Edge deployment: Bloom Filter at CDN edge for zero-latency checks
  Regional DB replicas for L3 lookups

Phase 5 (global at scale):
  Distribute Bloom Filter as CDN edge worker asset
  Zero-RTT reputation check for 99.9% of URLs (known-safe)
  Cloud function for L2/L3 (Bloom positives only)
```

---

## Reliability Strategy
- **Bloom Filter in-process**: no network dependency for negative answers (99%+ of checks)
- **Redis down**: L2 skipped; falls to L3 (PostgreSQL); latency increases to ~5ms; no errors
- **PostgreSQL down**: return UNKNOWN (not SAFE); trigger async recheck; alert on-call
- **Threat feed disruption**: Bloom Filter serves last-known state; new malicious URLs not added; alert
- **False positive flood**: if FPR suddenly spikes → Bloom Filter rebuild may have bug; fall back to L3-only temporarily

---

## Security Considerations
- **No plaintext URLs in logs**: log url_hash, not raw URL (URL may contain PII in query params)
- **Redirect chain following**: follow up to 3 hops; detect infinite redirect loops; check intermediate URLs too
- **IDN homograph attack**: normalize Unicode domain names (punycode → ASCII equivalent); `аpple.com` ≠ `apple.com`
- **IP address URLs**: check URL patterns like `http://192.168.1.1/login` against known malicious IP ranges
- **Short URL expansion**: always expand short URLs (bit.ly, t.co) before reputation check; cache expansion results
- **API key per client**: each caller (email-service, content-service) has its own API key; rate-limited separately

---

## Observability
```
Metrics:
  check_total_per_sec              (throughput)
  check_latency_p99 by tier        (L1: target < 0.5ms; L2: < 2ms; L3: < 10ms)
  bloom_filter_hit_rate            (% of checks flagged by Bloom as "maybe malicious")
  redis_cache_hit_rate             (among Bloom positives; target > 70%)
  bloom_false_positive_rate        (confirmed: Bloom hit but DB miss; target < 0.1%)
  verdict_distribution             (% SAFE, % MALICIOUS, % UNKNOWN)
  threat_feed_ingestion_rate       (new malicious URLs added/sec)
  bloom_filter_age_seconds         (time since last rebuild; alert if > 120s)
  malicious_url_count              (total in DB; trend monitoring)

Security monitoring:
  malicious_checks_per_client      (which client is seeing most malicious URLs)
  new_malicious_domain_rate        (spike = active phishing campaign)
  false_positive_report_rate       (spike = possible bug in Bloom rebuild or feed error)

Alerts:
  bloom_filter_age > 120s          → rebuild stalled; new threats not propagating
  bloom_false_positive_rate > 0.5% → Bloom Filter misconfigured; excessive L3 load
  threat_feed_ingestion_stopped    → feed disconnected; new threats not ingested
  verdict_UNKNOWN > 5%             → L3 DB issue; external API not responding
  check_latency_p99 > 20ms         → tier cascade failing; DB overloaded
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Redis down | L2 skipped; all Bloom positives hit PostgreSQL; latency ~5ms | Alert; DB handles burst; auto-recover |
| PostgreSQL down | UNKNOWN verdict for Bloom positives | Return UNKNOWN; async recheck; alert |
| Bloom Filter rebuild failure | Stale filter; new threats not blocked for up to 2 min | Alert at 120s; S3 fallback snapshot |
| Threat feed disruption | New malicious URLs not ingested | Alert; last-known state is still protective |
| False positive storm (FPR spike) | Legitimate URLs flagged as MALICIOUS | Fallback to L3-only; false positive report queue |
| External API (GSB) unavailable | UNKNOWN for new URLs | Acceptable; internal feeds still active |
| S3 unavailable (Bloom snapshot) | Cannot rebuild Bloom; use last in-memory filter | Alert; filter ages but continues to work |

---

## Chaos Testing
```
Experiment 1: Bloom Filter Cold Start (instance restart)
  Inject:    Kill one API instance; restart it
  Expected:  On startup: download Bloom snapshot from S3 (< 30s for 90 MB)
  Expected:  During startup: instance not yet serving traffic (health check fails)
  Expected:  After filter loaded: health check passes; traffic routed to instance
  Red flag:  Instance serves traffic before Bloom Filter is loaded (would have high L3 load)

Experiment 2: Known-Bad URL Check Under Load
  Inject:    Add 100 malicious URLs to DB; send 50,000 check requests for them
  Expected:  After Bloom rebuild (60s): all checks caught at L1
  Expected:  Before Bloom rebuild: caught at L3 (DB lookup)
  Expected:  Check latency < 1ms for L1 hits; < 5ms for L3 hits
  Red flag:  Any malicious URL returns SAFE verdict (false negative)

Experiment 3: Bloom False Positive Flood
  Inject:    Manually set Bloom Filter FPR to 5% (misconfigured rebuild)
  Expected:  5% of checks incorrectly go to L2/L3
  Expected:  Redis and DB query rate spikes to 5× normal
  Expected:  bloom_false_positive_rate metric fires alert
  Expected:  Admin can revert to S3 snapshot with known-good filter
  Red flag:  5% of legitimate URLs return MALICIOUS (false positives returned to users)

Experiment 4: Threat Feed Disconnect
  Inject:    Stop Kafka consumer (threat feed ingestion)
  Expected:  No new malicious URLs added to DB
  Expected:  Bloom Filter age metric plateaus (no new rebuilds add new entries)
  Expected:  Alert fires within 5 minutes (threat_feed_ingestion_stopped)
  Expected:  URLs added to feeds during outage: ingested on reconnect (Kafka replay)
  Red flag:  Events from disconnected period are permanently lost

Experiment 5: URL Normalization Bypass Test
  Inject:    Check URLs in non-normalized form for known-malicious entries:
               http://EVIL.COM/phishing  (uppercase)
               https://evil.com/phishing?utm_source=test  (tracking param)
               https://evil.com:443/phishing  (explicit default port)
  Expected:  All forms normalize to same url_hash as evil.com/phishing
  Expected:  All correctly return MALICIOUS verdict
  Red flag:  Any non-normalized form returns SAFE (normalization bypass)
```

---

## Monthly Cost Estimate
```
Scale assumption: 50K checks/sec, 50M malicious URLs, 1B total URL history

API Instances:
  4 × m5.xlarge (4 vCPU, 16 GB RAM, $150/mo)   = $600/mo
  (Bloom Filter: 90 MB; 16 GB RAM leaves ample headroom)
  (CPU-bound: hash computations; 50K RPS / 4 instances = 12.5K/instance)

Redis (verdict cache):
  1 × cache.r6g.medium (13 GB, $60/mo)          = $60/mo
  (50 MB of hot verdicts; smallest node is fine)

PostgreSQL (verdict registry):
  1 × db.r6g.large (2 vCPU, 16 GB, $200/mo)    = $200/mo
  1 read replica for L3 lookups                  = $200/mo
  Total PostgreSQL:                              = $400/mo

Kafka (threat feed ingestion):
  3 × kafka.m5.large ($150/mo)                  = $450/mo

S3 (Bloom Filter snapshots):
  90 MB filter × $0.023/GB                      = $0.002/mo (negligible)

External API (Google Safe Browsing):
  Free tier: 10K lookups/day; paid: $200/mo for high volume

Monitoring + networking:
                                                 = $200/mo
────────────────────────────────────────────────────────────
Total: ~$1,710/month

Cost drivers:
  Kafka (26%): threat feed ingestion
  PostgreSQL (23%): verdict registry
  API instances (35%): CPU for hash computation

Optimisation levers:
  Bloom Filter at edge (CDN edge workers): 99% of checks never reach API → major savings
  Consolidate Kafka with existing event bus (share Kafka cluster): saves $450/mo
  Use managed Google Safe Browsing: cheaper than maintaining full URL registry at small scale
```

---

## Migration Story

### Current State
All link-checking done via real-time Google Safe Browsing API call on every URL render. 5–15ms added per link. Rate limits frequently hit during email blasts. No internal threat intel.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Add Redis verdict cache in front of GSB API (zero downtime)
  Action:
    On check: GET verdict:{url_hash} from Redis first
    On Redis miss: call GSB API; cache result with TTL (5 min)
    Effect: repeat URLs cached; GSB API calls reduced by ~80%
  Monitor:
    Redis hit rate (expect > 80% within hours)
    GSB API call rate (should drop proportionally)
  Rollback:
    Remove Redis check; revert to direct GSB API calls

Step 2: Build internal URL registry + PostgreSQL (zero downtime)
  Action:
    Set up PostgreSQL url_verdicts table
    Ingest GSB verdict into PostgreSQL on every GSB API call (write-through)
    Check order: Redis → PostgreSQL → GSB API
    PostgreSQL hit rate grows as registry fills
  Monitor:
    DB hit rate (should grow over days)
    GSB API calls continue to drop as registry fills
  Rollback:
    Remove PostgreSQL check; Redis → GSB only

Step 3: Add Bloom Filter (zero downtime)
  Action:
    Deploy API with in-process Bloom Filter
    Filter built from url_verdicts on startup + rebuilt every 60s
    L1 check before Redis: Bloom MISS → return SAFE immediately
    Monitor: Bloom hit rate, false positive rate
  Monitor:
    Bloom false positive rate (target < 0.1%)
    Redis hit rate among Bloom positives (should be high for hot URLs)
    GSB API calls should drop near zero (most traffic caught by Bloom or Redis)
  Rollback:
    Disable Bloom Filter; revert to Redis → DB → GSB

Step 4: Add threat feed ingestion (zero downtime, additive)
  Action:
    Subscribe to Kafka topic from threat intel team
    Consumer: normalize URL → UPSERT to PostgreSQL → trigger Bloom rebuild
    Internal threats now propagate in < 60s
  Monitor:
    Threat feed ingestion rate
    Time from feed event to Bloom filter inclusion (target < 60s)
  Rollback:
    Stop Kafka consumer; internal threats not ingested (fall back to GSB)

Step 5: Make GSB API optional (not primary) (zero downtime)
  Action:
    Internal registry is now comprehensive; GSB API for UNKNOWN only
    GSB API becomes a fallback for truly new, never-seen URLs
  Monitor:
    UNKNOWN verdict rate (should be < 1%)
    GSB API call rate (should be < 1% of total checks)
  Rollback:
    Re-enable GSB API as primary source for all non-cached checks
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL url_verdicts table (authoritative for all malicious URL data).
  → Bloom Filter: derived from PostgreSQL; rebuilt every 60s; not authoritative.
  → Redis: cache of PostgreSQL verdicts; not authoritative.
  → Bloom says "maybe malicious" → L2/L3 is always authoritative.

□ Consistency model?
  → Bloom Filter: eventually consistent (up to 60s lag after DB write).
  → Redis cache: eventually consistent (TTL-based; up to 5 min stale).
  → PostgreSQL: strongly consistent (authoritative source).
  → False negative risk: Bloom MISS for URL just added to DB (< 60s window).
    Mitigation: on URL add, also SET Redis verdict directly (immediate protection).

□ Failure mode?
  → Redis down → L2 skipped; Bloom positives fall to DB; latency ~5ms; no errors.
  → DB down → Bloom positives return UNKNOWN; accept temporary false negatives.
  → Bloom Filter stale → new threats not caught until rebuild; 60s max gap.
  → External API down → UNKNOWN for never-before-seen URLs; acceptable.

□ Hot partitions?
  → Viral phishing campaign: millions of checks for same URL → hot Redis key.
  → Mitigation: Redis handles millions of GET ops/sec; single key is fine.
  → Bloom Filter: in-process per instance; no shared state at L1.

□ Cost driver?
  → Kafka (26%): can consolidate with existing Kafka cluster to save $450/mo.
  → API instances (35%): driven by CPU for hash computation; Bloom Filter is efficient.

□ Security risk?
  → Normalization bypass: attacker uses non-normalized URL to avoid Bloom/Redis hash.
    Mitigation: normalization is mandatory and comprehensive; test thoroughly.
  → False positive abuse: attacker reports legitimate URL as malicious to censor it.
    Mitigation: false positive report requires human review; no auto-removal.
  → Bloom Filter poisoning: attacker controls threat feed; injects legitimate URLs.
    Mitigation: each feed source has a trust level; manual review for high-confidence overrides.

□ Migration path?
  → Direct GSB API → Redis cache → Internal DB → Bloom Filter → Internal-primary.
  → Each step reduces GSB dependency; builds internal capability.
```

---

## Staff Engineer Discussion Points

**"Why not use a Cuckoo Filter instead of a Bloom Filter?"**
Cuckoo Filters support deletion — you can remove entries without rebuilding. This would eliminate the 60-second rebuild cycle. Trade-off: Cuckoo Filters are slightly less memory-efficient at very low FPR (<0.1%) and have slightly higher insertion cost. For URL reputation, we rebuild from DB anyway (to pick up expiry-based purges and corrections), so the rebuild cycle is actually useful for correctness — it handles both additions and deletions. If real-time deletion were a strict requirement (e.g., legal takedown must block within 1 second, not 60), I'd reconsider Cuckoo Filters.

**"How do you handle false positives that are high-profile? (e.g., flagging bank.com)"**
Three-layer mitigation: (1) Allowlist: high-profile domains (Alexa top 10K) are explicitly whitelisted — never put in malicious DB; always return SAFE. (2) False positive report queue: user or admin can submit a report; reviewed within 1 hour; if confirmed: SAFE_OVERRIDE record added to DB, propagates to all instances via Redis Pub/Sub invalidation. (3) Confidence score: return `confidence: 0.95` with every MALICIOUS verdict; clients can display a softer warning UI instead of hard block for lower-confidence verdicts.

**"How would you design this for real-time streaming (< 1s freshness, not 60s)?"**
Remove the batch rebuild. On every DB write (threat ingested), directly update the in-process Bloom Filters across all instances via Redis Pub/Sub: "new malicious URL hash: {hash}". Each instance calls `bloom.add(hash)` on receiving the message. This gives sub-second propagation for additions. For deletions (expiry, false positive removal): still need periodic full rebuild — but rebuild daily instead of every 60s (since additions are handled via Pub/Sub). Trade-off: Pub/Sub requires each API instance to maintain a Redis subscription (connection overhead), but this is the same pattern as the Feature Flag Service (Case 08).

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "99% of checks are negative; fast negative required"  → Bloom Filter (L1)
  "Exact check needed for positives"                    → Tiered lookup (L2 Redis, L3 DB)
  "Never miss a known-bad URL"                          → Bloom Filter (zero false negatives)
  "1B URLs but only check membership"                   → Probabilistic filter (not hash set)
  "New threats propagate fast"                          → Kafka feed + periodic Bloom rebuild

BLOOM FILTER MATH:
  n entries, p false positive rate:
    m bits = -n × ln(p) / (ln(2))² 
    k hashes = (m/n) × ln(2)
  50M entries, 0.1% FPR: ~90 MB, ~10 hash functions
  1B entries, 0.1% FPR: ~1.8 GB, ~10 hash functions
  Standard Bloom: no deletion → rebuild to handle expiry

TIERED LOOKUP PATTERN:
  L1 (in-process): < 0.1ms — Bloom Filter (probabilistic; fast negative)
  L2 (Redis):      < 1ms   — Exact cache for Bloom positives
  L3 (DB):         < 5ms   — Authoritative for cache misses
  Short-circuit on first conclusive answer; only escalate if needed

PATTERNS REJECTED (and why):
  Hash set of all URLs      → 16 GB per instance; prohibitive at 1B URLs
  DB lookup on every check  → 5–50ms; unacceptable in page-load critical path
  External API only         → SPOF; rate limits; latency; no internal intel
  No URL normalization      → trivial bypass: add tracking param to any malicious URL

SCC LENS:
  STATE:
    Source of truth: PostgreSQL url_verdicts (durable; authoritative)
    Probabilistic state: Bloom Filter (90 MB in-process; derived from DB; rebuilt 60s)
    Hot cache: Redis verdict cache (small; TTL-managed; bridges L1 and L3)

  COORDINATION:
    Bloom rebuild: ingestion pipeline writes DB → rebuilds filter → uploads S3
                   API instances download from S3 every 60s → atomic swap
    Threat feed: Kafka fan-out from threat intel → DB ingestion → Bloom rebuild
    Cache invalidation: on DB write → SET Redis directly (immediate for hot URLs)

  CONCENTRATION:
    Hot malicious URL (viral phishing): Redis key hit millions of times → fine (GET is O(1))
    Bloom Filter: in-process per instance; no shared concentration at L1
    DB: Bloom false positive rate < 0.1% → only 0.1% of 50K checks = 50 DB queries/sec

INTERVIEW ANSWER TRIGGER:
  "Design a URL reputation/safety check service" →
    1. Spot signals: high volume, 99% negative, fast response, zero false negatives
    2. Bloom Filter L1 (fast negative); Redis L2 (hot positives); DB L3 (cold)
    3. URL normalization mandatory (bypass prevention)
    4. Probabilistic filter choice: 90 MB for 50M URLs vs 16 GB for hash set
    5. Freshness: Kafka feed → DB → Bloom rebuild every 60s
    6. SCC: State=PostgreSQL+Bloom+Redis; Coordination=Kafka+rebuild cycle;
            Concentration=viral URL → Redis hot key (fine at scale)
```

---

*Next: Case Study 10 — Audit Logging Service*
