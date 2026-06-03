# Case Study 46 — Session Management and Token Revocation

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Session store design, Revocation vs JWT TTL, Sliding vs absolute expiry.

---

## Business Context

A payment app user should stay logged in while active but be automatically logged out after inactivity or when they explicitly sign out. An employee whose laptop is stolen should have their session revoked immediately. A merchant API key that was leaked should be invalidated within seconds. Session management is the gap between authentication (proving who you are) and access decisions (are you still allowed in right now?).

The fundamental tension: JWTs are stateless (no revocation), sessions are stateful (revocable but require storage). The solution for a payment platform is a hybrid: short-lived JWTs for API calls + a thin revocation layer for emergency termination.

---

## Recognition Framework

### Sessions vs Tokens

```
Server-Side Sessions:
  Server stores: session_id → { user_id, role, tenant, created_at, last_active, ip, device }
  Client stores: session_id cookie (HttpOnly; Secure; SameSite)
  Revocation: DELETE session_id from store → immediate effect
  Scaling: shared session store (Redis) across all server instances
  
  Pros: immediate revocation; full control; see all active sessions
  Cons: storage overhead; Redis is a dependency; not naturally stateless
  
  Use for: user-facing web/mobile apps; anything needing immediate logout

JWT Access Tokens (Stateless):
  Server stores: nothing (JWT is self-contained)
  Client stores: JWT in memory or localStorage
  Revocation: cannot revoke without a blocklist (token valid until TTL expires)
  Scaling: stateless; any instance validates without DB/Redis call
  
  Pros: no storage; horizontally scalable; no shared state
  Cons: cannot revoke (only expire); token theft valid until TTL; must use short TTL
  
  Use for: API-to-API calls; short-lived operations; when statelessness is critical

CHOSEN: Hybrid approach
  User-facing app: server-side session (revocable) + short-lived JWT (15 min)
  How it works:
    Login → create session in Redis → issue JWT with session_id embedded
    JWT validates locally (fast); session check on revocation-sensitive operations only
    Logout → delete session → JWT is still valid for 15 min but session is dead
    New requests after logout: JWT validates locally but session check fails → reject
```

---

## Problem Statement

Session management for a payment platform. Redis-backed session store. Revocation for stolen credentials. Device management (see all active sessions). Sliding vs absolute expiry. API key revocation.

---

## Session Store Design

```sql
-- Session metadata (PostgreSQL for audit; Redis for hot path)
CREATE TABLE sessions (
    session_id       VARCHAR(64)  PRIMARY KEY,  -- cryptographically random 32 bytes (base64url)
    user_id          VARCHAR(64)  NOT NULL,
    tenant_id        VARCHAR(64),
    device_id        VARCHAR(64),               -- fingerprint of device
    device_name      VARCHAR(128),              -- "Chrome on MacBook" (user-visible)
    ip_address       INET,
    user_agent       TEXT,
    auth_method      VARCHAR(32),              -- PASSWORD / SAML / OIDC / PASSKEY
    mfa_verified     BOOLEAN DEFAULT FALSE,
    mfa_verified_at  TIMESTAMP,
    created_at       TIMESTAMP DEFAULT NOW(),
    last_active_at   TIMESTAMP DEFAULT NOW(),
    absolute_expiry  TIMESTAMP,               -- hard expiry regardless of activity
    status           VARCHAR(16) DEFAULT 'ACTIVE'  -- ACTIVE / REVOKED / EXPIRED
);
```

```
Redis session store (hot path; TTL-managed):
  Key:   session:{session_id}
  Value: { user_id, tenant_id, device_id, role, mfa_verified, created_at, last_active }
  TTL:   sliding expiry (e.g., 30 min; refreshed on every active request)
  
  On session validation:
    GET session:{session_id}
    If null → session expired (TTL elapsed) or never existed → 401
    If status=REVOKED → 401 (check DB for revocation status; or embed in Redis value)
    Update last_active_at + EXPIRE (sliding window)

Sliding vs Absolute expiry:
  Sliding: TTL resets on every request → active users never logged out
    User browsing dashboard: 30-min idle timeout
    User types every 5 min: stays logged in indefinitely
    Risk: session that lives forever if user is very active (not always desirable)
  
  Absolute: hard expiry at session creation + N hours, regardless of activity
    e.g., 24-hour absolute expiry for consumer app (re-login once per day)
    8-hour absolute for payment dashboard (match work day)
  
  CHOSEN: Sliding (30 min idle) + Absolute (24h hard limit)
    User stays in as long as active (good UX)
    But must re-authenticate every 24h (security requirement)
    Sensitive operations (large transfers): also require MFA re-verification
```

---

## Session Operations

### Login (Session Creation)
```java
@PostMapping("/api/v1/auth/login")
public AuthResponse login(@RequestBody LoginRequest req, HttpServletRequest httpReq) {
    // 1. Verify credentials
    User user = userService.verify(req.getEmail(), req.getPassword());
    
    // 2. Check if MFA is required
    boolean mfaRequired = mfaPolicy.isMfaRequired(user, extractDeviceInfo(httpReq));
    
    // 3. Create session (even before MFA; mark mfa_verified=false)
    String sessionId = generateSecureToken(32);  // 32 random bytes → base64url
    Session session = Session.builder()
        .sessionId(sessionId)
        .userId(user.getId())
        .tenantId(user.getTenantId())
        .deviceId(extractDeviceFingerprint(httpReq))
        .deviceName(parseUserAgent(httpReq.getHeader("User-Agent")))
        .ipAddress(extractClientIp(httpReq))
        .mfaVerified(!mfaRequired)  // verified if MFA not required
        .createdAt(Instant.now())
        .absoluteExpiry(Instant.now().plus(24, HOURS))
        .build();
    
    // 4. Store in Redis (sliding 30-min TTL) + PostgreSQL (audit)
    redis.set("session:" + sessionId, session, Duration.ofMinutes(30));
    sessionRepo.save(session);
    
    // 5. Issue JWT with session_id embedded
    String jwt = jwtService.issue(user, sessionId, Duration.ofMinutes(15));
    
    // 6. Set HttpOnly cookie for web; return JWT for API/mobile
    return AuthResponse.of(jwt, sessionId, mfaRequired);
}
```

### Revocation (Logout / Force Logout)
```java
// Immediate logout (user-initiated)
@PostMapping("/api/v1/auth/logout")
public void logout(@RequestHeader("Authorization") String bearer) {
    String sessionId = jwtService.extractSessionId(bearer);
    
    // Delete from Redis: all future requests with this session fail immediately
    redis.delete("session:" + sessionId);
    
    // Update DB for audit trail
    sessionRepo.updateStatus(sessionId, SessionStatus.REVOKED, Instant.now());
}

// Force logout (admin revokes specific session — e.g., stolen device)
@DeleteMapping("/internal/v1/sessions/{sessionId}")
public void forceRevoke(@PathVariable String sessionId, @RequestHeader("X-Admin-ID") String adminId) {
    redis.delete("session:" + sessionId);
    sessionRepo.updateStatus(sessionId, SessionStatus.REVOKED, Instant.now());
    auditLog.record("SESSION_FORCED_REVOKE", adminId, sessionId);
}

// Revoke ALL sessions for a user (termination / suspected compromise)
@DeleteMapping("/internal/v1/users/{userId}/sessions")
public void revokeAllSessions(@PathVariable String userId) {
    List<String> sessions = sessionRepo.findActiveByUserId(userId);
    for (String sessionId : sessions) {
        redis.delete("session:" + sessionId);
    }
    sessionRepo.revokeAllForUser(userId);
    // All JWT tokens for this user: next validation fails (session not found in Redis)
}
```

---

## JWT + Session Hybrid Validation

```java
@Component
public class AuthenticationFilter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        String jwt = extractBearerToken((HttpServletRequest) req);
        if (jwt == null) { reject401(); return; }
        
        // Step 1: Validate JWT locally (fast; no Redis call for most requests)
        JwtClaims claims = jwtService.validate(jwt);
        if (claims == null) { reject401(); return; }  // invalid signature or expired
        
        // Step 2: Session check (Redis; only for sensitive operations OR revocation check)
        String sessionId = claims.getSessionId();
        boolean requireSessionCheck = isHighRiskOperation(req) || isAdminOperation(req);
        
        if (requireSessionCheck || isSessionValidationEnabled()) {
            Session session = redis.get("session:" + sessionId);
            if (session == null) { reject401(); return; }  // session revoked or expired
            if (session.getAbsoluteExpiry().isBefore(Instant.now())) { reject401(); return; }
            
            // Refresh sliding TTL
            redis.expire("session:" + sessionId, Duration.ofMinutes(30));
        }
        
        // Set request context
        SecurityContext.set(claims);
        chain.doFilter(req, resp);
    }
    
    // For most requests: JWT validation only (< 1ms)
    // For sensitive operations: JWT + session check (< 2ms; Redis round trip)
    // On revocation: session deleted from Redis → next request fails (max 15 min lag for JWT-only paths)
}
```

---

## Device Management (User-Visible Sessions)

```
User sees in Settings > Security:
  "Active Sessions"
  ┌─────────────────────────────────────────────────────────────────┐
  │ Chrome on MacBook (Mumbai, IN) — Active now            [Revoke] │
  │ iPhone 15 (Mumbai, IN) — 2 hours ago                   [Revoke] │
  │ Unknown device (London, UK) — 3 days ago               [Revoke] │  ← suspicious
  └─────────────────────────────────────────────────────────────────┘
  [Revoke All Other Sessions]

Implementation:
  GET /api/v1/users/me/sessions
  Returns: SELECT session_id, device_name, ip_address, last_active_at
           FROM sessions WHERE user_id = ? AND status = 'ACTIVE' ORDER BY last_active_at DESC
  
  User clicks "Revoke" on London session:
    DELETE /api/v1/users/me/sessions/{session_id}
    → redis.delete("session:" + sessionId)
    → sessionRepo.revoke(sessionId)
  
  Security signal: user sees unknown session → trigger security review + force password reset

Session anomaly detection:
  Same user_id, two sessions: different countries within 2 hours → impossible travel
  Action: email alert to user; require re-authentication on suspicious session
  → "New sign-in from London detected. If this wasn't you, revoke it immediately."
```

---

## API Key Revocation

```
Merchant API keys are long-lived tokens (not sessions)
  Format: sk_live_{32_random_chars}  (32 chars = 192 bits entropy)
  Storage: store HASH (bcrypt or SHA-256); never plaintext
  
  API key validation per request:
    prefix = api_key.split("_")[2][:8]  # first 8 chars (lookup key)
    stored_hash = apiKeyStore.findByPrefix(prefix)
    verified = bcrypt.verify(api_key, stored_hash)
    
  Why prefix + hash?
    Prefix: enables O(1) lookup (don't iterate all keys to find match)
    Hash: never store plaintext (database breach → all keys safe)
  
  Revocation: mark as REVOKED in DB + Redis cache
    redis.set("apikey:revoked:" + keyId, "1", Duration.ofDays(90))
    On validation: check revoked cache before bcrypt verify (cheap fast path)

Secret key rotation:
  Merchant requests: "rotate my API key"
  System: issue sk_live_NEW_{chars}
  Old key: 24-hour grace period (both old and new work; gives merchant time to update)
  After 24h: old key revoked
  
  This prevents: merchant updating their code → brief window of no valid key
  Never: immediate revocation of old key on rotation (breaks live production)

Key scopes (beyond revocation):
  sk_live_{chars}: full access
  rk_live_{chars}: restricted key (read-only; or limited to specific endpoints)
  sk_test_{chars}: test mode key (different environment; harmless)
  
  Scope enforcement: gateway checks key scope against requested operation
```

---

## Observability
```
Session metrics:
  active_sessions_count by {auth_method, device_type}
  session_creation_rate
  session_revocation_rate by {reason}         (logout, admin-forced, expired, stolen)
  session_duration_distribution               (how long users stay logged in)
  concurrent_sessions_per_user_avg            (anomaly: > 5 concurrent sessions)
  impossible_travel_detections_total          (security signal)
  
API key metrics:
  api_key_validation_latency_p99              (target < 5ms; bcrypt verify)
  api_key_revocation_effective_latency        (time from revoke to rejection)
  api_key_usage_by_key_id                    (detect leaked keys via unusual usage spike)
```

---

## Architecture Review Checklist
```
□ Revocation is immediate (within 1 request):
  Server-side session: delete from Redis → immediate rejection
  JWT alone: cannot revoke before TTL (max 15 min lag)
  Hybrid: JWT + session check → effective immediate revocation for sensitive ops

□ Session security:
  session_id: 32 random bytes (256-bit entropy; not guessable)
  HttpOnly + Secure + SameSite cookies (browser sessions)
  session_id NOT in URL (logs, referrer headers would leak it)
  
□ Absolute expiry is mandatory:
  Sliding alone: session lives forever for active users
  Absolute: even active users re-authenticate every 24h (security hygiene)

□ Device visibility:
  User can see ALL active sessions
  User can revoke individual sessions (stolen device scenario)
  Suspicious session alert (new country → email notification)

□ API key security:
  Never store plaintext keys (bcrypt hash; prefix for lookup)
  Revocation: Redis cache check (fast) before bcrypt verify
  Rotation: grace period (24h both old and new work)

□ SCC LENS:
  STATE: Redis (hot session store; TTL-managed), PostgreSQL (session audit log)
  COORDINATION: session TTL refresh (sliding window), revocation propagation (Redis pub/sub to clear L1 caches)
  CONCENTRATION: Redis is the single session validation point → cluster; fail-open NOT acceptable (security system)
```

---

*Next: Case Study 47 — Compliance Architecture (PCI-DSS, GDPR, RBI)*
