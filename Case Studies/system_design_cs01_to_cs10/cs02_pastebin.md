# Case Study 02 — Pastebin

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Recognition Framework, Rejected Patterns, Migration Story.

---

## Business Context

Pastebin is a developer and collaboration tool. Users paste code snippets, config files, error logs, and share them via a URL. The business model: free tier (public pastes, size-limited), paid tier (private pastes, longer retention, API access, custom expiry).

The product promise is: **the content you saved is there when you need it**. Durability and read availability outweigh write speed. Storage cost matters because pastes can be large (up to 10 MB) and retention can be long (years for paid users).

**Business goals driving architecture:**
- Content availability is the primary SLO — a paste that disappears is a broken product
- Storage cost is a real constraint at scale — object storage, not DB columns
- Public pastes should load fast globally — CDN is the right answer
- Private pastes need access control — cannot use public CDN URLs
- Syntax highlighting is cosmetic — compute client-side, store language hint server-side only

---

## Recognition Framework

### Signals from the Problem
```
- Content is variable-size (a few bytes to 10 MB) — DB TEXT columns are wrong
- Public content is read-heavy and immutable after creation — cacheable at edge
- Private content needs auth-gated access — cannot be a public URL
- Content does not change after creation — no update semantics needed
- Expiry is a core feature — background cleanup process required
- Low write volume (~12/sec) vs moderate reads (~115/sec) — not write-heavy
- Avg paste ~10 KB, max 10 MB — blob storage fits perfectly
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Blob Storage (S3) | Variable-size content up to 10 MB — DB rows cannot handle this efficiently |
| CDN | Public pastes: immutable, globally cacheable, served from edge |
| ID Generation | Unique paste key, URL-safe, short enough to share |
| Expiry Worker | Background cleanup: delete S3 + DB row + cache + CDN invalidation |
| Presigned URLs | Private pastes: time-limited access without proxying large content through API |
| Redis Metadata Cache | Hot paste metadata avoids DB roundtrip on every read |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Store content in PostgreSQL TEXT column | 10 MB rows destroy index performance; makes DB backups enormous; no CDN integration possible |
| Redis for content storage | RAM cost prohibitive for 10 MB blobs; no native CDN integration |
| Kafka for write path | Write volume (12/sec) is trivially low — async decoupling adds complexity with no benefit |
| CQRS | Read model is just metadata + S3 fetch; no complex projection needed |
| Event Sourcing | Paste content is immutable after creation — no state evolution to source |
| File server (NFS/local disk) | Single point of failure; no durability guarantee; no CDN integration; ops burden |

### Why S3 and Not a File Server?
```
File server problems:
  - Single point of failure (unless replicated, which is complex)
  - No built-in CDN integration
  - No lifecycle management (auto-delete after N days)
  - No presigned URL support (need to build access control layer)
  - Disk management is an ops burden

S3 advantages:
  - 11 nines durability (99.999999999%)
  - Native CloudFront CDN integration
  - Lifecycle rules: auto-transition to Glacier, auto-delete after N days
  - Presigned URLs: time-limited, cryptographically signed, client fetches directly
  - Scales to petabytes without ops intervention
  - Multipart upload: handles large files reliably
```

### Why Presigned URLs for Private Pastes?
```
Option A: API proxies content (proxy pattern)
  Client → API → S3 → content → API → Client
  Problems:
    - API must buffer 10 MB in memory per request
    - API bandwidth is the bottleneck
    - API becomes expensive at scale (egress × content size)

Option B: Presigned URL (chosen)
  Client → API (auth check) → API returns presigned URL → Client fetches from S3 directly
  Benefits:
    - API never touches content bytes
    - S3 handles the bandwidth
    - Presigned URL expires in 15 minutes (cannot be shared long-term)
    - No change to S3 bucket policy (stays private)
```

---

## Problem Statement

Users paste text content (code, logs, config) and receive a unique URL. Visitors read the paste via that URL. Pastes can be public or private, and can optionally expire.

---

## Functional Requirements
- Create paste (content, title, language hint, expiry, private flag)
- Read paste by unique key
- Delete paste (owner only)
- Optional: password-protected pastes
- Optional: syntax highlighting (language stored as metadata; rendering is client-side)
- Optional: paste versioning (for paid tier)

## Non-Functional Requirements
- Read/write ratio: **10:1**
- Read latency: **< 50ms p99** (content can be up to 10 MB)
- Availability: **99.9%**
- Durability: pastes must not be lost — S3 = 11 nines
- Storage: pastes up to 10 MB; average ~10 KB
- Retention: free tier 30 days; paid tier up to 10 years

---

## Capacity Estimation
```
Writes:
  1M new pastes/day = ~12 writes/sec (very low)

Reads:
  10M reads/day = ~115 reads/sec

Storage:
  Average paste: 10 KB
  1M pastes/day × 365 days × 10 KB = 3.65 TB/year

  Distribution (rough):
    80% of pastes < 1 KB  (short code snippets, notes)
    15% of pastes 1–100 KB (larger files, logs)
    5%  of pastes > 100 KB (large files, up to 10 MB)

Bandwidth:
  Read:  115 RPS × 10 KB avg = ~1.15 MB/s (mostly CDN hits; negligible)
  Write: 12 RPS × 10 KB avg  = ~120 KB/s (trivial)

Cache sizing:
  Hot metadata (top 10% of pastes drive 90% of reads):
  100K hot pastes × 1 KB metadata = 100 MB → easily fits in Redis
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Blob Storage (S3) | Variable-size content; durable; CDN-integrated; lifecycle management |
| CDN | Public pastes: immutable, globally cacheable; most reads never hit origin |
| ID Generation | Unique paste key; Base62 of Snowflake ID |
| Expiry Worker | Coordinated cleanup: S3 + PostgreSQL + Redis + CDN invalidation |
| Presigned URLs | Private paste access without proxying content through API |
| Redis Metadata Cache | Avoid DB hit for every read; hot paste metadata in memory |

---

## Data Model

```sql
-- Metadata in PostgreSQL (small, fast, indexable)
CREATE TABLE pastes (
  paste_key     VARCHAR(8)    PRIMARY KEY,
  title         VARCHAR(256),
  user_id       BIGINT,
  language      VARCHAR(32),           -- syntax hint: 'python', 'json', etc.
  is_private    BOOLEAN       DEFAULT FALSE,
  password_hash VARCHAR(256),          -- NULL if not password-protected
  storage_path  TEXT          NOT NULL, -- S3 key: pastes/{paste_key}.txt
  size_bytes    INTEGER,
  created_at    TIMESTAMP     DEFAULT NOW(),
  expires_at    TIMESTAMP,             -- NULL = never expires
  is_deleted    BOOLEAN       DEFAULT FALSE  -- soft delete (recovery window)
);

CREATE INDEX idx_pastes_user_id ON pastes(user_id);
CREATE INDEX idx_pastes_expires_at ON pastes(expires_at)
  WHERE expires_at IS NOT NULL AND is_deleted = FALSE;

-- Content stored in S3, NOT in DB
-- S3 key structure:
--   Public:  pastes/public/{paste_key}.txt   (public-read ACL → CDN can cache)
--   Private: pastes/private/{paste_key}.txt  (private ACL → presigned URL required)
```

**Key decisions:**
- Metadata in PostgreSQL: indexed, queryable, small rows
- Content in S3: avoids DB bloat; enables CDN; lifecycle management; presigned URLs
- Separate public and private S3 prefixes: different ACL policies; CDN only touches public prefix
- Soft delete (`is_deleted`): gives a recovery window before hard delete from S3

---

## API Design

```
POST /api/v1/pastes
Body: {
  "content": "def hello():\n    print('world')",
  "title": "Hello World",
  "language": "python",
  "expires_in_hours": 24,
  "is_private": false,
  "password": null
}
Response 201: {
  "paste_key": "aB3xQ",
  "paste_url": "https://paste.ly/aB3xQ",
  "expires_at": "2024-01-02T14:00:00Z"
}

GET /api/v1/pastes/{paste_key}
  (for public paste)
Response 200: {
  "paste_key": "aB3xQ",
  "title": "Hello World",
  "content": "def hello():\n    print('world')",
  "language": "python",
  "size_bytes": 41,
  "created_at": "2024-01-01T14:00:00Z",
  "expires_at": "2024-01-02T14:00:00Z",
  "view_count": 142
}
Response 404: { "error": "not_found" }
Response 410: { "error": "expired" }
Response 401: { "error": "private_paste" }  (auth required)

GET /api/v1/pastes/{paste_key}/raw
Response 200: plain text body (Content-Type: text/plain)
(for public pastes: CDN-cacheable; for private: presigned S3 URL redirect)

DELETE /api/v1/pastes/{paste_key}
Response 204 (soft delete: sets is_deleted = TRUE)

POST /api/v1/pastes/{paste_key}/unlock
Body: { "password": "secret" }
Response 200: { "content": "..." }  (for password-protected pastes)
```

---

## High-Level Architecture

```
Client
  │
  ▼
CDN (caches public pastes by paste_key, TTL = min(expires_at, 1h))
  │  Only public pastes are cached here; private pastes bypass CDN
  ▼
Load Balancer
  │
  ▼
API Service (stateless, N instances)
  │                         │
  ▼                         ▼
PostgreSQL              S3 / GCS
(metadata:              (paste content:
 paste_key, title,       pastes/public/{key}.txt
 language, expiry,       pastes/private/{key}.txt)
 storage_path)
  │
  ▼
Redis
(paste metadata cache:
 paste:{key} → metadata JSON
 TTL = min(expires_at, 1h))

Background Services:
  Expiry Worker (runs every 5 min):
    SELECT paste_key FROM pastes
    WHERE expires_at < NOW() AND is_deleted = FALSE
    → soft delete in DB
    → DELETE object from S3
    → DEL from Redis
    → CDN invalidation (batch)
```

---

## Detailed Components

### Write Path
```
1. Client POSTs content to API
2. Validate: size < 10 MB; content is valid UTF-8
3. Generate paste_key (Base62 of Snowflake ID → 7 chars)
4. Determine S3 prefix based on is_private flag
5. PUT content to S3: pastes/{public|private}/{paste_key}.txt
   → S3 PUT is idempotent: same key, same content = no problem on retry
6. INSERT metadata row to PostgreSQL
7. Return paste_key to client

Failure handling:
  If S3 PUT succeeds but DB INSERT fails:
    → Retry DB INSERT with same paste_key (idempotent)
    → S3 PUT is idempotent: re-PUT is safe
  If S3 PUT fails:
    → Return 503; do not write to DB
    → Client retries entire operation
```

### Read Path (Public Paste)
```
1. Client GET /api/v1/pastes/{paste_key}
2. CDN check (for /raw endpoint):
   → HIT: return cached content directly (< 1ms; no server involved)
   → MISS: proceed to API
3. Redis check: GET paste:{paste_key}
   → HIT: return cached metadata
         For content: CDN URL (public) or generate presigned URL (private)
   → MISS: proceed to PostgreSQL
4. PostgreSQL SELECT by paste_key (primary key lookup, < 5ms)
5. Check is_deleted → return 404
   Check expires_at → return 410
6. Populate Redis: SET paste:{paste_key} {metadata} TTL=3600
7. Return metadata + content URL

Content delivery:
  Public paste:  Return CDN URL (https://cdn.paste.ly/{paste_key}.txt)
                 Client fetches from CDN directly
  Private paste: Generate S3 presigned URL (15-min TTL)
                 Return presigned URL to client
                 Client fetches from S3 directly
                 API never touches content bytes
```

### Read Path (Private Paste)
```
1. Verify authentication (session token or API key)
2. PostgreSQL lookup: verify user_id matches paste owner
   (or check shared-access table for collaborators)
3. If password_hash is set:
   → Require POST /unlock with password before viewing
   → bcrypt.verify(password, password_hash)
4. Generate S3 presigned URL:
   s3.presign('GET',
     bucket='paste-content',
     key='pastes/private/{paste_key}.txt',
     expires_in=900)  # 15 minutes
5. Return presigned URL to client
6. Client fetches content directly from S3
   Presigned URL cannot be shared beyond 15-min expiry
```

### Expiry Worker
```
Runs every 5 minutes as a scheduled job (separate process or cron):

1. Query: SELECT paste_key, storage_path FROM pastes
          WHERE expires_at < NOW()
          AND is_deleted = FALSE
          LIMIT 1000

2. For each expired paste (in batches of 100):
   a. SET is_deleted = TRUE in PostgreSQL (soft delete first)
   b. DELETE S3 object at storage_path
   c. DEL Redis key: paste:{paste_key}
   d. Queue CDN invalidation: /paste_key (batch invalidate)

3. Repeat until no more expired pastes

Properties:
  - Idempotent: running twice for same paste is safe
    (S3 DELETE is idempotent; DB UPDATE is idempotent)
  - Soft delete first: gives recovery window if needed
  - Batched CDN invalidation: avoid hitting CloudFront API rate limits
  - Heartbeat metric: emit expiry_worker_last_run timestamp
    Alert if > 10 minutes since last run
```

### Multipart Upload for Large Pastes
```
For pastes > 5 MB (S3 multipart threshold):

1. Client → API: POST /api/v1/pastes/initiate
   Response: { "upload_id": "...", "paste_key": "aB3xQ" }

2. Client → S3 (directly, via presigned part URLs):
   For each 5 MB chunk:
   PUT {presigned_part_url} with chunk data

3. Client → API: POST /api/v1/pastes/{paste_key}/complete
   Body: { "upload_id": "...", "parts": [...] }
   API finalises multipart upload with S3
   API writes metadata to PostgreSQL

Benefits:
  - Client uploads directly to S3 (not through API)
  - Resumable: if one part fails, only retry that part
  - API handles only metadata; never touches content bytes
```

---

## Scaling Strategy
```
Phase 1 (0 → 100 RPS):
  Single API instance + PostgreSQL + S3
  No cache; every read hits DB + S3 fetch
  Works fine; write volume is trivially low

Phase 2 (100 → 1K RPS):
  Add Redis metadata cache
  Cache hit rate climbs to 80%+ for hot pastes
  Add CDN for public paste content (/raw endpoint)

Phase 3 (1K → 10K RPS):
  Horizontal API scaling (stateless)
  PostgreSQL read replicas
  Expiry worker as separate service (not cron in API process)

Phase 4 (10K+ RPS):
  S3 Transfer Acceleration for global upload performance
  Multi-region S3 replication for global read latency
  CDN covers majority of public paste reads (origin rarely hit)

Phase 5 (storage at scale):
  S3 lifecycle rules: pastes > 1 year → S3-IA; > 3 years → Glacier
  Partition PostgreSQL pastes table by created_at (monthly partitions)
  Archive old metadata to cold storage; keep recent 90 days hot
```

---

## Reliability Strategy
- **S3 durability**: 11 nines — content loss is practically impossible
- **DB backup**: daily snapshots + WAL streaming replication to standby
- **Idempotency on write**: S3 PUT is idempotent; DB INSERT uses paste_key as primary key
- **Soft delete**: `is_deleted = TRUE` before hard delete — 24h recovery window
- **Expiry worker**: idempotent; re-running cleanup is always safe; heartbeat monitoring
- **Large file uploads**: multipart upload with retry per-part; resumable on failure

---

## Security Considerations
- **Private pastes**: not in CDN; require authentication on every access; presigned URL expires in 15 min
- **Password-protected pastes**: bcrypt hash stored; client POSTs password to unlock; no plaintext storage
- **Content scanning**: S3 event → Lambda → malware/CSAM scanner on every upload; flag → quarantine
- **Rate limiting on uploads**: max 100 paste creations/hour per account; max 10 MB per paste (enforced at API)
- **Presigned URL abuse**: presigned URL cannot be reused after expiry; cannot be used from different IP if IP binding enabled
- **S3 bucket policy**: API service role has GET/PUT/DELETE; no public access to private prefix; CDN has GET on public prefix only
- **SSRF prevention**: if pastes can contain URLs, do not auto-fetch/render them server-side

---

## Observability
```
Metrics:
  paste_create_rate              (writes/sec; should be ~12 at expected load)
  paste_read_rate                (reads/sec by type: public/private)
  s3_upload_latency_p99          (target < 500ms for avg paste)
  redis_cache_hit_rate           (target > 80% for metadata)
  cdn_offload_rate               (% of public paste reads served from CDN)
  expiry_worker_last_run_seconds (alert if > 600s; worker stopped)
  expired_pastes_pending_cleanup (gauge; how far behind the worker is)

Logs (structured JSON, every request):
  paste_key, operation (create/read/delete), latency_ms,
  cache_tier (cdn/redis/db/miss), size_bytes, is_private,
  status_code, request_id, user_id

Alerts:
  s3_upload_error_rate > 1%              → S3 connectivity issue
  expiry_worker_last_run > 10 min        → worker stopped; pastes not being cleaned up
  redis_cache_hit_rate < 60%             → Redis undersized or evicting too aggressively
  s3_upload_latency_p99 > 2000ms         → S3 regional issue or paste too large
  db_connection_pool_exhausted           → scale API instances or increase pool size
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| S3 unavailable | Writes fail (503); reads fail for cache misses | CDN serves cached public pastes; accept writes fail |
| DB primary down | Metadata unavailable for cache misses | Redis serves hot metadata; failover to replica |
| Redis down | All reads fall to DB + S3; latency increases | Acceptable degradation; DB handles it; alert |
| Expiry worker crashes | Expired pastes remain accessible past expiry | Heartbeat alert; idempotent reprocess on restart |
| Large paste upload timeout | Write fails mid-stream | Multipart upload: only failed part retries; resume |
| CDN misconfiguration | Private pastes accidentally cached | S3 prefix separation; CDN only touches public prefix |
| Content scanner false positive | Legitimate paste quarantined | Human review queue; 24h SLA; appeal process |

---

## Chaos Testing
```
Experiment 1: Kill S3 (simulate regional outage)
  Inject:    Block all outbound S3 API calls from API service
  Expected:  CDN serves all hot public pastes (no degradation for cached content)
  Expected:  New paste creation fails with 503 (write path requires S3)
  Expected:  Private paste reads fail (presigned URL generation fails)
  Expected:  API returns 503 with Retry-After header; no crashes
  Red flag:  API crashes instead of returning 503

Experiment 2: Kill Redis
  Inject:    Stop Redis instance
  Expected:  All reads fall to PostgreSQL + S3 fetch
  Expected:  Read latency increases from ~5ms to ~50ms (still within SLO)
  Expected:  No errors; service fully functional at degraded performance
  Red flag:  Service returns 5xx errors (Redis should not be in critical path)

Experiment 3: Kill Expiry Worker
  Inject:    Kill expiry worker process
  Expected:  Expired pastes remain accessible (past their expiry time)
  Expected:  Alert fires within 10 minutes (heartbeat metric breach)
  Expected:  On worker restart: idempotently picks up all pending expired pastes
  Red flag:  Worker restarts and re-deletes already-deleted pastes (not idempotent)

Experiment 4: Flood of Large Paste Uploads
  Inject:    100 concurrent uploads of 10 MB pastes
  Expected:  Multipart upload handles concurrency; no OOM in API
  Expected:  S3 handles parallel uploads without issue
  Expected:  DB connection pool not exhausted (only metadata writes; small)
  Red flag:  API OOM (API is buffering content instead of streaming to S3)

Experiment 5: Private Paste Shared to Unauthorised User
  Inject:    Obtain a presigned URL for a private paste; wait 20 minutes; try to use it
  Expected:  S3 returns 403 (presigned URL expired after 15 minutes)
  Expected:  Generate a new presigned URL → still requires authentication
  Red flag:  Expired presigned URL still works (S3 clock skew issue)
```

---

## Monthly Cost Estimate
```
Scale assumption: 1M pastes/day, 10M reads/day, 3.65 TB stored after 1 year

API Service:
  2 × t3.medium ($34/mo)                       = $68/mo
  (write volume is trivially low; read volume manageable)

PostgreSQL:
  1 primary + 1 read replica × db.t3.medium    = $200/mo

Redis:
  1 × cache.t3.medium ($30/mo)                 = $30/mo
  (100 MB of hot metadata; single node fine)

S3 Storage:
  3.65 TB × $0.023/GB                          = $84/mo
  (first year; grows linearly)

S3 Lifecycle (after year 1):
  > 1 year → S3-IA: $0.0125/GB                = ~$46/mo
  > 3 years → Glacier: $0.004/GB              = ~$15/mo

S3 Requests:
  10M reads/day × 30 × $0.0004/1000 GET        = $120/mo
  1M writes/day × 30 × $0.005/1000 PUT         = $150/mo

CDN (CloudFront):
  ~5 TB/month egress × $0.008/GB               = $40/mo
  (most public paste reads served from CDN; S3 GETs saved)

Expiry Worker:
  Lambda or small EC2 t3.micro                  = $10/mo

Monitoring + misc:
                                                = $100/mo
────────────────────────────────────────────────────────────
Total: ~$802/month (year 1)
       ~$600/month (year 2+ with lifecycle transitions)

Cost drivers:
  S3 requests: 34% of total (PUT + GET)
  PostgreSQL:  25% of total
  CDN:         5% (cheap because most content is small)

Optimisation levers:
  - CDN caching reduces S3 GET requests dramatically
    (10M reads/day → CDN serves 90% → only 1M hits S3 → saves ~$108/mo)
  - S3 lifecycle rules save ~25% on storage after year 1
  - Reserved instances for PostgreSQL save ~30%
  - Compress content before S3 upload (gzip): reduces storage and egress cost
    (average 10 KB paste → ~3 KB compressed = ~70% storage savings)
```

---

## Migration Story

### Current State
Single server. Paste content stored on local disk at `/var/pastes/{paste_key}.txt`. PostgreSQL for metadata. Working fine at 10K pastes/day but disk is filling up, no HA, and there's no CDN.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Move content from local disk to S3 (zero downtime)
  Action:
    Update write path: new pastes go to S3 (keep local disk write as fallback)
    Update read path: try S3 first; if not found, try local disk (for old pastes)
    Run background migration job: upload existing /var/pastes/* to S3
      - For each file: PUT to S3 → verify ETag → mark migrated in DB
      - Idempotent: re-running safe
    Once all files migrated: remove local disk fallback from read path
    Once confirmed: remove local disk write from write path
  Monitor:
    Migration job progress (files remaining)
    S3 upload error rate
    Read path: s3_hit_rate vs local_disk_hit_rate
  Rollback:
    Re-enable local disk write as primary; S3 write as secondary
    Local disk is still intact until explicitly deleted

Step 2: Add Redis metadata cache (zero downtime)
  Action:
    Deploy Redis (single node initially)
    Update read path: check Redis before PostgreSQL
    Populate cache on miss; TTL = min(expires_at, 3600)
  Monitor:
    Redis cache hit rate (expect to climb to 80%+ within 1 hour)
    PostgreSQL read volume (expect to drop significantly)
  Rollback:
    Remove Redis check from read path; re-deploy
    PostgreSQL handles all reads (back to baseline)

Step 3: Add CDN for public paste content (zero downtime)
  Action:
    Create CloudFront distribution pointing to S3 public prefix
    Update public paste read path to return CDN URL instead of S3 URL
    Set Cache-Control headers on S3 objects: max-age=3600
  Monitor:
    CDN offload rate (% of reads served from edge)
    S3 GET request count (should drop significantly)
  Rollback:
    Remove CloudFront distribution
    Return S3 presigned URLs directly to clients (or make public S3 URLs)
    DNS change propagates within 5 minutes

Step 4: Add expiry worker as dedicated service (zero downtime, additive)
  Action:
    Deploy expiry worker as separate process (Lambda on schedule or EC2)
    Worker: SELECT expired pastes → soft delete → S3 delete → cache delete → CDN invalidate
    Add heartbeat monitoring
  Monitor:
    expiry_worker_last_run metric
    expired_pastes_pending_cleanup gauge
  Rollback:
    Stop expiry worker (pastes linger past expiry; no data loss)
    Manually clean up if needed

Step 5: Add PostgreSQL read replica + Redis Sentinel (5-min maintenance window)
  Action:
    Provision read replica; update read path to use replica for SELECTs
    Upgrade Redis to Sentinel (3-node: 1 primary + 2 replicas + 3 sentinels)
    Test failover: kill primary → verify < 5s failover
  Monitor:
    Replica lag (should be < 100ms)
    Redis Sentinel failover time
  Rollback:
    Route reads back to primary
    Point Redis client back to standalone Redis
```

**Key lesson:** Each step solves a specific problem. Step 1 solves durability and disk capacity. Step 2 solves read latency. Step 3 solves global read performance and cost. Step 4 solves data hygiene. Step 5 solves HA. Never conflate these into one big migration.

---

## Architecture Review Checklist
```
□ Source of truth?
  → S3 for paste content (authoritative; 11 nines durability).
  → PostgreSQL for metadata (authoritative; content location, expiry, access control).
  → Redis is a metadata cache — expendable; always reconstructable from DB.

□ Consistency model?
  → Write path: strong (S3 PUT + DB INSERT must both succeed; retry if DB fails).
  → Read path for metadata: eventual (Redis cache; up to 1h stale).
  → CDN cache for content: eventual (up to 1h stale for public pastes).
  → Private paste access: strong (auth check + presigned URL generated fresh each time).

□ Failure mode?
  → S3 down: CDN serves hot public pastes; writes fail (503); private reads fail.
  → DB down: Redis serves hot metadata for reads; writes fail.
  → Redis down: all reads fall to DB + S3 fetch; latency degrades; service stays up.
  → Expiry worker down: expired pastes linger; alert fires; no data loss.

□ Hot partitions?
  → Viral public paste → hot CDN edge node → S3 not in the critical path.
  → High-traffic paste → hot Redis key → GET/SET on single key; Redis handles this.
  → No write contention (each paste is a different key; no shared mutable state).

□ Cost driver?
  → S3 requests (34% of total at this scale).
  → CDN caching dramatically reduces S3 GETs; most important cost lever.
  → S3 lifecycle rules reduce storage cost after year 1.

□ Security risk?
  → Private paste inadvertently served publicly if S3 prefix is misconfigured.
    Mitigation: strict S3 bucket policy; CDN only touches public prefix.
  → Presigned URL shared beyond its window.
    Mitigation: 15-min expiry; IP-binding optional for high-sensitivity pastes.
  → Malicious content uploaded (malware, CSAM).
    Mitigation: async content scanning via S3 event trigger.

□ Operational burden?
  → Expiry worker: must be monitored; heartbeat alert; idempotent reprocessing.
  → S3 lifecycle rules: set once; manage automatically.
  → CDN cache invalidation: required on delete; batch to avoid rate limits.
  → PostgreSQL partitioning (at scale): monthly partitions by created_at.

□ Migration path?
  → Local disk → S3 (Step 1) → Redis cache (Step 2) → CDN (Step 3)
  → Expiry worker (Step 4) → HA (Step 5)
  → Each step independently rollbackable; no big-bang migrations.
```

---

## Staff Engineer Discussion Points

**"What if a paste goes viral — 10M reads in 1 hour?"**
CDN absorbs it entirely. CloudFront edge nodes cache the content globally. S3 is not in the critical path for public pastes once CDN is warm. API service is not involved at all. The only concern is CDN cache miss on the first request per edge node — at 10M reads, that's negligible. S3 itself also scales horizontally with no ops intervention.

**"How do you handle GDPR right-to-erasure?"**
User requests deletion → we have 30 days to comply. Steps: (1) `is_deleted = TRUE` in PostgreSQL, (2) DELETE S3 object (both public and private prefixes), (3) DEL Redis cache entry, (4) CDN invalidation. Keep an audit log of the deletion (user ID, paste_key, deletion timestamp) in a separate immutable audit table. Important: CDN cached copies may persist for up to TTL after deletion — for GDPR this is acceptable as the CDN invalidation is near-immediate (CloudFront invalidation < 60s).

**"How would you implement paste versioning for paid users?"**
Each version is a separate S3 object: `pastes/{paste_key}/v{version_number}.txt`. PostgreSQL stores a `current_version` pointer and a `paste_versions` table (paste_key, version, storage_path, created_at). Read path: fetch `current_version` metadata → fetch correct S3 object. Diff view: fetch two versions from S3; compute diff client-side. Version count can be capped per tier (e.g. 10 versions for paid).

**"What if S3 is temporarily unavailable during a write — how do you handle partial failure?"**
S3 PUT succeeds but DB INSERT fails: the S3 object is an orphan. Retry the DB INSERT (paste_key is idempotent as PK). If retry also fails: add paste_key to a "pending DB writes" queue (Redis or SQS); background job retries. S3 object exists; no data lost; eventual DB consistency.

DB INSERT succeeds but S3 PUT fails: paste_key in DB but no content in S3. Client gets 503. On retry, client re-sends full request. API generates same paste_key (idempotent: same user + same content → same Snowflake ID window), re-attempts S3 PUT. The orphan DB row is either overwritten or the insert is idempotent via `ON CONFLICT DO UPDATE`.

**"How would you support team/organisation pastes (shared access)?"**
Add a `paste_access` table: (paste_key, user_id or team_id, permission_level). API checks this table in addition to owner check. For CDN: team pastes that are internal-only (not public, not private) get a short-lived presigned URL approach — same as private pastes. CDN cannot serve these (auth required). Alternatively: generate a team-scoped CDN URL with signed cookie (CloudFront signed cookies for group access).

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Variable-size content (up to 10 MB)"     → Blob Storage: S3 + metadata in DB
  "Public content, globally cacheable"      → CDN: CloudFront on S3 public prefix
  "Private content needs access control"    → Presigned URLs: time-limited, S3-signed
  "Content expires after N days"            → Expiry Worker: coordinated cleanup
  "Avoid proxying large content through API"→ Presigned URLs: client fetches S3 directly

DECISION TREE USAGE:
  Store large content?                      → Object store (S3), not DB TEXT columns
  Public and globally readable?             → CDN caching (immutable content = cacheable)
  Private with access control?              → Presigned URLs (not proxy pattern)
  Content that expires?                     → Background worker + soft delete
  Large file uploads (> 5 MB)?             → Multipart upload (resumable)

PATTERNS REJECTED (and why):
  PostgreSQL TEXT column  → 10 MB rows destroy index perf; no CDN integration
  Redis for content       → RAM too expensive for blobs; no CDN; wrong tool
  File server             → SPOF; no durability; no CDN; ops burden
  Kafka for writes        → 12 writes/sec is trivially low; overkill
  Proxy pattern           → API buffers 10 MB per request; bandwidth bottleneck

SCC LENS:
  STATE:
    Source of truth for content:  S3 (11 nines durability; object storage)
    Source of truth for metadata: PostgreSQL (indexed; queryable; authoritative)
    Cache (expendable):           Redis metadata cache; CDN content cache
    Both caches reconstructable from S3 + PostgreSQL at any time

  COORDINATION:
    Write order matters: S3 PUT before DB INSERT
    (S3 is idempotent → retry-safe; DB INSERT follows S3 success)
    Expiry cleanup: soft delete in DB first → then hard delete from S3
    (never delete S3 first; if DB soft-delete fails, paste appears deleted but S3 orphan exists)
    Cache invalidation on delete: DEL Redis → CDN invalidation
    (Redis DEL is immediate; CDN invalidation is < 60s)

  CONCENTRATION:
    Viral public paste → CDN absorbs (S3 and API not in hot path)
    Hot metadata (top 10% of pastes drive 90% of reads) → Redis absorbs
    S3 is horizontally scalable; no partition concern
    Write path is trivially low volume (12/sec); no write concentration

INTERVIEW ANSWER TRIGGER:
  "Design a Pastebin" →
    1. Spot the signals: variable-size content, public vs private, expiry
    2. Select: S3 (blob), CDN (public), Presigned URLs (private), Expiry Worker
    3. Reject with reason: DB TEXT (size), Redis (RAM), File server (SPOF)
    4. SCC: State=S3+PostgreSQL+Redis, Coordination=write order+soft delete,
            Concentration=CDN for viral public pastes
    5. Discuss tradeoffs: proxy vs presigned, soft delete window, TTL on cache
```

---

*Next: Case Study 03 — Rate Limiter*
