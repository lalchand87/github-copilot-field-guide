# Case Study 05 — API Gateway

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Recognition Framework, Circuit Breaker, Migration Story.

---

## Business Context

An API gateway creates a single, consistent entry point for all client traffic — enforcing auth, rate limiting, and routing in one place rather than duplicating that logic across dozens of microservices. Every feature added to the gateway is a potential failure mode. Every millisecond of gateway overhead affects every service.

The design principle: **keep the gateway thin**. A gateway outage is a total service outage. For a payment platform, this means every payment transaction fails. The gateway must be the most reliable component in the system.

**Business goals driving architecture:**
- Centralise cross-cutting concerns (auth, rate limiting, routing) — one place, not 50
- Gateway must never be the bottleneck (< 10ms added overhead p99)
- Config changes (new routes, limit changes) without restarts or deployments
- Backends reachable only through gateway — enforce via mTLS
- Gateway failure must not cause total outage — horizontal scaling + health checks

---

## Recognition Framework

### Signals from the Problem
```
- Single entry point for all traffic → routing is the core function
- Auth at edge → avoid per-service duplication (consistency + security)
- Rate limiting per client → shared state (Redis) across stateless instances
- Circuit breaking → prevent cascade failures when a backend is slow/down
- Zero-downtime config changes → hot-reload without restarts
- Gateway is SPOF → must be horizontally scaled with health checks
- < 10ms overhead budget → eliminates any solution with a network hop per request
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Routing (Trie) | Path/method-based dispatch; O(path_length) lookup; hot-swappable |
| Auth in-process (JWKS cache) | 10ms budget eliminated network hop to auth service; JWT validation is pure CPU |
| Rate Limiting (Redis Lua) | Shared state across stateless instances (from Case 03) |
| Circuit Breaker (in-process) | Prevent cascade failures; independent per-instance; no Redis dependency |
| Service Discovery | Upstream addresses change dynamically as services scale |
| Config Hot-Reload (etcd watch) | Push-based notification; zero-downtime route updates |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Centralised auth service (RPC per request) | Adds 5–20ms network hop; violates 10ms overhead budget |
| Shared circuit breaker state (Redis) | Adds Redis dependency to circuit breaker path; in-process is sufficient and safer |
| Config polling (instead of etcd watch) | Polling adds lag; etcd watch is push-based and near-instant |
| Single gateway instance | SPOF; horizontally scale behind L4 LB |
| Gateway as business logic layer | "API gateway does X for this endpoint" is a slippery slope; keep it thin |
| Sidecar proxy per request hop | Adds latency at every hop; gateway pattern is centralised |

### Why In-Process JWT Validation and Not a Dedicated Auth Service?
```
Option A: Dedicated auth service (RPC per request)
  Client → Gateway → Auth Service (validate JWT) → Backend
  Latency: +15ms for auth service round-trip on every request
  Availability: auth service down = all requests fail
  Verdict: REJECTED — latency and SPOF risk

Option B: In-process JWT validation (chosen)
  Client → Gateway (validates JWT locally using cached JWKS key)
  Latency: ~1ms (pure CPU; RSA signature verification)
  JWKS cache: loaded from auth server on startup; refreshed every 5 minutes
  On JWKS refresh failure: use last-known keys (auth server outage doesn't break gateway)
  Verdict: CHOSEN — eliminates network hop; cache tolerates auth server downtime

Trade-off: JWKS cache is 5 minutes stale
  If a signing key is rotated, old tokens remain valid for up to 5 minutes
  Mitigation: Force JWKS refresh when a 401 is returned by a backend
  Acceptable: key rotation is a rare, planned event
```

### Why In-Process Circuit Breaker and Not Shared Redis State?
```
In-process (chosen):
  Each gateway instance tracks upstream health independently
  No Redis dependency on the circuit breaker path
  If Redis is unavailable, circuit breaker still works
  Tradeoff: each instance opens its own circuit (not globally coordinated)
  In practice: if a backend is failing, it fails for all instances quickly → all circuits open

Shared Redis state (rejected):
  Global circuit state visible to all instances
  But: adds Redis as a dependency of circuit breaking
  If Redis is slow → circuit breaker check adds latency → defeats the purpose
  And: the benefit (global coordination) is marginal — backends fail for all instances anyway
```

---

## Problem Statement

API gateway as the single entry point for a microservices backend (payment-service, user-service, notification-service, etc.). Handles routing, authentication, rate limiting, circuit breaking, TLS termination, and config hot-reload.

---

## Functional Requirements
- Route requests to correct backend service based on path and HTTP method
- Authenticate requests (validate JWT / API key)
- Rate limit per client (from Case 03)
- Circuit break failing or slow backends
- TLS termination at gateway edge
- Config hot-reload: route changes, new services, limit changes without restart
- Request/response transformation: add correlation headers, strip auth headers to backends
- mTLS to backends: only gateway can call backends

## Non-Functional Requirements
- Latency added by gateway: **< 10ms p99**
- Availability: **99.999%** (five nines — gateway down = total outage)
- Throughput: **100,000 RPS** through gateway
- Zero-downtime config updates

---

## Capacity Estimation
```
100,000 RPS through gateway.

Per-request overhead budget (must sum to < 10ms):
  TLS handshake:          ~0ms  (connection reuse; TLS session resumption)
  Route lookup (trie):    ~0.1ms (in-process; O(path_length))
  JWT validation:         ~1ms  (in-process; RSA verify; cached JWKS)
  Rate limit check:       ~2ms  (Redis Lua round-trip)
  Request transformation: ~0.1ms (add/strip headers; in-process)
  Circuit breaker check:  ~0.1ms (in-process HashMap lookup)
  Total gateway overhead: ~3–4ms (well within 10ms budget)
  Upstream call:          50–200ms (not gateway's problem)

Memory per gateway instance:
  Route trie:              10,000 routes × 200 bytes = ~2 MB
  JWKS cache:              ~10 KB (a few public keys)
  Circuit breaker state:   100 upstreams × 200 bytes = ~20 KB
  Connection pool:         1,000 connections × ~8 KB = ~8 MB
  Total per instance:      ~10 MB (trivially small)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Routing (Trie) | O(path_length) per lookup; atomic hot-swap without restart |
| Auth (JWKS in-process) | Zero network hop; ~1ms; tolerates auth server downtime |
| Rate Limiting (Redis Lua) | Shared state across stateless gateway instances |
| Circuit Breaker (in-process) | Prevents cascade; no Redis dependency in failure path |
| Config Hot-Reload (etcd watch) | Push-based; < 1s propagation; zero-downtime |
| mTLS to Backends | Only gateway can call backends; backends reject direct calls |

---

## Data Model

```yaml
# Route config (stored in etcd, watched by gateway)
# Loaded into in-process trie on startup; hot-swapped on change

routes:
  - id: payment-create
    path: /api/v1/payments
    methods: [POST]
    upstream: payment-service
    auth_required: true
    rate_limit_tier: standard
    timeout_ms: 5000
    retry_count: 2
    strip_headers: [Authorization]
    add_headers:
      X-Gateway-Version: "2.1"

  - id: payment-get
    path: /api/v1/payments/{id}
    methods: [GET]
    upstream: payment-service
    auth_required: true
    rate_limit_tier: standard
    timeout_ms: 3000

  - id: user-profile
    path: /api/v1/users/{id}
    methods: [GET, PUT]
    upstream: user-service
    auth_required: true
    strip_headers: [Authorization, X-Internal-Token]

  - id: health
    path: /health
    methods: [GET]
    upstream: null              # gateway responds directly (no upstream call)
    auth_required: false
    response_body: '{"status":"ok"}'

upstreams:
  payment-service:
    discovery: kubernetes       # or consul / static IPs
    namespace: payments
    service: payment-svc
    port: 8080
    health_check_path: /health
    health_check_interval_ms: 5000
    circuit_breaker:
      error_rate_threshold: 0.5   # open circuit if > 50% errors in window
      window_seconds: 10
      open_duration_seconds: 30
      half_open_probe_count: 1

  user-service:
    discovery: kubernetes
    namespace: users
    service: user-svc
    port: 8080
```

```sql
-- Client registry (PostgreSQL, cached in-process for auth + rate limiting)
CREATE TABLE api_clients (
  api_key         VARCHAR(64)  PRIMARY KEY,
  client_id       BIGINT,
  client_name     VARCHAR(128),
  rate_limit_tier VARCHAR(32),
  is_active       BOOLEAN,
  allowed_scopes  TEXT[],       -- ['payments:read', 'payments:write', 'users:read']
  created_at      TIMESTAMP    DEFAULT NOW()
);
```

---

## API Design

```
All inbound traffic:
  Any HTTP method to any path → Gateway → Backend (if route matches)

Headers ADDED by gateway to upstream request:
  X-Client-ID:      {client_id from JWT or API key}
  X-Request-ID:     {UUID generated per request — trace correlation}
  X-Forwarded-For:  {original client IP}
  X-Gateway-Route:  {route_id — for backend logging}

Headers STRIPPED by gateway from upstream request:
  Authorization:    (backend gets X-Client-ID; never gets raw token)
  X-Internal-*:     (any internal headers from client are stripped)

Headers ADDED to response:
  X-Request-ID:     {same UUID — enables full trace correlation}
  X-RateLimit-*:    (from rate limiter — Case 03)

Gateway error responses (not from upstream):
  401 Unauthorized:     auth failed (invalid/expired JWT or API key)
  429 Too Many Requests: rate limit exceeded
  502 Bad Gateway:       upstream returned unexpected error
  503 Service Unavailable: circuit open (upstream in cooldown)
  504 Gateway Timeout:   upstream did not respond within timeout_ms
```

---

## High-Level Architecture

```
Internet
  │
  ▼
WAF / DDoS Protection (Cloudflare / AWS Shield)
  │  Blocks: SQL injection, XSS, oversized payloads, known attack patterns
  ▼
L4 Load Balancer (TCP; distributes connections to gateway instances)
  │  Health checks: TCP connect to :443 every 5s; remove unhealthy instances
  ▼
API Gateway Cluster (N stateless instances; horizontally scaled)
  │
  ├── Per-request processing pipeline:
  │
  │   1. TLS Termination
  │      Certificate from ACM/Let's Encrypt; TLS 1.2+ only; session resumption
  │
  │   2. Route Lookup (in-process trie; O(path_length))
  │      Match: path prefix + HTTP method
  │      If no match: 404
  │
  │   3. Authentication (in-process; no network hop)
  │      JWT: verify RS256 signature using cached JWKS public key
  │      API Key: lookup in in-process LRU cache → PostgreSQL on miss
  │      Extract: client_id, scopes, expiry
  │      If auth fails: 401
  │
  │   4. Authorisation (in-process; scope check)
  │      Verify client scopes include required scope for this route
  │      If scope missing: 403
  │
  │   5. Rate Limiting (Redis Lua; ~2ms)
  │      Check sliding window counter (Case 03)
  │      If limit exceeded: 429
  │
  │   6. Request Transformation
  │      Strip: Authorization header, X-Internal-* headers
  │      Add: X-Client-ID, X-Request-ID, X-Forwarded-For
  │
  │   7. Circuit Breaker Check (in-process; per upstream)
  │      If circuit OPEN: 503 immediately (no upstream call)
  │      If circuit CLOSED or HALF-OPEN: proceed
  │
  │   8. Forward to Upstream (HTTP/2 connection pool)
  │      Use service discovery to get upstream IP:port
  │      Connection pool: 100 connections per upstream per gateway instance
  │      Timeout: route.timeout_ms (e.g. 5000ms for payment-service)
  │      Retry: route.retry_count on 5xx (not on 4xx; not on POST)
  │
  │   9. Response Transformation
  │      Add: X-Request-ID header (for trace correlation)
  │      Strip: internal headers from upstream response
  │
  │  10. Structured logging (async; never in critical path)
  │      Log: method, path, client_id, upstream, status, latency, request_id
  │
  ├── Shared state:
  │   Redis:      rate limit counters + API key LRU cache
  │   etcd:       route config (watched; hot-reload on change)
  │   PostgreSQL: client registry (read on API key cache miss)
  │
  └── Per-instance state:
      JWKS cache:        public keys for JWT validation
      API key LRU cache: in-process; 10K entries; 30s TTL
      Circuit breakers:  one per upstream; in-process state machine
      Route trie:        in-memory; hot-swapped on etcd change

Backends (payment-service, user-service, etc.):
  - mTLS only: reject any call without gateway's client certificate
  - Cannot be called directly from internet (network policy + security group)
  - Trust X-Client-ID header (only gateway can set it after stripping Authorization)
```

---

## Detailed Components

### Route Lookup (Trie)
```
Data structure: prefix trie with method matching at leaf nodes

/api/v1/payments          → payment-service (POST, GET)
/api/v1/payments/{id}     → payment-service (GET, PUT, DELETE)
/api/v1/users/{id}        → user-service (GET, PUT)
/api/v1/notifications/**  → notification-service (any method; ** = wildcard)
/health                   → gateway (no upstream)

Lookup: O(path_length) → for /api/v1/payments/123: ~5 trie traversals
Hot-swap: on etcd config change:
  1. Parse new config
  2. Validate (no missing upstreams; valid path patterns)
  3. Build new trie in background thread
  4. Atomic pointer swap (old trie → new trie)
  5. Old trie GC'd after in-flight requests drain
  Zero dropped requests; zero downtime

Path parameters: {id} captures any single segment; ** captures all remaining segments
```

### JWT Validation (In-Process, No Network Call)
```
Flow:
  1. Extract Bearer token from Authorization header
  2. Base64-decode JWT header → get "kid" (key ID)
  3. Look up public key by kid in JWKS cache
     Cache hit:  proceed to step 4
     Cache miss: fetch JWKS from auth server (https://auth.internal/jwks)
                 populate cache; proceed
  4. Verify RS256 signature using public key
     Failure: return 401 immediately
  5. Validate claims:
     "exp" > current timestamp  (not expired)
     "iss" == expected issuer   (not a token from another system)
     "aud" includes this service (not a token for a different audience)
  6. Extract claims:
     "sub"    → client_id (injected as X-Client-ID to upstream)
     "scopes" → list of permissions (checked in step 4 of pipeline)

Total: ~1ms (CPU-bound; RSA verify is O(1))

JWKS refresh strategy:
  Refresh every 5 minutes (background goroutine/thread)
  On refresh failure: use existing cached keys (auth server outage doesn't break gateway)
  On new kid in token (unknown key): force immediate JWKS refresh (handles key rotation)
  On 401 from upstream: consider forcing JWKS refresh (backend saw invalid token)
```

### Circuit Breaker (In-Process State Machine)
```
States per upstream:
  CLOSED   → Normal operation; requests forwarded to upstream
  OPEN     → Upstream unhealthy; requests fail fast with 503 (no upstream call)
  HALF-OPEN → Testing recovery; allow 1 probe request; evaluate outcome

State transitions:
  CLOSED → OPEN:      error_rate > 50% in last 10s (errors include: 5xx, timeouts)
  OPEN → HALF-OPEN:   after open_duration_seconds (default: 30s)
  HALF-OPEN → CLOSED: probe request succeeds (2xx response within timeout)
  HALF-OPEN → OPEN:   probe request fails (5xx or timeout)

Error counting:
  Error = status code >= 500 OR response time > timeout_ms
  Not an error = 4xx (client errors don't indicate upstream unhealthy)
  Sliding window: last 10 seconds of requests

Benefits of in-process state:
  No Redis dependency → works even if Redis is unavailable
  No network hop for circuit check → ~0.1ms lookup
  Each gateway instance self-heals independently
  In practice: all instances open circuit for same upstream within ~10s of failure

Implementation:
  ConcurrentHashMap<String, CircuitBreakerState> breakers;
  // Key: upstream name ("payment-service")
  // Value: state + counters + last state change timestamp
```

### Config Hot-Reload (etcd Watch)
```
Startup:
  1. Connect to etcd cluster
  2. GET /config/gateway/routes → load full route config
  3. Build in-memory trie
  4. Start etcd watch on /config/gateway/routes

On etcd change event:
  1. Receive new config payload
  2. Parse and validate
     - All upstream names resolve in service discovery
     - No duplicate path+method combinations
     - Required fields present
  3. If validation fails: reject update; keep old trie; log error
  4. If valid: build new trie
  5. Atomic pointer swap (old → new)
  6. Old trie cleaned up after in-flight requests complete

Propagation time: etcd watch → gateway config updated in < 1s
(etcd push-based watch; not polling)

Rollback: push previous version to etcd
  → all gateway instances revert to previous config within 1s
```

---

## Scaling Strategy
```
Phase 1 (0 → 10K RPS):
  NGINX reverse proxy with static config
  Manual config updates (restart required)
  No circuit breaking

Phase 2 (10K → 50K RPS):
  Custom gateway (or Envoy) with dynamic config
  Redis rate limiting (from Case 03)
  JWT validation in-process
  Basic circuit breaking

Phase 3 (50K → 200K RPS):
  Multiple gateway instances behind L4 LB
  etcd for dynamic config + hot-reload
  mTLS to backends
  Service discovery (Consul or Kubernetes)
  Full circuit breaking with retry logic

Phase 4 (200K+ RPS):
  Envoy proxy with xDS control plane (dynamic config at massive scale)
  HTTP/2 multiplexing to upstreams (reuse connections; much lower overhead)
  Geo-distributed gateways with anycast routing
  WAF integration (Cloudflare, AWS WAF)

Phase 5 (multi-region):
  One gateway cluster per region
  Global load balancing (latency-based routing)
  Consistent config across regions (etcd replication or GitOps)
```

---

## Reliability Strategy
- **Gateway instances**: stateless; L4 LB removes unhealthy instances within 10s
- **etcd down**: gateway continues with last-known config; no config updates until recovery
- **Redis down**: fail-open on rate limiting (from Case 03); gateway continues serving
- **Backend down**: circuit breaker opens; gateway returns 503; no cascade to healthy backends
- **Rolling deploys**: drain connections before stopping; new version starts health check; LB adds after passing
- **Graceful shutdown**: stop accepting new connections; drain in-flight requests (up to 30s); then exit

---

## Security Considerations
- **mTLS to backends**: gateway presents client certificate; backends verify it; no direct backend access
- **WAF before gateway**: block SQL injection, XSS, oversized payloads before gateway logic
- **Scope enforcement**: JWT scopes checked against route's required scope; 403 on mismatch
- **Header stripping**: Authorization header stripped before forwarding; backends never see raw token
- **Request size limit**: reject requests > 10 MB at gateway (before upstream sees them)
- **SSRF prevention**: upstreams defined in config (no client-controlled redirect targets)
- **TLS**: TLS 1.2+ only; strong cipher suites; HSTS headers in responses
- **Secrets**: JWKS refresh URL, etcd creds, Redis password in secrets manager (not config)

---

## Observability
```
Per-request structured log (async emit; never blocks request):
  {
    "request_id": "req-abc",
    "method": "POST",
    "path": "/api/v1/payments",
    "client_id": "client-123",
    "upstream": "payment-service",
    "status_code": 201,
    "latency_total_ms": 187,
    "latency_gateway_ms": 3,        ← gateway overhead only
    "latency_upstream_ms": 184,
    "cache_hit": {"jwks": true, "api_key": false},
    "circuit_breaker_state": "CLOSED",
    "rate_limit_remaining": 743,
    "timestamp": "2024-01-01T14:00:00.123Z"
  }

Metrics:
  gateway_requests_total            (counter; labels: upstream, status_code, route_id)
  gateway_latency_overhead_p99      (target: < 10ms; alert: > 20ms)
  gateway_upstream_latency_p99      (per upstream; for SLO monitoring)
  circuit_breaker_state             (gauge; 0=CLOSED, 1=OPEN, 2=HALF-OPEN; per upstream)
  circuit_breaker_open_duration_s   (alert if any upstream OPEN > 60s)
  rate_limit_denied_total           (from Case 03)
  jwks_cache_refresh_errors_total   (alert if > 0 for > 5 min)
  config_last_updated_timestamp     (gauge; alert if not updated for > 1h when changes expected)

Distributed tracing:
  Generate X-Request-ID on every request
  Propagate to all upstreams as standard header
  Backends log X-Request-ID → full request trace across services
  Integrate with Jaeger or Zipkin for visual trace exploration

Dashboards:
  - Request rate and error rate by upstream and route
  - Gateway overhead latency percentiles (P50, P95, P99)
  - Circuit breaker states (should all be CLOSED in normal operation)
  - Rate limit hit rate by tier
  - Top 10 clients by request volume
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Gateway instance crashes | LB removes within 10s; traffic reroutes to healthy instances | Horizontal scaling; health checks every 5s |
| etcd cluster down | No config updates; gateway serves with last-known config | In-process config cache survives etcd outage; alert on-call |
| Redis down | Rate limiting bypassed (fail-open); gateway continues | From Case 03; fail-open is correct |
| Backend slow (not down) | Requests queue at gateway until timeout | Timeout (5s) + circuit breaker opens at 50% error rate |
| Backend down entirely | Circuit opens; 503 to clients for that backend | Other backends unaffected; alert on-call |
| JWKS key rotation | Old tokens valid up to 5 min (JWKS cache TTL) | Force refresh on 401 from backend; planned rotation is graceful |
| Cascade failure risk | Slow backend causes gateway thread pool exhaustion | Circuit breaker prevents this; timeout per route prevents thread exhaustion |
| WAF outage | Gateway exposed to raw internet without WAF | Secondary WAF instance; CloudFront WAF is HA by default |

---

## Chaos Testing
```
Experiment 1: Kill One Gateway Instance (simulate pod crash)
  Inject:    Kill one gateway pod (SIGKILL; no graceful shutdown)
  Expected:  L4 LB detects unhealthy within 10s; stops routing to it
  Expected:  In-flight requests to that instance fail (TCP reset or timeout)
  Expected:  All other requests route to healthy instances normally
  Expected:  Zero client-visible errors if at least 2 healthy instances remain
  Red flag:  L4 LB takes > 30s to remove failed instance

Experiment 2: Backend Latency Spike (circuit breaker trigger)
  Inject:    Add 6s artificial latency to payment-service (timeout_ms=5000)
  Expected:  Requests to payment-service start timing out (504)
  Expected:  After 10s: 50% error rate threshold crossed → circuit OPENS
  Expected:  Circuit open: gateway returns 503 immediately (no 6s wait)
  Expected:  After 30s: circuit moves to HALF-OPEN → 1 probe request sent
  Expected:  Probe fails (still slow) → circuit re-opens for another 30s
  Red flag:  Circuit never opens (error rate threshold not working)
  Red flag:  Other backends affected (cascade failure)

Experiment 3: etcd Down (config store unavailable)
  Inject:    Stop etcd cluster
  Expected:  Gateway continues serving with last-known route config
  Expected:  No route updates possible; alert fires
  Expected:  No client-visible impact for existing routes
  Red flag:  Gateway crashes when etcd is unavailable
  Red flag:  Gateway starts returning 404 for existing routes

Experiment 4: JWKS Endpoint Down (auth server unavailable)
  Inject:    Block outbound calls to JWKS endpoint (https://auth.internal/jwks)
  Expected:  Gateway uses cached JWKS keys (last refreshed up to 5 min ago)
  Expected:  JWT validation continues normally for up to 5 minutes
  Expected:  JWKS refresh failure logged; metric incremented; alert fires
  Expected:  No client-visible auth failures during 5-min window
  Red flag:  All auth fails immediately (cache not being used)

Experiment 5: Config Change Rollout (bad route introduced)
  Inject:    Push config change with new route to non-existent upstream
  Expected:  Config validation rejects update (upstream not found in service discovery)
  Expected:  Old config remains active; no disruption
  Expected:  Validation error logged with details
  Red flag:  Bad route accepted; all requests to that path start returning 502
```

---

## Monthly Cost Estimate
```
Scale assumption: 100K RPS peak, 50 backend services

API Gateway Instances:
  8 × c5.2xlarge (8 vCPU, 16 GB, $280/mo)       = $2,240/mo
  (CPU-bound: TLS + JWT + routing; 100K RPS / 8 = 12.5K RPS per instance)

Redis Cluster (rate limiting):
  3 × cache.r6g.large ($130/mo)                  = $390/mo

etcd Cluster (config):
  3 × t3.medium ($33/mo)                         = $99/mo
  (etcd is lightweight; small instances sufficient)

L4 Load Balancer (AWS NLB):
  $0.008/LCU-hour + $16/mo base                  = ~$100/mo

PostgreSQL (client registry):
  db.t3.medium ($70/mo)                          = $70/mo

WAF (AWS WAF):
  ~$200/mo (depends on rules and request volume)

Monitoring + networking:
                                                  = $500/mo
────────────────────────────────────────────────────────────
Total: ~$3,599/month at 100K RPS peak

Cost drivers:
  Gateway compute: 62% (CPU-bound: TLS termination + JWT validation)
  Redis:           11%
  Monitoring:      14%

Optimisation levers:
  HTTP/2 multiplexing to upstreams: reduces TLS handshake overhead; lower CPU per request
  JWT verification cache: cache validated token results for their lifetime
    (same token used multiple times → skip RSA verify after first validation)
    Saves ~0.8ms per request for clients making frequent calls
  Reserved instances for stable baseline: ~30% discount
  Graviton2 instances (c7g): better price/performance for compute-bound workloads (~20% faster/cheaper)
```

---

## Migration Story

### Current State
5 microservices each implement their own JWT validation, rate limiting, and logging. Each has a different implementation. Inconsistent error responses. Auth bugs discovered in 3 out of 5 services in the last quarter.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Deploy gateway in transparent passthrough mode (zero downtime)
  Action:
    Deploy gateway cluster; point to same backends as current DNS
    Gateway routes but does NOT enforce auth, rate limiting, or circuit breaking yet
    Compare: gateway response matches direct backend response (diff in staging)
    Gradually shift traffic: 10% → 50% → 100% (weighted DNS or LB)
  Monitor:
    Error rate must not increase (gateway in passthrough mode is transparent)
    Latency: gateway overhead should be < 5ms
  Rollback:
    Shift DNS/LB weight back to 0% for gateway
    Direct traffic continues to backends unchanged

Step 2: Enable auth enforcement at gateway (canary rollout)
  Action:
    Enable JWT validation + API key check in gateway for 10% of traffic
    Leave per-service auth enabled (both gateway and service validate — redundant for now)
    Monitor: 401 rate via gateway should match existing auth rejection rate
    If matching: expand to 100% of traffic through gateway
    Verify: gateway 401s and service 401s match (same tokens rejected)
  Rollback:
    Disable gateway auth enforcement; revert to per-service auth only

Step 3: Remove auth from individual services (per service, staged)
  Action:
    Service by service: remove JWT validation from payment-service
    Add mTLS check: payment-service now trusts only gateway's client certificate
    payment-service uses X-Client-ID header (set by gateway) instead of parsing JWT
    Test: direct call to payment-service without gateway cert → rejected (403)
    Test: call via gateway → succeeds (gateway cert accepted; X-Client-ID present)
    Repeat for each service over 2-week period
  Rollback (per service):
    Re-enable service-level JWT validation; stop mTLS enforcement

Step 4: Enable rate limiting at gateway (zero downtime, start high)
  Action:
    Enable rate limiting with very high initial limits (100× normal expected load)
    Monitor: verify no false 429s; verify X-RateLimit-* headers correct
    Gradually lower limits to target values (10× → 5× → 2× → target)
    Remove per-service rate limiting code
  Rollback:
    Set all gateway limits to 999999 (effectively disabled)
    Re-enable per-service rate limiting

Step 5: Enable circuit breaking (zero downtime, additive)
  Action:
    Deploy with circuit breakers configured (initially with high thresholds: 95% error rate)
    Monitor: circuit breaker state dashboard for all upstreams (should all be CLOSED)
    Gradually tighten thresholds to target (95% → 80% → 50%)
    Test: simulate backend failure → verify circuit opens; 503 returned; other services unaffected
  Rollback:
    Set error_rate_threshold to 1.0 (circuit never opens) — safe no-op while fixing issues
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → Route config: etcd (pushed to all instances via watch).
  → Client registry: PostgreSQL (in-process LRU cache; 30s TTL).
  → JWKS keys: auth server (cached in-process; refreshed every 5 min).
  → Circuit breaker state: in-process per instance (not shared; intentional).

□ Consistency model?
  → Route changes: eventual (< 1s via etcd watch; all instances updated).
  → Auth config: eventual (30s in-process cache; acceptable for client registry).
  → Circuit breaker: independent per instance (eventual; all instances converge within ~10s).

□ Failure mode?
  → Gateway instance down → LB removes within 10s; remaining instances absorb load.
  → etcd down → stale route config; no updates; service continues.
  → Redis down → rate limiting fail-open; gateway continues.
  → Backend down → circuit opens; 503 to clients; other backends unaffected.
  → Verdict: gateway is highly resilient; no single dependency takes it down.

□ Hot partitions?
  → Gateway itself is the concentration point for all traffic.
  → Mitigation: keep gateway thin; horizontal scaling; stateless design.
  → Redis: partitioned by client_id (from Case 03); no single hot key.

□ Cost driver?
  → Gateway compute (62%): CPU-bound due to TLS + JWT.
  → Optimise: HTTP/2 connection reuse reduces TLS overhead significantly.

□ Security risk?
  → mTLS misconfiguration: backend accepts calls without gateway cert.
    Mitigation: integration test that verifies direct backend call is rejected.
  → JWT cache stale after key rotation.
    Mitigation: force JWKS refresh on 401 from backend.
  → Header injection: client sets X-Client-ID before gateway strips it.
    Mitigation: always strip X-Client-ID from inbound client requests before adding gateway's value.

□ Operational burden?
  → etcd cluster: small but requires HA; 3-node minimum.
  → JWKS rotation: must coordinate with auth team; plan rotation not to exceed 5-min cache window.
  → Circuit breaker tuning: thresholds need tuning per upstream (payment-service vs notification-service have different latency profiles).
  → mTLS cert rotation: gateway client cert expires; automated renewal required.

□ Migration path?
  → Passthrough → Auth enforcement (canary) → Remove per-service auth
  → Rate limiting (high limits → tighten) → Circuit breaking.
  → Each step independently rollbackable; no big-bang migration.
```

---

## Staff Engineer Discussion Points

**"The gateway is a single point of failure — how do you address that?"**
Two answers. Operationally: horizontal scaling behind an L4 LB with health checks; losing one instance is transparent. Architecturally: keep the gateway thin. Every feature added to the gateway is a failure mode. If the gateway does auth, rate limiting, routing, circuit breaking, request transformation, and response caching — that's 6 potential failure modes in the critical path. The answer to "SPOF" is not just "add more instances" — it's "do less in the gateway."

**"How do you deploy a bad route config that breaks an existing route?"**
Config validation before etcd write: validate all upstream names resolve, no duplicate path+method, required fields present. Additionally: canary the config change — push to one gateway instance first via a separate etcd key; watch error rate for 60s; if clean, push to all instances. Rollback is instant: push the previous config version to etcd → all instances revert within 1s.

**"At 1M RPS, what becomes the bottleneck?"**
At 1M RPS, the centralised gateway model breaks down. TLS termination and connection management at that scale requires more CPU than is cost-effective in a centralised proxy. The answer is sidecar proxies (Envoy per service): each service gets its own proxy that handles TLS, retries, and circuit breaking. A lightweight centralised gateway remains for edge auth and rate limiting, but most per-service concerns move to the sidecar. This is the service mesh pattern (Istio + Envoy).

**"How does this relate to your ForgeRock and IAM platform work?"**
The API gateway is the enforcement point for your IAM platform. JWT tokens issued by ForgeRock (or your OIDC provider) are validated at the gateway. The gateway enforces scopes from the token — `payments:write`, `payments:read` — against route-level required scopes. This is where ABAC (attribute-based access control) can be enforced at a coarse level. Fine-grained decisions (can this user access this specific payment record?) remain in the backend service with OPA. The gateway handles "can this client call this endpoint type at all" — coarse-grained; backends handle "can this client access this specific resource" — fine-grained.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Single entry point for all services"        → API Gateway Pattern
  "Prevent cascade failures"                   → Circuit Breaker (in-process)
  "Zero-downtime route changes"                → etcd watch + trie hot-swap
  "Auth without network hop per request"       → In-process JWKS cache + JWT validation
  "Backends only reachable via gateway"        → mTLS (mutual TLS)

DECISION TREE USAGE:
  Auth at gateway or per-service?              → Gateway (consistency; one place to fix bugs)
  Circuit breaker state: shared or in-process? → In-process (no Redis dependency in failure path)
  Config propagation: push or poll?            → Push (etcd watch); polling adds lag
  Backend communication: HTTP or mTLS?        → mTLS (cryptographic identity; IP allowlist is weaker)

PATTERNS REJECTED (and why):
  Auth service RPC per request    → +15ms network hop; violates 10ms budget
  Shared circuit breaker (Redis)  → Redis dependency in circuit breaker path; circular failure risk
  Config polling                  → Adds lag; etcd watch is near-instant
  Gateway as business logic layer → Feature creep kills reliability; keep gateway thin
  Single gateway instance         → SPOF; unacceptable for payment platform

SCC LENS:
  STATE:
    Route config (in-process trie): hot-swapped from etcd; O(path_length) lookup
    JWKS keys (in-process cache): refreshed every 5 min; tolerates auth server outage
    Circuit breaker state (in-process): independent per instance; no shared store
    Rate limit counters (Redis): shared across instances; from Case 03

  COORDINATION:
    Config changes: etcd watch propagates to all instances in < 1s (push-based)
    Rate limiting: Redis Lua script for atomic cross-instance counting
    Circuit breaker: NO cross-instance coordination (in-process by design)
    mTLS: mutual certificate verification coordinates trust between gateway and backends

  CONCENTRATION:
    Gateway IS the concentration point for all traffic
    Mitigation: keep gateway thin; stateless; horizontal scaling
    At 1M RPS: decompose to sidecar mesh (Envoy/Istio)
    Redis (rate limiting): sharded by client_id; no single hotspot

INTERVIEW ANSWER TRIGGER:
  "Design an API gateway" →
    1. Spot signals: single entry point, auth at edge, circuit breaking, hot-reload
    2. Pipeline: TLS → route (trie) → auth (JWKS) → rate limit (Redis) → circuit breaker → upstream
    3. Reject: auth service RPC (latency), shared circuit breaker (Redis dependency)
    4. Hot-reload: etcd watch + atomic trie pointer swap
    5. SCC: State=in-process trie+JWKS+circuit, Coordination=etcd push+Redis, Concentration=thin+horizontal
    6. IAM tie-in: gateway enforces coarse-grained scopes; OPA in backend for fine-grained
```

---

*Next: Case Study 06 — Distributed Cache*
