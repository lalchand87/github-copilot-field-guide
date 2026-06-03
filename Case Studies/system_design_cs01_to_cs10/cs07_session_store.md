# Case Study 07 — Session Store

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Opaque vs JWT decision, Fail-Closed reasoning, IAM Platform tie-in.

---

## Business Context

A session store is the infrastructure behind "stay logged in." Every request from an authenticated user requires a round-trip to verify identity. For a payment platform, session store failure means payment transactions fail mid-flow — not a degraded experience, but a total auth outage.

The session store must be treated with the same operational priority as the primary payments database. It is not a cache. It is authoritative for session state.

**Business goals driving architecture:**
- Session validation is in the hot path of every authenticated request — must be < 2ms
- Revocation must be instant: logout, forced logout, fraud-triggered session termination
- PCI-DSS compliance: payment sessions must have absolute timeout (not just idle)
- Multi-device: users logged in on 3–5 devices simultaneously
- Fail-CLOSED: unlike rate limiter, auth must fail if store is unreachable (availability < security)

---

## Recognition Framework

### Signals from the Problem
```
- Called on EVERY authenticated request → sub-millisecond required
- Revocation must be instant → cannot use self-contained JWT (cannot revoke before expiry)
- Multiple gateway instances need shared session state
- Session data is small (~500 bytes) and ephemeral
- Failure must fail-CLOSED: stale session = auth bypass = security hole
- Multi-device: one user has multiple concurrent sessions
- PCI-DSS: absolute 8h timeout regardless of activity
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Redis (Key-Value) | Sub-millisecond GET/SET; atomic DEL for instant revocation |
| Opaque Token | Must revoke instantly; no user data embedded in token |
| Sliding + Absolute Expiry | UX (idle timeout) + PCI-DSS (hard absolute limit) |
| Replication | HA — session store failure = effective mass logout |
| Strong Consistency (Primary reads only) | Stale session after forced logout = auth bypass risk |
| Fail-Closed | Unlike rate limiter: auth must fail if Redis unreachable |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| JWT for sessions | JWT cannot be revoked before expiry; blocklist to fix this defeats JWT's purpose |
| PostgreSQL as session store | 10–50ms per query; unacceptable at 100K validations/sec |
| In-process session HashMap | Multiple instances each have independent state; breaks shared auth |
| Read from Redis replicas | Replication lag: forced logout may not propagate before next request = auth bypass |
| Fail-open on Redis downtime | Unlike rate limiter: stale session = security hole; must fail-closed |

### Opaque Token vs JWT — The Core Decision

This is the most important decision in session store design. It determines everything downstream.

```
Opaque Token (server-side session — chosen for sessions):
  Format:    "sess_" + base64url(CSPRNG(32 bytes)) → 256-bit entropy
  Lookup:    GET session:{token} → session data (Redis; ~1ms)
  Revocation: DEL session:{token} → immediate; valid on next request check
  Data:      Stored in Redis (roles, device, expiry, created_at)
  Trade-off: Requires Redis on every authenticated request
  Use when:  Revocation must be instant (sessions, admin actions, fraud detection)

JWT (stateless — correct for short-lived ACCESS tokens):
  Format:    base64(header).base64(payload).signature
  Lookup:    Verify RS256 signature locally → extract claims (no network)
  Revocation: Cannot revoke before "exp" claim (expiry); need a blocklist to override
             → blocklist requires Redis → you now have Redis dependency anyway
  Data:      Claims embedded in token (roles, scopes, user_id)
  Trade-off: No network hop; stateless; cannot revoke instantly
  Use when:  Short-lived (< 15 min); revocation on expiry is acceptable

The correct pattern for a payment platform:
  Session token (opaque): long-lived (hours); revocable; stored in Redis
  Access token (JWT):     short-lived (15 min); stateless; issued by session token refresh
  
  Client uses session token to refresh access token every 15 min.
  Session revocation: DEL session token → next refresh fails → user re-authenticates.
  No need for JWT blocklist: access tokens expire naturally within 15 min.
  
This is how ForgeRock/Keycloak/Okta work:
  SSO session = opaque token in CTS/session store
  OIDC access token = short-lived JWT
  ForgeRock CTS (Core Token Service) is exactly the Redis session store described here.
```

### Why Fail-Closed (Unlike Rate Limiter)?

```
Rate Limiter (Case 03): FAIL-OPEN
  Redis down → allow all requests through
  Reasoning: rate limit accuracy is less important than API availability
             losing rate limiting for 5s doesn't cause data corruption
             payment transactions succeeding is more important

Session Store (Case 07): FAIL-CLOSED
  Redis down → return 401 to all authenticated requests
  Reasoning: accepting a session we cannot verify = potential auth bypass
             a forced-logout (fraud case) that can't be verified = attacker stays logged in
             5s of auth downtime is acceptable; fraud is not
             PCI-DSS requires: "Implement access control systems with fail secure default"

The difference: the consequence of the wrong failure mode.
  Rate limiter fail-closed: 5s of payment failures → business harm
  Session store fail-open: 5s of unverified auth → security harm

Always ask: "What does failure unlock that should be locked?"
If the answer is "access to data/money/PII" → fail-closed.
If the answer is "a bit of extra traffic" → fail-open.
```

---

## Problem Statement

Session store for a web and mobile application. Create session on login, validate on every authenticated request, invalidate on logout or forced logout. Support multi-device sessions with idle and absolute expiry.

---

## Functional Requirements
- Create session on login; return opaque session token
- Validate token on every authenticated request (< 2ms)
- Retrieve session data: user_id, roles, device_id, expiry metadata
- Invalidate: single session (logout) or all sessions for a user (forced logout)
- Sliding expiry: idle timeout resets on each active request
- Absolute expiry: hard limit regardless of activity (PCI-DSS: 8h for payment sessions)
- Multi-device: up to 5 concurrent sessions per user
- Session count enforcement: prevent session flooding attack

## Non-Functional Requirements
- Validation latency: **< 2ms p99**
- Availability: **99.99%**
- Throughput: **100,000 validations/sec**
- Consistency: **strong** — revoked session must not validate on next request
- Durability: session loss on restart is acceptable (users re-login)

---

## Capacity Estimation
```
Active sessions:
  10M active users; 10% concurrent = 1M active sessions
  Session size: ~500 bytes (user_id, roles array, device_id, timestamps)
  Memory: 1M × 500 bytes = 500 MB → fits in a single Redis node

Throughput:
  100,000 validations/sec → 100,000 Redis GET ops/sec
  Redis: 500K ops/sec per node → ample headroom (5 nodes = 2.5M ops/sec)

Multi-device index:
  Redis Set: user_sessions:{user_id} → set of session tokens
  Max 5 tokens per user × 32 bytes each = 160 bytes per user
  1M users with active sessions: 160 MB → trivially small
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Redis Key-Value | Sub-millisecond GET; atomic DEL for instant revocation |
| Opaque Token (CSPRNG) | Instant revocation; 256-bit entropy; no sensitive data in token |
| Sliding + Absolute Expiry | UX idle timeout + PCI-DSS compliance |
| Strong Consistency (Primary reads) | Revoked session must not validate; no replica reads |
| Fail-Closed | Auth must fail if Redis unreachable |

---

## Data Model

```
Primary session record:
  Redis key:   session:{session_token}
  Redis type:  Hash (field-value pairs; efficient partial reads)
  Fields:
    user_id              → "u-12345"
    email                → "user@example.com" (for logging; never in JWT payload)
    roles                → '["payments:read","payments:write"]' (JSON array)
    device_id            → "d-abc123"
    device_type          → "mobile_ios" / "desktop_browser" / "api_client"
    ip_at_login          → "203.0.113.42" (for fraud detection; not enforced)
    created_at           → "1712345600" (Unix timestamp; for absolute expiry check)
    absolute_expires_at  → "1712374400" (created_at + 8h; for PCI-DSS)
    last_active          → "1712349200" (updated on each request; for logging)
  TTL: 900 seconds (15 min idle sliding window; reset on each authenticated request)
  Absolute expiry: enforced by middleware (not Redis TTL) by checking absolute_expires_at field

Multi-device index:
  Redis key:   user_sessions:{user_id}
  Redis type:  Set
  Members:     [session_token_1, session_token_2, ...]
  TTL:         same as longest session for that user (or no TTL; cleaned up via session cleanup)

Session count enforcement:
  On login: SCARD user_sessions:{user_id}
  If >= 5: SMEMBERS → sorted by created_at → DEL oldest session → SREM from set
  Then: create new session + SADD to set
```

---

## API Design (Internal — Not Exposed to Clients)

```
Used by: Auth Middleware in API Gateway (Case 05) or per-service auth

POST /internal/v1/sessions
Body: {
  "user_id": "u-12345",
  "email": "user@example.com",
  "roles": ["payments:read", "payments:write"],
  "device_id": "d-abc123",
  "device_type": "mobile_ios",
  "ip_at_login": "203.0.113.42"
}
Response 201: {
  "session_token": "sess_aB3xQ...",  // 256-bit opaque token
  "expires_at": 1712349200,          // sliding: now + 15min
  "absolute_expires_at": 1712374400  // absolute: now + 8h (PCI)
}

GET /internal/v1/sessions/{session_token}
Response 200: {
  "user_id": "u-12345",
  "roles": ["payments:read", "payments:write"],
  "device_id": "d-abc123",
  "absolute_expires_at": 1712374400,
  "session_age_seconds": 3600
}
Response 401: { "error": "session_not_found_or_expired" }

PATCH /internal/v1/sessions/{session_token}/touch
Response 200: { "new_expires_at": 1712350100 }
(slides TTL; async — called after auth check succeeds)

DELETE /internal/v1/sessions/{session_token}
Response 204 (logout single session)

DELETE /internal/v1/sessions?user_id=u-12345
Response 204 (forced logout all sessions for user)
Body (response): { "sessions_terminated": 3 }

GET /internal/v1/sessions?user_id=u-12345
Response 200: {
  "active_sessions": [
    { "device_type": "mobile_ios", "created_at": 1712345600, "last_active": 1712349200 },
    { "device_type": "desktop_browser", "created_at": 1712340000, "last_active": 1712348000 }
  ]
}
```

---

## High-Level Architecture

```
Client (mobile app / browser)
  │  Cookie: sess_aB3xQ... (HttpOnly, Secure, SameSite=Strict)
  │  OR Header: Authorization: Bearer sess_aB3xQ...
  ▼
API Gateway (Case 05) → Auth Middleware
  │
  ├── Step 1: Extract token from cookie/header
  ├── Step 2: GET session:{token} from Redis PRIMARY (not replica)
  │          HIT  → Step 3
  │          MISS → return 401 immediately
  │
  ├── Step 3: Validate session
  │          Check: absolute_expires_at > now (PCI-DSS hard limit)
  │          If expired: DEL session:{token} + return 401
  │          If valid: inject user_id + roles into request context
  │
  ├── Step 4: Async (non-blocking): EXPIRE session:{token} 900
  │          Slide the idle TTL without blocking the request
  │          Update last_active field (HSET)
  │
  └── Step 5: Forward authenticated request to backend service
              Backend trusts X-Client-ID + X-User-Roles headers (from gateway)

Redis Sentinel (session store):
  1 primary + 2 replicas + 3 sentinel nodes
  Primary: all reads and writes (strong consistency; no replica reads)
  Sentinel: auto-failover if primary unreachable > 5s

On Redis unavailable:
  Auth middleware: catch connection error → return 401
  (Fail-closed: cannot verify session → cannot allow access)
  Alert: immediate page to on-call
  Expected: Sentinel failover resolves within 5s
```

---

## Detailed Components

### Token Generation (Security-Critical)
```java
public String generateSessionToken() {
    byte[] randomBytes = new byte[32];             // 256 bits
    SecureRandom.getInstanceStrong().nextBytes(randomBytes);  // CSPRNG
    String encoded = Base64.getUrlEncoder()
                           .withoutPadding()
                           .encodeToString(randomBytes);
    return "sess_" + encoded;
    // Result: "sess_" + 43 chars = 48-char token
    // 256-bit entropy: brute force attack impossible
    // (2^256 ≈ 10^77 attempts needed to find valid token)
}

// NEVER use:
//   UUID.randomUUID()   → only 122 bits of randomness (UUID v4)
//   Math.random()       → not cryptographically secure
//   sequential IDs      → guessable; enumeration attack
//   user_id + timestamp → predictable; allows targeted attacks
```

### Sliding + Absolute Expiry Implementation
```java
// Called on every authenticated request:
public AuthResult validateSession(String token) {
    // Step 1: Get session data (from PRIMARY — no replica reads)
    Map<String, String> session = redis.hgetall("session:" + token);

    // Step 2: Check key exists
    if (session == null || session.isEmpty()) {
        return AuthResult.INVALID; // Token not found or expired by Redis TTL
    }

    // Step 3: Check absolute expiry (PCI-DSS hard limit)
    long absoluteExpiry = Long.parseLong(session.get("absolute_expires_at"));
    if (System.currentTimeMillis() / 1000 > absoluteExpiry) {
        redis.del("session:" + token);                  // Clean up immediately
        redis.srem("user_sessions:" + session.get("user_id"), token);
        return AuthResult.EXPIRED;
    }

    // Step 4: Session is valid — slide TTL asynchronously
    CompletableFuture.runAsync(() -> {
        redis.expire("session:" + token, 900);          // Reset 15-min idle TTL
        redis.hset("session:" + token, "last_active",
                   String.valueOf(System.currentTimeMillis() / 1000));
    });

    // Step 5: Return session data
    return AuthResult.valid(session.get("user_id"),
                            parseRoles(session.get("roles")));
}
// Total time: ~1ms (single Redis HGETALL from primary; async TTL slide)
```

### Forced Logout (All Devices) — Atomic
```lua
-- Redis Lua script: atomic forced logout
-- KEYS[1]: user_sessions:{user_id}
-- Returns: number of sessions terminated

local user_sessions_key = KEYS[1]
local tokens = redis.call('SMEMBERS', user_sessions_key)
local count = 0

for _, token in ipairs(tokens) do
    redis.call('DEL', 'session:' .. token)
    count = count + 1
end

redis.call('DEL', user_sessions_key)
return count

-- All DELs in single atomic script: 
--   no window where some sessions are deleted and others are not
--   if script fails halfway, retry is safe (DEL is idempotent)
```

### Session Count Enforcement (Max 5 Per User)
```java
public void enforceSessionLimit(String userId, int maxSessions) {
    // SCARD: O(1) — just get count
    long currentCount = redis.scard("user_sessions:" + userId);

    if (currentCount >= maxSessions) {
        // Get all sessions, find oldest
        Set<String> tokens = redis.smembers("user_sessions:" + userId);
        String oldestToken = tokens.stream()
            .map(t -> Map.entry(t, redis.hget("session:" + t, "created_at")))
            .filter(e -> e.getValue() != null)
            .min(Comparator.comparing(e -> Long.parseLong(e.getValue())))
            .map(Map.Entry::getKey)
            .orElse(null);

        if (oldestToken != null) {
            redis.del("session:" + oldestToken);
            redis.srem("user_sessions:" + userId, oldestToken);
            log.info("Session limit enforced: evicted oldest session for user {}", userId);
        }
    }
}
// Called before creating new session on login
```

---

## Scaling Strategy
```
Phase 1 (0 → 10K validations/sec):
  PostgreSQL sessions table (simple; works at low scale)
  Schema: session_token, user_id, roles, created_at, expires_at
  SELECT on primary key (session_token) → fast but 10–50ms per query

Phase 2 (10K → 100K validations/sec):
  Redis standalone (single node)
  Drop PostgreSQL for session validation; keep as audit log
  Redis handles 100K GET ops/sec easily

Phase 3 (100K → 500K validations/sec):
  Redis Sentinel (HA: 1 primary + 2 replicas + 3 sentinels)
  Read from PRIMARY only (strong consistency)
  Failover < 5s on primary failure

Phase 4 (> 500K validations/sec or > 1 GB session data):
  Redis Cluster (3–5 shards)
  Partition by user_id hash (keep user's sessions on same shard)
  Forced logout: user_sessions:{user_id} always co-located with sessions (same shard)

Phase 5 (multi-region):
  Regional session stores; sessions pinned to region of login
  Cross-region session lookup via global registry (adds ~50ms; use for mobile roaming)
  OR: accept re-login when user crosses regions (simpler; often acceptable)
```

---

## Reliability Strategy
- **Redis Sentinel HA**: primary failure → replica promoted in < 5s; auth resumes
- **Fail-closed on Redis unavailable**: return 401; do not allow access; page on-call immediately
- **Session durability**: sessions are ephemeral; loss means re-login (not data loss); no RDB/AOF persistence needed
- **Graceful shutdown**: flush in-flight requests; drain connections; stop accepting new auth requests
- **Replica reads**: explicitly disabled; all reads from primary (strong consistency requirement)

---

## Security Considerations
- **Token entropy**: 256 bits (CSPRNG); brute force is computationally infeasible
- **Cookie flags**: `HttpOnly` (no JS access), `Secure` (HTTPS only), `SameSite=Strict` (CSRF protection)
- **Session fixation**: regenerate session token on privilege escalation (login, sudo, MFA step-up)
- **Session flooding**: max 5 sessions per user; enforce on every login
- **IP binding (optional)**: store IP at login; warn (but don't hard-block) if IP changes (mobile users roam across networks)
- **Audit log**: every session creation, destruction, and forced logout → immutable audit log (Case 10)
- **HTTPS only**: session tokens over HTTP are interceptable; enforce HTTPS everywhere
- **Token rotation**: regenerate token after every sensitive action (payment authorisation, password change)

---

## Observability
```
Metrics (critical — session store is security infrastructure):
  session_validation_latency_p99     (target: < 2ms; alert: > 5ms)
  session_validation_error_rate      (target: 0%; alert: > 0.1%)
  redis_primary_available            (binary gauge: 0 = down; alert immediately)
  active_sessions_total              (gauge; business metric)
  session_creation_rate              (logins/sec; spike = potential brute force)
  session_forced_logout_rate         (security events)
  multi_device_session_eviction_rate (session limit enforcement activity)
  redis_replication_lag_bytes        (alert if > 0; replica must be current)

Security metrics:
  failed_auth_rate_by_ip             (alert on IP with > 10 failures/min)
  session_creation_from_new_country  (alert for high-privilege users)
  concurrent_sessions_per_user       (alert if user has > 5; suggests account sharing)

Alerts:
  redis_primary_available = 0                     → IMMEDIATE PAGE (auth is down)
  session_validation_latency_p99 > 5ms            → Redis issue; approaching fail-closed
  session_creation_rate > 10× baseline in 5 min   → brute force or credential stuffing attack
  failed_auth_rate > 100/min from single IP       → block IP at WAF level
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Redis primary down | All session validations → 401 (fail-closed) | Sentinel failover < 5s; immediate page on-call |
| Sentinel failover in progress | 401s for ~5s during failover | Acceptable; PCI-DSS requires fail-secure default |
| Replica replication lag | Not an issue — we never read from replicas | Strong consistency enforced at client level |
| Session token collision | Two users share a token | 256-bit entropy → probability is 10^-77; negligible |
| Session fixation attack | Attacker pre-sets a known token | Generate server-side only; never accept client-suggested token |
| Replay after logout | Stolen token used after DEL | DEL is immediate; next GET returns MISS → 401 |
| Redis memory full | Cannot create new sessions | Alert at 80% memory; sessions are small (500 bytes each) |
| Sentinel quorum failure (2 of 3 sentinels down) | No automatic failover | Three sentinels across 3 AZs; 2 AZ failures required to lose quorum |

---

## Chaos Testing
```
Experiment 1: Kill Redis Primary
  Inject:    Forcefully terminate Redis primary (SIGKILL)
  Expected:  Sentinel promotes replica within 5s
  Expected:  401 errors for all authenticated requests during ~5s failover window
  Expected:  After failover: auth resumes normally
  Expected:  fail_closed_activations metric increments during window
  Red flag:  Any authenticated request succeeds during Redis unavailability (fail-open)
  Red flag:  Failover takes > 30s (Sentinel misconfigured or quorum broken)

Experiment 2: Forced Logout During Active Session
  Inject:    Trigger forced logout for user while they are actively making requests
  Expected:  In-flight requests complete (session token still valid for duration of request)
  Expected:  Immediately subsequent request: GET session:{token} → MISS → 401
  Expected:  All devices for that user receive 401 on next request
  Red flag:  Subsequent requests still succeed after forced logout
  Red flag:  Only one device is logged out (Lua script partial execution)

Experiment 3: Session Absolute Expiry (PCI-DSS)
  Inject:    Create session; set absolute_expires_at to 10 seconds from now
  Expected:  Session valid for 10 seconds; subsequent requests succeed
  Expected:  After 10 seconds: middleware detects absolute_expires_at passed → 401
  Expected:  Idle TTL (15 min) does NOT override absolute expiry
  Red flag:  Session still valid after absolute_expires_at (PCI-DSS violation)

Experiment 4: Session Flooding (Max 5 Sessions)
  Inject:    Log in from 10 different devices for same user
  Expected:  After 5 active sessions: 6th login evicts oldest session
  Expected:  At no point does user have > 5 concurrent sessions in Redis
  Expected:  Evicted session returns 401 on next use
  Red flag:  User accumulates > 5 sessions (session flooding attack possible)

Experiment 5: Redis AUTH Failure (wrong password)
  Inject:    Change Redis AUTH password; do not update application secret
  Expected:  All session validations fail immediately (Redis rejects AUTH)
  Expected:  All users receive 401 (fail-closed)
  Expected:  Alert fires within 30s
  Expected:  Fix: update application secret → auth resumes
  Red flag:  Application uses unauthenticated fallback path
```

---

## Monthly Cost Estimate
```
Scale assumption: 1M active sessions, 100K validations/sec

Redis Sentinel:
  1 primary × cache.r6g.large (32 GB, $130/mo)   = $130/mo
  2 replicas × cache.r6g.large ($130/mo)          = $260/mo
  3 sentinel nodes × cache.t3.micro ($15/mo)      = $45/mo
  Total Redis:                                     = $435/mo

PostgreSQL (session audit log only):
  db.t3.medium                                     = $70/mo

Monitoring + alerting:
  Session store is security-critical; enhanced monitoring
                                                   = $100/mo
────────────────────────────────────────────────────────────
Total: ~$605/month

Context:
  500 MB for 1M sessions → fits in smallest Redis instance with room to spare
  The $435/mo Redis cost is ~50× less than the business value of the auth it provides

At 10M concurrent sessions:
  5 GB session data → single cache.r6g.xlarge (32 GB) is sufficient → $260/mo
  (Redis scales via memory, not instances, for this use case)

At 100M concurrent sessions (very large platform):
  50 GB session data → 2 shards × cache.r6g.2xlarge (64 GB) → ~$1,000/mo
  Still extremely cheap for the value provided

Optimisation: sessions are already tiny (500 bytes); compression not worth complexity
```

---

## Migration Story

### Current State
Sessions stored in PostgreSQL (`sessions` table). At 10K RPS: DB CPU at 80% from session lookups. Redirect latency 30ms p99 (mostly from session SELECT).

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Dual-write sessions to PostgreSQL + Redis (zero downtime)
  Action:
    On session creation: write to BOTH PostgreSQL AND Redis
    On session validation: read from Redis first; fall back to PostgreSQL on miss
    On session deletion: delete from BOTH
  Monitor:
    Redis hit rate climbs as active sessions migrate (sessions have TTL; old ones expire)
    After 1 TTL cycle (15 min): nearly all active sessions in Redis
    PostgreSQL SELECT rate drops as Redis hit rate climbs
  Rollback:
    Disable Redis read; all validation falls back to PostgreSQL (back to baseline)

Step 2: Make Redis authoritative (zero downtime)
  Action:
    Stop writing sessions to PostgreSQL (new sessions: Redis only)
    Old sessions (in PostgreSQL but not Redis): expire naturally; users re-login
    PostgreSQL sessions table: keep as historical audit log (don't delete yet)
  Monitor:
    Redis hit rate = ~100% for new sessions
    No unexpected 401 spikes (if any: old sessions in PostgreSQL only → re-login required)
  Rollback:
    Resume dual-write; Redis becomes backup again

Step 3: Add absolute expiry enforcement (zero downtime)
  Action:
    Add absolute_expires_at field to all new session creations (created_at + 8h)
    Add middleware check: if now > absolute_expires_at → 401 + DEL
    Backfill: existing Redis sessions without absolute_expires_at → add field (HSET)
    Test: create session; wait 8h (or set to 10s for test); verify 401 after absolute expiry
  Monitor:
    No unexpected session terminations (verify absolute_expires_at calculation is correct)
    Compliance: PCI-DSS QSA can now verify absolute timeout policy
  Rollback:
    Remove absolute expiry check from middleware (TTL-only expiry remains)

Step 4: Upgrade to Redis Sentinel (5-min maintenance window)
  Action:
    Deploy Sentinel cluster (1 primary + 2 replicas + 3 sentinels)
    Update connection string to Sentinel endpoint
    Sentinel client handles primary failover transparently
    Test failover: kill primary → verify < 5s auto-promotion → auth resumes
  Monitor:
    Sentinel failover events (should be 0 in normal operation)
    Replication lag (should be < 10ms)
  Rollback:
    Update connection string back to standalone Redis (still running; sessions intact)

Step 5: Drop PostgreSQL sessions table (planned cleanup)
  Action:
    Archive old session rows to cold storage (audit compliance)
    DROP TABLE sessions (or keep as audit_sessions view with no new inserts)
    Remove dual-write code path
  Monitor:
    No auth regressions (PostgreSQL fallback no longer exists; Redis is sole store)
  Rollback:
    Restore PostgreSQL sessions table from backup; re-enable dual-write
    (This step is irreversible once PostgreSQL table is dropped; plan carefully)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Redis is authoritative for active sessions.
  → PostgreSQL holds session audit log (immutable; append-only).
  → These are different concerns: Redis = live auth state; PostgreSQL = compliance evidence.

□ Consistency model?
  → Strong: all reads from PRIMARY only; no replica reads.
  → Replica reads are explicitly disabled: replication lag could allow revoked session to pass.
  → This is different from Case 06 (distributed cache) where replica reads are acceptable.

□ Failure mode?
  → Redis down → fail-CLOSED (401 to all authenticated requests).
  → Sentinel failover: ~5s of 401s → acceptable; PCI-DSS requires fail-secure.
  → No graceful degradation possible: you cannot partially validate a session.

□ Hot partitions?
  → Single user with many requests → same user_id → same Redis key.
  → Not a concern: Redis can handle 100K GET ops/sec per node; session keys are different.
  → Only hot partition if one user is making 100K+ req/sec (API client abuse → rate limiting).

□ Cost driver?
  → Redis Sentinel (~72% of session store cost).
  → Sessions are tiny (500 bytes); cost is driven by HA topology, not data volume.

□ Security risk?
  → Session fixation: client suggests token. Mitigation: always generate server-side.
  → Token enumeration: 256-bit entropy eliminates this.
  → Replay after logout: instant DEL eliminates this (next GET = MISS = 401).
  → Absolute expiry bypass: enforced in middleware on every request; not just Redis TTL.

□ Operational burden?
  → Sentinel: monitor failover events; test failover quarterly.
  → Absolute expiry: middleware must be correct; test extensively.
  → Session limit enforcement: monitor eviction rate; alert if users regularly hit limit.
  → Token rotation: implement for sensitive actions (payment auth, password change).

□ Migration path?
  → PostgreSQL → dual-write → Redis authoritative → absolute expiry → Sentinel → drop PostgreSQL.
  → Each step independently rollbackable until the final DROP TABLE.
```

---

## Staff Engineer Discussion Points

**"How does this map to your ForgeRock and IAM platform work?"**
ForgeRock's CTS (Core Token Service) is exactly this architecture. CTS uses a specialised distributed store (similar to Redis Cluster) to store session tokens, OAuth tokens, and SAML assertions. The session token in ForgeRock is an opaque string stored in CTS. When a request arrives at AM (Access Manager), it performs a CTS GET — equivalent to our Redis HGETALL. ForgeRock's "session stateless JWT" feature is the equivalent of switching from opaque tokens to JWT for sessions — gaining statelessness at the cost of instant revocation. Understanding this case study gives you the mental model for how CTS works under the hood.

**"What's the difference between a session token and an access token in an OAuth/OIDC flow?"**
Session token (opaque): represents the user's authentication state with your platform. Long-lived (hours). Revocable instantly via Redis DEL. Used by the browser/mobile app to maintain "logged in" state. Stored in an HttpOnly cookie. Access token (JWT): short-lived (15 min). Carries the user's identity and permissions for a specific API call. Stateless — verified by signature, no network hop. Presented to backend services as Bearer token. When it expires, the client uses the session token to request a new access token from the auth server. This separation means: revoke the session token → user cannot get new access tokens → effectively logged out (within 15 min, existing access tokens also expire).

**"How do you handle a user whose phone is stolen?"**
Fraud team or user triggers forced logout via DELETE /internal/v1/sessions?user_id=u-12345. All Redis session tokens for that user are DEL'd atomically (Lua script). Next request from any device for that user → GET → MISS → 401. Additionally: invalidate any outstanding access tokens — for stateless JWTs, this requires an access token blocklist (Redis Set of revoked JTI claims) with TTL matching the JWT expiry. For 15-min JWT expiry, the blocklist only needs to hold entries for 15 minutes — manageable size.

**"How does this design satisfy PCI-DSS session requirements?"**
PCI-DSS 8.2.8: "If a session has been idle for more than 15 minutes, require the user to re-authenticate." → Implemented: Redis TTL of 900 seconds (15 min); sliding on each authenticated request. PCI-DSS 12.3.3: "Sessions for interactive sessions do not exceed 8 hours." → Implemented: absolute_expires_at = created_at + 8h; enforced in middleware on every request. PCI-DSS 8.8: "All individual user authentication factors are revoked upon personnel departure." → Implemented: forced logout (DELETE all sessions for user_id). PCI-DSS requirement for fail-secure → Implemented: fail-closed on Redis unavailability.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Validate user identity on every request fast"  → Session Store (Redis; opaque token)
  "Revoke access immediately on logout/fraud"     → Opaque token + Redis DEL (not JWT)
  "PCI-DSS absolute session timeout"              → absolute_expires_at in session data
  "Multi-device: force logout all devices"        → user_sessions:{uid} Redis Set + Lua DEL

OPAQUE TOKEN vs JWT DECISION:
  Need instant revocation?         → Opaque token (Redis DEL; immediate)
  Tolerate up to 15 min stale?     → JWT access token (no network hop; stateless)
  Best practice for payment apps:  → Both: session token (opaque) + access token (JWT, 15 min)

FAIL-CLOSED vs FAIL-OPEN:
  Rate limiter (Case 03):  FAIL-OPEN  → Redis down → allow requests (availability > accuracy)
  Session store (Case 07): FAIL-CLOSED → Redis down → 401 (security > availability)
  Rule: "What does failure unlock?" If the answer is data/money/PII → fail-closed.

PATTERNS REJECTED (and why):
  JWT for sessions       → cannot revoke before expiry; blocklist defeats JWT's purpose
  PostgreSQL for sessions → 10–50ms per query; cannot handle 100K validations/sec
  Replica reads          → replication lag allows revoked session to pass; auth bypass risk
  Fail-open on Redis down → stale session = potential fraud access; never acceptable

SCC LENS:
  STATE:
    Active session state: Redis primary (authoritative; ephemeral; loss = re-login)
    Session audit log:    PostgreSQL (durable; append-only; compliance evidence)
    No other state: session data is fully in Redis; DB is not in validation hot path

  COORDINATION:
    Forced logout: Lua script atomically DELs all tokens for user (no partial state)
    Absolute expiry: middleware checks absolute_expires_at on every request (not just TTL)
    Session creation: enforce max-5 limit before creating (SCARD + evict oldest if needed)
    Token rotation: regenerate on privilege escalation (session fixation prevention)

  CONCENTRATION:
    Hot path: Redis GET on every authenticated request → Redis IS the bottleneck
    Size it right: 100K validations/sec → Redis node handles this; sentinel for HA
    Single user flood: same user_id → same session keys → Redis handles; rate limit user

IAM PLATFORM CONNECTION:
  ForgeRock CTS = this architecture (distributed session store)
  ForgeRock AM session validation = middleware auth check (GET session:{token})
  ForgeRock "global logout" = forced logout (Lua DEL all user sessions)
  ForgeRock "session stateless" = switching opaque to JWT (tradeoff: no instant revocation)
  ForgeRock session quota = max sessions per user (SCARD enforcement)

INTERVIEW ANSWER TRIGGER:
  "Design a session store" →
    1. Opaque token (not JWT): revocation requirement drives this
    2. Redis primary-only reads: fail-closed; no replica lag tolerated
    3. Sliding + absolute expiry: UX + PCI-DSS compliance
    4. Fail-CLOSED on Redis down (opposite of rate limiter — explain why)
    5. SCC: State=Redis (authoritative), Coordination=Lua forced logout,
            Concentration=Redis is hot path → size correctly + Sentinel HA
    6. IAM tie-in: ForgeRock CTS is this architecture in production
```

---

*Next: Case Study 08 — Feature Flag Service*
