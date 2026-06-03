# Case Study 08 — Feature Flag Service

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: In-Process SDK design, Propagation Flow, Percentage Rollout determinism.

---

## Business Context

A feature flag service exists to decouple code deployment from feature release. Without it, deploying a new feature means simultaneously releasing it to all users — a high-risk, high-stakes event. With flags, code ships dark (inactive), then is gradually revealed: 1% rollout → monitor → 10% → 50% → 100%. The kill switch is the most valuable feature: disable a broken feature in seconds, not hours.

**Business goals driving architecture:**
- Reduce deployment risk: ship code continuously; release carefully
- Kill switch: broken feature disabled in < 10 seconds globally
- SDK resilience: flag service outage must not affect application availability
- A/B testing infrastructure: percentage rollouts for product experiments
- Audit trail: who changed which flag, when — critical for post-incident analysis

---

## Recognition Framework

### Signals from the Problem
```
- 10,000 services need the same flag data → distribution is the core problem
- Flag evaluation called millions of times/sec per service → cannot have network hop per call
- Flag changes must propagate in seconds → streaming required (not just polling)
- Flag service outage must not affect evaluation → SDK must hold local copy
- Total flag data is small (1,000 flags × 2 KB = 2 MB) → entire dataset fits in SDK memory
- Config changes are rare (1,000/day) vs evaluations (billions/day) → write-light, read-heavy
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| In-Process SDK Cache | Zero-latency evaluation; resilient to flag service outage |
| Redis Pub/Sub | Fan-out flag changes from management API to all streaming servers |
| Server-Sent Events (SSE) | Push-based propagation; simpler than WebSocket for one-way stream |
| Polling Fallback | SDK resilience if SSE connection drops |
| Deterministic Percentage Hash | Same user always gets same result for same flag (consistent UX) |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| SDK calls flag service per evaluation | 1ms network hop × billions of evals = catastrophic latency |
| Polling only (no SSE) | Kill switch takes up to 30s; too slow for incident response |
| WebSocket (bidirectional) | Flag propagation is one-way; SSE is simpler and sufficient |
| Kafka for propagation | 10K SDK instances = 10K Kafka consumers; connection overhead; overkill |
| Database polling by SDK | N SDKs × polling interval = N×1/30s DB queries = DB connection storm |
| Shared Redis counter for evaluations | Redis hop per evaluation; defeats the purpose of in-process SDK |

### Why In-Process Evaluation is Non-Negotiable

```
Scenario: payment-service evaluates "new-checkout-flow" flag on every payment request
  Payment-service traffic: 50,000 RPS
  Flag evaluations: 50,000/sec

Option A: SDK calls flag service per evaluation
  50,000 HTTP calls/sec to flag service
  Latency: +1ms per payment for flag lookup
  50,000 payments/sec × 1ms = 50 seconds of cumulative latency/second
  Flag service becomes a SPOF for payments
  Verdict: IMPOSSIBLE at this scale

Option B: In-process SDK evaluation
  SDK holds all flags in a HashMap in-process (2 MB)
  Evaluation: HashMap.get("new-checkout-flow") + rule evaluation
  Latency: ~0.1ms (pure CPU; no network)
  Verdict: CHOSEN — zero coupling to flag service at evaluation time

Consequence: SDK must keep its local copy fresh.
Solution: SSE streaming from flag service to SDK instances.
If SSE fails: polling fallback (30s max lag).
If flag service is completely down: SDK uses last-known state.
```

---

## Problem Statement

Feature flag service for runtime feature toggling without deployments. Evaluate flags in-process (< 1ms). Propagate changes to 10,000 SDK instances within 10 seconds. SDK must evaluate even if flag service is unreachable.

---

## Functional Requirements
- Create/update/delete flags with targeting rules (all users, % rollout, user allowlist, attribute match)
- Evaluate: is_enabled(flag_name, user_context) → true/false
- Kill switch: disable feature globally in < 10 seconds
- Percentage rollout: gradually enable for N% of users (deterministic: consistent per user)
- User allowlist: enable for specific user_ids (internal testing)
- Attribute targeting: enable for users matching country, tier, device type, etc.
- Audit log: every flag change (who, what, when, before, after)
- SDKs: Java, Go, Python, JavaScript

## Non-Functional Requirements
- Evaluation latency: **< 1ms** (in-process; not a network call)
- Change propagation: **< 10 seconds** to all SDK instances
- SDK availability: **evaluate even if flag service is unreachable**
- Consistency: **eventual** (10s propagation lag acceptable)

---

## Capacity Estimation
```
Flag data size:
  1,000 flags × ~2 KB each (flag definition + rules JSON) = ~2 MB total
  This is the ENTIRE dataset — trivially small
  Each SDK instance holds all 2 MB in memory

Propagation:
  1 flag update = 2 KB JSON payload
  10,000 SDK instances need this update
  Network: 10,000 × 2 KB = ~20 MB for one flag change
  SSE connections: 10,000 long-lived HTTP connections per streaming server
  Streaming servers needed: 10,000 connections ÷ 1,000 connections/server = 10 servers

Evaluation throughput:
  payment-service: 50,000 evaluations/sec
  In-process HashMap: handles millions of ops/sec → no concern
  
Flag changes:
  ~1,000 changes/day = ~0.01 changes/sec → write load is negligible
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| In-Process SDK Cache | Zero-latency; decoupled from flag service at runtime |
| Redis Pub/Sub | Fan-out flag change notification to all streaming servers |
| SSE Streaming | Push-based; < 10ms propagation; simpler than WebSocket |
| Polling Fallback | Resilience: SDK polls every 30s if SSE disconnects |
| Deterministic Hash | Consistent user experience: same user, same flag, same result |

---

## Data Model

```sql
-- Flag definitions (PostgreSQL — authoritative store)
CREATE TABLE feature_flags (
  flag_key       VARCHAR(128)  PRIMARY KEY,
  display_name   VARCHAR(256),
  description    TEXT,
  is_active      BOOLEAN       DEFAULT FALSE,  -- global kill switch
  default_value  BOOLEAN       DEFAULT FALSE,  -- value when no rules match
  rules          JSONB         NOT NULL,       -- targeting rules (see below)
  owner_team     VARCHAR(64),
  created_by     VARCHAR(64),
  updated_by     VARCHAR(64),
  created_at     TIMESTAMP     DEFAULT NOW(),
  updated_at     TIMESTAMP     DEFAULT NOW()
);

-- Rules JSONB schema:
-- {
--   "rules": [
--     {
--       "type": "allowlist",
--       "attribute": "user_id",
--       "values": ["u-123", "u-456", "u-789"]
--     },
--     {
--       "type": "percentage",
--       "percentage": 10,
--       "salt": "new-checkout-flow-v1"  -- versioned salt for consistency
--     },
--     {
--       "type": "attribute",
--       "attribute": "country",
--       "operator": "in",
--       "values": ["IN", "SG", "MY"]
--     },
--     {
--       "type": "attribute",
--       "attribute": "user_tier",
--       "operator": "eq",
--       "values": ["premium"]
--     }
--   ],
--   "default": false
-- }

-- Immutable audit log (append-only; never UPDATE or DELETE)
CREATE TABLE flag_audit_log (
  id          BIGSERIAL     PRIMARY KEY,
  flag_key    VARCHAR(128)  NOT NULL,
  changed_by  VARCHAR(64)   NOT NULL,
  old_value   JSONB,                    -- null on creation
  new_value   JSONB         NOT NULL,
  change_type VARCHAR(32)   NOT NULL,   -- CREATE / UPDATE / DELETE / ACTIVATE / DEACTIVATE
  changed_at  TIMESTAMP     DEFAULT NOW()
);
CREATE INDEX idx_audit_flag_key ON flag_audit_log(flag_key, changed_at DESC);

-- SDK keys (for authenticating SDK sync requests)
CREATE TABLE sdk_keys (
  sdk_key      VARCHAR(64)   PRIMARY KEY,
  service_name VARCHAR(128),
  environment  VARCHAR(32),             -- production / staging / development
  is_active    BOOLEAN       DEFAULT TRUE,
  created_at   TIMESTAMP     DEFAULT NOW()
);
```

---

## API Design

```
Management API (used by engineers via UI or CLI):

POST /api/v1/flags
Body: {
  "flag_key": "new-checkout-flow",
  "display_name": "New Checkout Flow",
  "rules": {
    "rules": [{"type": "percentage", "percentage": 10, "salt": "v1"}],
    "default": false
  }
}
Response 201: { "flag_key": "new-checkout-flow", "created_at": "..." }

PUT /api/v1/flags/{flag_key}
Body: { "is_active": true, "rules": { ... } }
Response 200: updated flag object

PATCH /api/v1/flags/{flag_key}/activate    → set is_active = true (kill switch ON)
PATCH /api/v1/flags/{flag_key}/deactivate  → set is_active = false (KILL SWITCH)
Response 200: { "flag_key": "...", "is_active": false, "propagated_at": "..." }

GET /api/v1/flags/{flag_key}/audit?limit=50
Response 200: { "changes": [...] }

SDK Sync API (called by SDK; not by engineers):

GET /sdk/v1/flags
Headers: Authorization: SDK-Key {sdk_key}
Response 200: {
  "flags": {
    "new-checkout-flow": {
      "is_active": true,
      "default_value": false,
      "rules": { ... }
    },
    ...
  },
  "version": 4821,   // monotonic counter; SDK uses to detect missed updates
  "fetched_at": "2024-01-01T14:00:00Z"
}

GET /sdk/v1/stream
Headers: Authorization: SDK-Key {sdk_key}
Accept: text/event-stream
Response: SSE stream
  event: flag_updated
  data: {"flag_key":"new-checkout-flow","is_active":true,"rules":{...},"version":4822}

  event: flag_deleted
  data: {"flag_key":"old-feature","version":4823}

  event: heartbeat
  data: {"version":4823,"timestamp":"2024-01-01T14:01:00Z"}
  (every 30s — SDK detects stale connection if heartbeat missing for > 60s)
```

---

## High-Level Architecture

```
Engineer → Management UI / CLI
  │
  ▼
Flag Management API
  ├── Write flag to PostgreSQL
  ├── Write to flag_audit_log
  └── PUBLISH flag change to Redis channel "flag-updates"
        Payload: { flag_key, new_definition, version, change_type }

Redis Pub/Sub
  Channel: "flag-updates"
  │
  ▼
Flag Streaming Service (10 instances; each holds ~1K SSE connections)
  ├── Subscribes to Redis "flag-updates" channel
  ├── On message received: push SSE event to all connected SDK instances
  └── Heartbeat: send "heartbeat" SSE event every 30s to all connections

SDK (embedded in each application: payment-service, user-service, etc.)
  ├── Startup:
  │     GET /sdk/v1/flags → load all ~2 MB of flags into HashMap
  ├── Runtime:
  │     Open SSE connection to Streaming Service (any instance via LB)
  │     On SSE event received: update in-process HashMap atomically
  │     On SSE disconnect: reconnect after 5s; poll every 30s as fallback during gap
  ├── Evaluation:
  │     HashMap.get(flag_key) → evaluate rules against UserContext → boolean
  │     Total time: ~0.1ms (no network; pure in-process)
  └── Resilience:
        If flag service completely unreachable:
          Use last-known HashMap state (never throw; never block)
          Log warning; emit metric; alert if > 5 min unreachable

Application code:
  if (featureFlags.isEnabled("new-checkout-flow", userContext)) {
    // new checkout path
  } else {
    // old checkout path
  }
  // This line executes in 0.1ms regardless of flag service health
```

---

## Detailed Components

### In-Process SDK Evaluation (Java)
```java
public class FeatureFlagSDK {

    // ConcurrentHashMap: thread-safe; read-optimized; allows concurrent updates
    private final ConcurrentHashMap<String, FlagDefinition> flagCache =
        new ConcurrentHashMap<>();

    public boolean isEnabled(String flagKey, UserContext user) {
        FlagDefinition flag = flagCache.get(flagKey);

        // Unknown flag: return false (safe default)
        if (flag == null) {
            metrics.increment("flag.unknown", tag("key", flagKey));
            return false;
        }

        // Global kill switch: is_active = false means disabled for everyone
        if (!flag.isActive()) {
            return false;
        }

        // Evaluate rules in order; return true on first match
        for (Rule rule : flag.getRules()) {
            if (evaluateRule(rule, user)) {
                return true;
            }
        }

        // No rules matched: return default value
        return flag.getDefaultValue();
    }

    private boolean evaluateRule(Rule rule, UserContext user) {
        return switch (rule.getType()) {
            case ALLOWLIST -> rule.getValues().contains(user.getUserId());
            case PERCENTAGE -> isInPercentage(user.getUserId(), rule);
            case ATTRIBUTE -> matchAttribute(user, rule);
        };
    }

    private boolean isInPercentage(String userId, PercentageRule rule) {
        // Deterministic: same userId + same salt → same result always
        int hash = Math.abs(murmur3_32(userId + ":" + rule.getSalt())) % 100;
        return hash < rule.getPercentage();
        // hash is 0-99; if percentage=10, users with hash 0-9 are enabled (10%)
    }
}
```

### Percentage Rollout — Why Determinism Matters
```
Problem with random():
  User visits checkout 3 times in a row.
  Each evaluation: random() % 100 < 10 → different result each time
  User sees new checkout on attempt 1, old on attempt 2, new on attempt 3
  Experience: broken, inconsistent, confusing

Problem with user_id % 100:
  users 0, 100, 200, 300... are always in the same bucket for ALL flags
  Adding a new 10% rollout always enables it for the same 10% of users
  Those users become "flag guinea pigs" for every experiment
  Biased experimentation results

Solution: deterministic hash with per-flag salt
  hash = murmur3_32(user_id + ":" + flag_salt) % 100
  if hash < target_percentage: enabled

  Why murmur3?
    Fast (non-cryptographic; no security requirement here)
    Good distribution (uniform hash)
    Deterministic: same inputs → always same output

  Why per-flag salt?
    salt = "new-checkout-flow-v1"
    user_id="u-123" always gets hash=47 for new-checkout-flow
    user_id="u-123" gets a DIFFERENT hash for "new-payment-method" (different salt)
    Prevents correlation: a user isn't always "in" or "out" for every experiment

  Why version in salt ("v1")?
    If you need to re-roll (reset who is in the experiment):
    Change salt: "new-checkout-flow-v2" → everyone gets new hash → fresh randomization
    Users in v1 don't necessarily stay in v2 (desired for experiment reset)
```

### SSE Propagation — Full Timeline
```
T+0ms:    Engineer clicks "deactivate" on new-checkout-flow (kill switch)
T+1ms:    Management API receives request
T+2ms:    PostgreSQL UPDATE feature_flags SET is_active = false
T+3ms:    Management API writes to flag_audit_log
T+4ms:    Redis PUBLISH "flag-updates" {flag_key: "new-checkout-flow", is_active: false, ...}
T+5ms:    All 10 Streaming Service instances receive Redis Pub/Sub message
T+6–10ms: Each Streaming Service pushes SSE event to its connected SDK instances
           10 streaming servers × 1,000 connections = 10,000 SDKs notified
T+10ms:   All 10,000 SDK instances update their in-process HashMap
           flagCache.put("new-checkout-flow", updatedDefinition)
T+10ms:   All subsequent evaluations return false (flag is deactivated)

Total end-to-end propagation: ~10ms
Kill switch effectiveness: flag is disabled across entire fleet within 10ms

Worst case (SDK on polling fallback; SSE disconnected):
  SDK polls every 30s → up to 30s delay
  Kill switch still effective within 30s — acceptable for non-critical incidents
```

### SSE Connection Management
```
SDK connects to Streaming Service:
  GET /sdk/v1/stream HTTP/1.1
  Accept: text/event-stream
  Connection: keep-alive
  Authorization: SDK-Key {sdk_key}

Streaming Service response:
  HTTP/1.1 200 OK
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive
  
  event: connected
  data: {"version": 4821, "timestamp": "..."}
  
  [heartbeat every 30s:]
  event: heartbeat
  data: {"version": 4821}
  
  [on flag change:]
  event: flag_updated
  data: {"flag_key": "...", "is_active": true, "rules": {...}, "version": 4822}

SDK reconnection logic:
  On SSE disconnect (network error, server restart):
    Wait 5 seconds (exponential backoff: 5s → 10s → 30s max)
    Reconnect: GET /sdk/v1/stream
    On reconnect: fetch full flag set (GET /sdk/v1/flags) to catch missed updates
    Compare version numbers: if version gap > 0, missed updates → full refresh
```

---

## Scaling Strategy
```
Phase 1 (0 → 10 services):
  PostgreSQL + SDK polling (30s interval)
  No streaming; kill switch lag = 30s
  Simple; low ops burden; acceptable for small teams

Phase 2 (10 → 100 services):
  Add Redis Pub/Sub + SSE streaming
  Kill switch lag: < 10s
  1–2 Streaming Service instances handle 100 × 10 SDK instances per service = 1K connections

Phase 3 (100 → 1,000 services):
  Scale Streaming Service horizontally (10 instances × 1K connections = 10K connections)
  Redis Pub/Sub handles fan-out to all Streaming instances
  SDK keys per service type (isolate environments: prod, staging, dev)

Phase 4 (1,000+ services or multi-region):
  Regional flag services (one per region)
  PostgreSQL replicated across regions
  Accept brief cross-region propagation lag (< 5s from primary → regional replicas)
  OR: global Redis Pub/Sub (adds ~50ms cross-region RTT but simpler)

Phase 5 (edge evaluation):
  Client-side flags (browser/mobile):
  CDN edge workers evaluate flags (Cloudflare Workers / Lambda@Edge)
  Flag definitions pushed to edge cache
  Zero network round-trip for client-side flag evaluation
```

---

## Reliability Strategy
- **SDK resilience**: flag service completely down → SDK uses last-known HashMap state; zero application errors
- **Streaming service crash**: SDK detects SSE disconnect; falls back to polling; reconnects on recovery
- **PostgreSQL down**: flag changes fail (acceptable); current flag state in all SDKs unchanged; flag service returns 503
- **Redis Pub/Sub down**: streaming stops; SDKs fall to polling (30s lag); flag service alert fires
- **Flag default values**: every flag has a default (`false`); unknown flags return `false`; safe degradation

---

## Security Considerations
- **SDK key isolation**: each service gets a unique SDK key; key scoped to read-only flag access
- **Management API auth**: engineer authentication + RBAC (only flag owners can modify their flags)
- **Audit trail immutability**: flag_audit_log uses append-only access; no UPDATE or DELETE
- **Flag key naming**: enforce `{team}.{service}.{feature}` namespace; prevent collision
- **Sensitive targeting rules**: user_id allowlists visible to SDK → use anonymous IDs for sensitive segments
- **Rate limiting on Management API**: prevent rapid flag changes that could disrupt services
- **SDK key rotation**: annual rotation; old key accepted for 24h after new key issued (no downtime)

---

## Observability
```
Management metrics:
  flag_change_total          (counter; by change_type and team)
  flag_active_count          (gauge; how many flags are currently active)
  rollout_percentage_by_flag (gauge; current rollout % per flag)

Streaming metrics:
  sse_connections_active     (gauge per streaming server; target: < 1,000)
  sse_message_latency_ms     (time from Redis pub to SSE delivery; target: < 5ms)
  sse_reconnect_rate         (counter; SDK reconnections per hour)

SDK metrics (emitted from SDK to application's metric system):
  flag_evaluation_total      (counter; by flag_key and result)
  flag_cache_age_seconds     (gauge; time since last flag update received)
  sdk_using_stale_state      (gauge; 1 if flag service unreachable for > 5 min)

Business metrics (most valuable):
  error_rate_by_flag_variant  (compare error rate for enabled vs disabled users)
  latency_by_flag_variant     (P99 latency for enabled vs disabled)
  conversion_rate_by_flag     (A/B test outcome measurement)

Alerts:
  sse_connections_active drops significantly → streaming service issue
  sdk_using_stale_state = 1 for > 5 min    → flag service unreachable; stale flags in use
  flag_change_total spikes abnormally       → automation bug or security event
  error_rate increases during rollout       → trigger kill switch analysis
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Flag service API down | No new changes; all SDKs use last-known state | SDK resilient; alert on-call; flag service is not in eval hot path |
| Redis Pub/Sub down | No streaming updates; SDKs fall to 30s polling | Polling fallback; kill switch lag = 30s max |
| Streaming service crashes | SSE connections drop; SDKs reconnect and poll | Auto-reconnect with backoff; polling fallback during gap |
| Bad flag change (logic error) | Feature broken for % of users | Kill switch (deactivate flag): propagates in ~10ms |
| PostgreSQL down | Cannot update flags; read from Redis cache | Cache last flag state in Redis; alert on-call |
| SDK version bug (wrong evaluation) | Feature incorrectly enabled/disabled | SDK is versioned; rollback SDK version; flag evaluation is separate from business logic |

---

## Chaos Testing
```
Experiment 1: Kill Flag Service API
  Inject:    Stop flag service API (all instances)
  Expected:  SDK evaluations continue from last-known HashMap state
  Expected:  No application errors; flag evaluations return last-known values
  Expected:  sdk_using_stale_state metric = 1; alert fires after 5 min
  Expected:  On flag service recovery: SDKs reconnect SSE; full flag refresh
  Red flag:  Application throws exception when flag service is unreachable
  Red flag:  Application returns 503 because flag service is down (complete coupling)

Experiment 2: Kill Streaming Service (SSE failure)
  Inject:    Stop all streaming service instances
  Expected:  SDKs detect SSE disconnect (heartbeat timeout)
  Expected:  SDKs switch to polling mode (GET /sdk/v1/flags every 30s)
  Expected:  Flag changes still propagate within 30s via polling
  Expected:  Kill switch effective within 30s (not 10ms, but acceptable)
  Red flag:  SDKs stop evaluating flags (blocking on reconnect)

Experiment 3: Kill Switch End-to-End Test
  Inject:    Enable flag for 100% of users; observe error rate spike
             Trigger kill switch: PATCH /api/v1/flags/new-checkout-flow/deactivate
  Expected:  Within 10ms: all SSEs push deactivation event
  Expected:  All SDK instances update HashMap: is_active = false
  Expected:  error_rate drops immediately (within 1 request after propagation)
  Expected:  Total time from kill switch click to error rate drop: < 15s
  Red flag:  Kill switch takes > 30s to propagate (SSE not working; polling fallback)

Experiment 4: Percentage Rollout Consistency
  Inject:    Enable flag for 10% of users; call isEnabled() 1,000× for same user_id
  Expected:  All 1,000 evaluations return same result (deterministic hash)
  Expected:  Approximately 10% of unique user_ids return true (uniform distribution)
  Red flag:  Same user gets different results on different calls (non-deterministic)
  Red flag:  All users get same result (hash function broken; no distribution)

Experiment 5: Redis Pub/Sub Failure
  Inject:    Stop Redis cluster
  Expected:  Streaming servers lose Pub/Sub connection
  Expected:  No new flag changes pushed via SSE
  Expected:  SDKs fall to 30s polling as fallback
  Expected:  Flag changes made during outage: picked up on next poll
  Expected:  Alert fires within 1 minute
  Red flag:  Streaming servers crash (not handling Redis disconnect gracefully)
```

---

## Monthly Cost Estimate
```
Scale assumption: 10,000 SDK instances, 1,000 flags, 1,000 flag changes/day

Flag Management API:
  2 × t3.small ($15/mo)                         = $30/mo
  (flag changes are rare; tiny traffic)

PostgreSQL:
  db.t3.medium ($70/mo)                         = $70/mo
  (2 MB of flags; tiny; small instance fine)

Redis (Pub/Sub):
  cache.t3.medium ($30/mo)                      = $30/mo
  (Pub/Sub is lightweight; no data stored)

Streaming Service (SSE):
  10 × t3.small ($15/mo)                        = $150/mo
  (10,000 SSE connections ÷ 1,000/server = 10 servers)
  (each SSE connection is a long-lived HTTP; low CPU but high connection count)

Load Balancer:
  $30/mo

Monitoring:
  $50/mo
────────────────────────────────────────────────────────────
Total: ~$360/month

Comparison:
  LaunchDarkly Pro (equivalent scale): $400–$4,000/month
  Statsig (similar): $300–$2,000/month
  Build vs buy: at this scale, build is cheaper AND gives full control
  Build when: team has ops capacity; need deep customisation; data sovereignty required
  Buy when: team is small; speed to market matters; ops burden is too high

Cost driver:
  Streaming service (42%): connection count drives horizontal scaling
  Optimisation: consider long-polling instead of SSE (simpler connection management)
  OR: use WebSocket multiplexing (many logical streams over fewer connections)
```

---

## Migration Story

### Current State
Feature flags are hardcoded constants in application code:
```java
static final boolean NEW_CHECKOUT_FLOW = false; // change this and deploy
```
Changing a flag requires code change + PR review + deployment (30–60 min process). Kill switch = emergency deployment. No audit trail. No percentage rollout.

### Migration Plan (Each Step Independently Rollbackable)

```
Step 1: Build flag service with polling-only SDK (zero downtime)
  Action:
    Deploy PostgreSQL + flag management API
    Build SDK with 30s polling (no SSE yet)
    Integrate SDK into 1 service (e.g. payment-service)
    Migrate 5 hardcoded flags to flag service
    Keep hardcoded constant as fallback: if SDK returns error → use constant
  Monitor:
    SDK polling success rate (should be > 99%)
    Flag evaluation accuracy (compare SDK result vs hardcoded constant)
  Rollback:
    Remove SDK; revert to hardcoded constants in code

Step 2: Roll out SDK to all services (service by service)
  Action:
    Add SDK to each service (1 per week)
    Each service: SDK + hardcoded fallback until confidence is high
    After 2 weeks stable: remove hardcoded fallback
  Monitor:
    No unexpected flag evaluation changes (monitor error rates during each rollout)
  Rollback:
    Per service: remove SDK; revert to constants

Step 3: Add SSE streaming (zero downtime)
  Action:
    Deploy Streaming Service (Redis Pub/Sub + SSE)
    Update SDK: try SSE first; fall back to 30s polling on disconnect
    Add version number tracking to detect missed updates
  Monitor:
    SSE connection count (should grow as services adopt new SDK)
    Propagation latency: from flag change to SDK update (target: < 10ms)
    SDK reconnect rate (should be < 1/hour per SDK instance)
  Rollback:
    SDK: disable SSE; use polling only (simple config change; no re-deploy)

Step 4: Remove hardcoded constants from codebase (planned, service by service)
  Action:
    For each service: identify all hardcoded feature flags
    Migrate to flag service (if not already)
    Remove the constant; rely entirely on SDK
    Add flag to "permanent flags" list (flags that are intended to be permanent)
  Monitor:
    No increase in error rates after constant removal
  Rollback:
    Re-add hardcoded constant as fallback

Step 5: Enable audit log + management UI (zero downtime)
  Action:
    Deploy management UI (web app for non-engineers to manage flags)
    Ensure flag_audit_log captures all changes
    Integrate with PagerDuty: flag changes during incidents auto-create timeline entries
  Rollback:
    Not needed (additive features; no behavioural change)
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL for flag definitions (authoritative; durable).
  → SDK in-process HashMap: a copy of PostgreSQL state; refreshed via SSE/polling.
  → If SDK and PostgreSQL disagree: PostgreSQL wins; next SSE update or poll corrects it.

□ Consistency model?
  → Eventual: < 10ms via SSE; < 30s via polling fallback.
  → Flag change → SDK update is NOT atomic; there is always a propagation window.
  → During propagation: some SDK instances have new state; others have old state.
  → This is acceptable: flag changes are not financial transactions.

□ Failure mode?
  → Flag service down → SDK uses last-known state (no application errors).
  → SSE down → SDK falls to polling (30s max lag; kill switch still effective).
  → Redis Pub/Sub down → streaming stops; polling fallback.
  → In all cases: SDK never blocks; never throws; evaluates from last-known state.

□ Hot partitions?
  → Streaming service: 10,000 SSE connections spread across 10 servers.
  → If one streaming server crashes: its 1,000 SDKs reconnect to other servers.
  → Redis Pub/Sub: single channel "flag-updates"; 10 streaming servers subscribe.

□ Cost driver?
  → Streaming service (42%): driven by SSE connection count.
  → At 100K SDK instances: need 100 streaming servers → $1,500/mo for streaming alone.
  → Optimisation: multiplex SSE connections (fewer servers, more connections each).

□ Security risk?
  → SDK key exposed in client-side JavaScript → reveals flag names/targeting rules.
    Mitigation: use anonymous user IDs in browser SDK; don't expose server-side flags to browser.
  → Flag changes without audit trail → cannot reconstruct incident timeline.
    Mitigation: flag_audit_log is mandatory; all changes go through Management API.
  → Rapid flag changes (automated abuse) → flag service as DoS amplifier.
    Mitigation: rate limiting on Management API; max 10 changes/min per flag.

□ Operational burden?
  → Streaming service: SSE connection management; reconnect handling; heartbeat monitoring.
  → SDK upgrades: when SDK has a bug, must coordinate upgrade across all services.
  → Flag lifecycle: permanent flags accumulate; need flag retirement process.
  → Percentage rollout: need statistical analysis tooling to measure A/B test outcomes.

□ Migration path?
  → Hardcoded constants → polling SDK → SSE streaming → UI/audit log.
  → Each step independently rollbackable; SDK has hardcoded fallback during transition.
```

---

## Staff Engineer Discussion Points

**"How is this different from A/B testing?"**
Feature flags are operational: "can we deploy this safely and roll it back if it breaks?" A/B testing is product: "does this variant improve our metric with statistical significance?" They share infrastructure (percentage rollout, user bucketing, deterministic hash) but serve different purposes. Feature flags prioritise safety and speed of deployment. A/B tests prioritise statistical correctness (experiment isolation, sample size calculation, significance testing). You can build A/B testing infrastructure on top of a feature flag system, but they're not the same thing. A bad A/B test design (too short, biased segments) produces misleading results; a bad feature flag design (non-deterministic) breaks user experience.

**"How do you handle a gradual rollout that triggers a cascade failure?"**
Cascade failure during rollout is the classic scenario: 1% rollout → looks fine → 10% → DB CPU spikes. Pattern: (1) Monitor error rate AND latency AND DB metrics during every percentage increase. (2) Set automatic rollback: if error_rate increases by > 1% during rollout → automatically set flag to 0% (automatic kill switch). (3) Keep rollout increments small and slow: 1% → wait 10 min → 5% → wait 30 min → 25% → wait 1h → 100%. (4) Canary deployment pattern: rollout by server first (1 server = 5% of traffic), not by user percentage (more controlled blast radius).

**"What's your flag lifecycle strategy — how do flags retire?"**
Flags accumulate over time. After a feature is fully rolled out to 100%, the flag becomes dead code — it always returns true; the old code path is never used. If you don't clean up flags: (1) codebase accumulates dead code paths, (2) flag service accumulates stale flags (operational debt), (3) engineers making new flag changes have a harder time understanding which flags are still relevant. Strategy: (1) Every flag must have an owner team and a planned retirement date set at creation. (2) After 100% rollout for 30 days → automated reminder to owner to delete flag and remove code path. (3) "Permanent flags" (kill switches for critical features) are explicitly marked as permanent and exempt from retirement.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Push config to 10K services in seconds"         → Config Distribution: Redis Pub/Sub + SSE
  "Evaluate rule without network call"             → In-process SDK cache (HashMap)
  "Same user always gets same result"              → Deterministic percentage hash (murmur3 + salt)
  "Kill switch: disable in < 10s"                  → SSE streaming (not polling)
  "SDK resilient to flag service outage"           → Last-known state; never throw; never block

PROPAGATION DECISION:
  Need < 1s propagation?                           → WebSocket (bidirectional overkill for flags)
  Need < 10s propagation?                          → SSE (one-way; simpler; sufficient)
  Need < 30s propagation?                          → Polling (simplest; fine for non-critical flags)

PATTERNS REJECTED (and why):
  SDK RPC per eval      → 1ms × billions of evals; flag service becomes SPOF for all services
  Polling only          → 30s kill switch lag; too slow for incidents
  Kafka for propagation → 10K Kafka consumers; connection overhead; overkill for 2 MB of data
  DB polling by SDK     → N SDKs × 1/30s = connection storm at scale
  Random() for % rollout → non-deterministic; same user gets different results per call

SCC LENS:
  STATE:
    Authoritative: PostgreSQL (flag definitions; 2 MB; rarely changes)
    Propagated copy: in-process HashMap in each SDK (2 MB × 10K services = 20 GB fleet-wide)
    Immutable log: flag_audit_log (compliance; incident reconstruction)

  COORDINATION:
    Redis Pub/Sub: flag management API publishes; streaming service subscribes
    SSE: streaming service pushes to SDK instances (one-way; no ack)
    Version numbers: SDK uses version counter to detect missed updates (full refresh on gap)
    Deterministic hash: "coordinates" which users are in each experiment cohort (no state needed)

  CONCENTRATION:
    Streaming service: 10,000 SSE connections → must scale horizontally (10 servers)
    Redis Pub/Sub: single fan-out point → if Redis down, streaming stops (polling fallback)
    SDK HashMap: each service holds its own copy → no shared concentration point at eval time

IAM PLATFORM CONNECTION:
  Feature flags in IAM context:
    "enable new auth policy for 10% of users" → percentage rollout flag
    "disable legacy SAML endpoint for tenant X" → allowlist flag (tenant_id)
    "enable OPA policy enforcement for payments only" → attribute flag (service_name)
    Kill switch for a broken auth policy → immediate deactivation (propagates in 10ms)
  ForgeRock context:
    ForgeRock Authentication Trees can be wrapped in feature flags
    New authentication journey for mobile: flag-controlled; gradual rollout
    FAPI compliance features: flag-gated until fully tested

INTERVIEW ANSWER TRIGGER:
  "Design a feature flag service" →
    1. Core insight: evaluation MUST be in-process (not network call)
    2. Distribution: SSE streaming for < 10s propagation; polling fallback
    3. Deterministic hash: murmur3(userId + ":" + flag_salt) % 100 (not random())
    4. SDK resilience: flag service down → use last-known state; never block
    5. Kill switch: deactivate → Redis Pub/Sub → SSE → SDK update in ~10ms
    6. SCC: State=PostgreSQL+SDK HashMaps; Coordination=Redis Pub/Sub+SSE;
            Concentration=streaming service (horizontal scaling)
```

---

*Next: Case Study 09 — URL Reputation Service*
