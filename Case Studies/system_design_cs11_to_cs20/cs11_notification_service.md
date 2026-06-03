# Case Study 11 — Notification Service

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Multi-channel fan-out, Deduplication, Rate Limiting per user.

---

## Business Context

A notification service sends messages across email, SMS, push, and in-app channels. For a payment platform, notifications are transactional (payment confirmed, OTP, fraud alert) and marketing (offers, reminders). Transactional notifications have near-zero tolerance for failure and delivery latency. Marketing notifications have cost and frequency constraints.

The hardest problem is not sending — it is knowing when NOT to send: duplicate suppression, user preference respect, frequency caps, and channel fallback when primary delivery fails.

**Business goals driving architecture:**
- Transactional (OTP, fraud alert): deliver within 10 seconds; retry on failure; at-least-once
- Marketing: respect frequency caps; user preferences; unsubscribe; batching for cost
- Exactly-once delivery semantics: payment confirmation must not send twice
- Channel fallback: if push fails, try SMS; if SMS fails, try email
- Cost: SMS is $0.004/message; push is free; email is $0.001/message → prefer cheaper channels

---

## Recognition Framework

### Signals from the Problem
```
- Multiple channels (email, SMS, push, in-app) with different latency/cost
- High fan-out: one event → potentially millions of notifications (marketing blast)
- Transactional vs marketing have different reliability/cost tradeoffs
- Duplicate prevention: payment confirmation sent twice = user confusion + refund request
- User preferences: DND hours, channel preferences, unsubscribes
- Channel failure is expected: push token expires, SMS undeliverable → need fallback
- Cost per channel varies 100×: push (free) vs SMS ($0.004) → channel selection matters
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Priority Queues (Kafka) | Transactional and marketing have different SLOs; must not block each other |
| Fan-out Worker | One event → multiple channel workers; parallel delivery |
| Idempotency Key | Prevent duplicate sends on retry; payment confirmation exactly-once |
| Channel Fallback Chain | Push → SMS → email; try cheapest first; escalate on failure |
| Rate Limiter per User | Frequency cap: max 3 notifications/day for marketing per user |
| Dead Letter Queue | After N retries, route to DLQ for manual review; never silently drop |

### Patterns Rejected
| Pattern | Reason Rejected |
|---|---|
| Single queue for all channels | Marketing blasts starve transactional OTPs; priority queues required |
| Synchronous delivery in request path | Sending email/SMS is slow (100–500ms); must be async |
| No idempotency | Retry storms on failure → duplicate OTPs/payment confirmations → user confusion |
| Fire-and-forget (no retry) | Push token expiry, SMTP transient failures are common; must retry with backoff |
| Send all channels simultaneously | Cost explosion; user annoyance; channel fallback is correct model |

---

## Problem Statement

Multi-channel notification service. Ingests notification requests from platform services, routes to the correct channel(s), enforces user preferences and frequency caps, handles retries, prevents duplicates, and provides delivery status.

---

## Functional Requirements
- Send: email, SMS, push notification, in-app notification
- Channel selection: based on notification type, user preference, cost optimization
- Channel fallback: if primary fails, try secondary
- Deduplication: idempotency key prevents same notification being sent twice
- User preferences: DND hours, channel opt-out, unsubscribe
- Frequency cap: max N notifications/day per user per category
- Delivery status: delivered, failed, bounced, unsubscribed
- Templates: parameterized message templates with variable substitution

## Non-Functional Requirements
- Transactional latency: **< 10 seconds end-to-end** (ingestion → delivery)
- Marketing throughput: **1 million notifications/hour** for blast campaigns
- Duplicate rate: **< 0.001%** (payment confirmation must not duplicate)
- Availability: **99.9%** for transactional; **99%** for marketing

---

## Capacity Estimation
```
Transactional notifications:
  Peak: 10,000/sec (payment OTPs during high traffic)
  Avg: 1,000/sec

Marketing blasts:
  1M users × 1 notification/hour = ~278/sec average
  Burst: all at once = 1M notifications queued in seconds

Delivery channel volumes:
  Push: 50% of all notifications (free; prefer first)
  Email: 30% ($0.001/email = $300/month at 1M/day)
  SMS: 20% ($0.004/SMS = $2,400/month at 600K/day)
  Total channel cost: ~$2,700/month at scale

Idempotency store:
  10K idempotency keys/sec × TTL 24h = 864M keys/day
  But only deduplicate recent window (24h): 10K RPS × 86400s = 864M keys
  At 100 bytes/key: 86 GB → use Redis with 24h TTL; ~86 GB Redis
  Optimization: only store keys for transactional (not marketing); reduces to ~5 GB
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Kafka Priority Topics | Separate topics for transactional vs marketing; different consumer SLOs |
| Fan-out per Notification | Parallel channel delivery; independent retry per channel |
| Idempotency Key (Redis) | Dedup: SET NX with 24h TTL; second attempt returns cached result |
| Channel Fallback Chain | Prefer push → SMS → email based on cost and delivery probability |
| User Preference Store | Redis cache of user preferences (DND, opt-out); checked before every send |
| DLQ + Retry | Exponential backoff; DLQ after 5 retries; alert on DLQ growth |

---

## Data Model

```sql
-- Notification definitions (PostgreSQL)
CREATE TABLE notifications (
  notification_id   UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key   VARCHAR(128)  UNIQUE,            -- caller-supplied; dedup key
  type              VARCHAR(64)   NOT NULL,           -- OTP / PAYMENT_CONFIRMATION / MARKETING
  priority          VARCHAR(16)   NOT NULL,           -- TRANSACTIONAL / MARKETING
  recipient_user_id VARCHAR(64)   NOT NULL,
  channels_requested TEXT[]       NOT NULL,           -- ['push', 'sms', 'email']
  template_id       VARCHAR(64),
  template_vars     JSONB,
  status            VARCHAR(32)   DEFAULT 'PENDING',  -- PENDING / SENT / FAILED / DEDUPLICATED
  created_at        TIMESTAMP     DEFAULT NOW(),
  sent_at           TIMESTAMP
);

-- Delivery attempts (per channel per notification)
CREATE TABLE delivery_attempts (
  attempt_id        BIGSERIAL     PRIMARY KEY,
  notification_id   UUID          REFERENCES notifications(notification_id),
  channel           VARCHAR(16)   NOT NULL,           -- push / sms / email / in_app
  attempt_num       INTEGER       DEFAULT 1,
  status            VARCHAR(32),                      -- SENT / FAILED / BOUNCED / UNSUBSCRIBED
  provider          VARCHAR(64),                      -- FCM / APNS / Twilio / SendGrid
  provider_msg_id   VARCHAR(128),                     -- provider's message ID for status lookup
  attempted_at      TIMESTAMP     DEFAULT NOW(),
  delivered_at      TIMESTAMP,
  error_code        VARCHAR(64),
  error_message     TEXT
);

-- User notification preferences
CREATE TABLE user_notification_prefs (
  user_id           VARCHAR(64)   PRIMARY KEY,
  email_opt_in      BOOLEAN       DEFAULT TRUE,
  sms_opt_in        BOOLEAN       DEFAULT TRUE,
  push_opt_in       BOOLEAN       DEFAULT TRUE,
  dnd_start_hour    INTEGER,                          -- UTC hour 0-23
  dnd_end_hour      INTEGER,
  marketing_opt_in  BOOLEAN       DEFAULT TRUE,
  updated_at        TIMESTAMP     DEFAULT NOW()
);
```

```
Redis keys:
  idem:{idempotency_key}           → notification_id (TTL: 24h; SET NX for dedup)
  freq_cap:{user_id}:{category}:{date} → count (TTL: 48h; INCR for frequency check)
  user_prefs:{user_id}             → JSON preferences (TTL: 5min; cache from DB)
  push_token:{user_id}:{platform}  → FCM/APNS token (TTL: 30 days; updated on app open)
```

---

## API Design

```
POST /api/v1/notifications
Body: {
  "idempotency_key": "payment-pay-abc-confirmation",   // caller-generated; unique per event
  "type": "PAYMENT_CONFIRMATION",
  "priority": "TRANSACTIONAL",
  "recipient_user_id": "u-12345",
  "channels": ["push", "sms"],       // preferred channels; fallback if first fails
  "template_id": "payment-confirmed",
  "template_vars": {
    "amount": "₹5,000",
    "merchant": "Amazon",
    "txn_id": "pay-abc"
  },
  "scheduled_at": null               // null = send immediately
}
Response 202: {
  "notification_id": "notif-uuid",
  "status": "ACCEPTED",
  "deduplicated": false              // true if idempotency_key was already seen
}

GET /api/v1/notifications/{notification_id}/status
Response 200: {
  "notification_id": "notif-uuid",
  "status": "SENT",
  "channel_results": [
    {"channel": "push", "status": "DELIVERED", "delivered_at": "..."},
    {"channel": "sms",  "status": "SKIPPED", "reason": "push_delivered"}
  ]
}

POST /api/v1/notifications/batch
Body: { "notifications": [...up to 1000...] }   // for marketing blasts
Response 202: { "accepted_count": 1000, "batch_id": "batch-xyz" }

POST /api/v1/users/{user_id}/preferences
Body: { "sms_opt_in": false, "dnd_start_hour": 22, "dnd_end_hour": 8 }
Response 200
```

---

## High-Level Architecture

```
Producing Service (payment-service, fraud-service, marketing-service)
  │
  └── POST /api/v1/notifications (HTTP; returns 202 immediately)
        │
        ▼
Notification Ingestion API
  ├── Check idempotency: SET NX idem:{idempotency_key} NX EX 86400
  │   → Already exists: return 202 { deduplicated: true } — do nothing
  │   → New key: proceed
  ├── Write notification to PostgreSQL (status=PENDING)
  ├── Push to Kafka topic based on priority:
  │     TRANSACTIONAL: topic "notif-transactional" (consumers: 32 partitions)
  │     MARKETING:     topic "notif-marketing"     (consumers: 8 partitions)
  └── Return 202 Accepted

Kafka Topics:
  notif-transactional  (32 partitions; consumers target < 2s lag)
  notif-marketing      (8 partitions; consumers target < 60s lag)
  notif-dlq            (dead letter; consumed for manual review + alerts)

Notification Dispatcher (per topic):
  ├── Read notification from Kafka
  ├── Load user preferences: GET user_prefs:{user_id} (Redis → DB on miss)
  ├── Check DND window: if transactional → skip DND; if marketing → check DND
  ├── Check frequency cap: INCR freq_cap:{user_id}:{category}:{date}
  │   → If over cap: drop marketing; never drop transactional
  └── Fan-out to channel workers (parallel):
        Each channel worker is independent; independent retry

Channel Workers (separate Kafka topics per channel):
  push-worker:   FCM (Android) or APNS (iOS)
  sms-worker:    Twilio / Sinch
  email-worker:  SendGrid / SES
  inapp-worker:  write to in_app_notifications table

  Per worker retry logic:
    Attempt 1: immediate
    Attempt 2: after 30s
    Attempt 3: after 2min
    Attempt 4: after 10min
    Attempt 5: after 1h
    After 5 failures: → DLQ; update delivery_attempts status=FAILED; alert

Channel Fallback:
  If push → FAILED (token expired, device unreachable):
    → Try SMS automatically (if user has SMS opt-in)
  If SMS → FAILED:
    → Try email
  If all channels fail → DLQ
```

---

## Detailed Components

### Idempotency (Exactly-Once Delivery)
```java
public NotificationResponse submit(NotificationRequest request) {
    String idempKey = request.getIdempotencyKey();

    // SET NX: only set if key doesn't exist
    Boolean isNew = redis.setIfAbsent("idem:" + idempKey,
                                       request.getNotificationId(),
                                       Duration.ofHours(24));
    if (!isNew) {
        // Already seen: return existing notification_id
        String existingId = redis.get("idem:" + idempKey);
        return NotificationResponse.deduplicated(existingId);
    }

    // New request: persist and enqueue
    notificationRepo.save(request);
    kafkaProducer.send(request.getPriority().topic(), request);
    return NotificationResponse.accepted(request.getNotificationId());
}

// Caller responsibility: idempotency_key must uniquely identify the notification intent
// Good:  "payment-pay-abc-confirmation" (payment ID + notification type)
// Bad:   UUID.randomUUID() (new UUID on every retry = dedup doesn't work)
// Bad:   user_id only (all notifications to same user deduplicated)
```

### Frequency Cap Check
```lua
-- Redis Lua: atomic check + increment
-- KEYS[1]: freq_cap:{user_id}:{category}:{date}
-- ARGV[1]: max_count (e.g. 3 for marketing)
-- Returns: {allowed: 0|1, current_count: N}

local key = KEYS[1]
local max = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or 0)

if current >= max then
  return {0, current}
end

redis.call('INCR', key)
redis.call('EXPIRE', key, 172800)  -- 48h TTL (covers today + tomorrow)
return {1, current + 1}
```

### Channel Fallback Chain
```java
public DeliveryResult deliver(Notification notif, List<String> channels) {
    for (String channel : channels) {
        // Check opt-in for this channel
        if (!userPrefs.isOptedIn(notif.getUserId(), channel)) {
            continue; // skip to next channel
        }

        DeliveryResult result = channelWorker(channel).send(notif);

        if (result.isSuccess()) {
            updateDeliveryAttempt(notif, channel, "DELIVERED");
            // Skip remaining channels (push delivered → don't also send SMS)
            return result;
        }

        if (result.isPermanentFailure()) {
            // e.g. unsubscribed, invalid token → don't retry this channel
            updateDeliveryAttempt(notif, channel, result.getStatus());
            // Try next channel
        } else {
            // Transient failure: schedule retry (via Kafka delay topic)
            scheduleRetry(notif, channel, result);
            return DeliveryResult.RETRY_SCHEDULED;
        }
    }
    // All channels failed/skipped
    moveToDLQ(notif);
    return DeliveryResult.ALL_CHANNELS_FAILED;
}
```

### DND (Do Not Disturb) Window
```java
public boolean isInDndWindow(String userId, NotificationPriority priority) {
    // Transactional (OTP, fraud alert) always bypass DND
    if (priority == TRANSACTIONAL) return false;

    UserPrefs prefs = getPrefs(userId);
    if (prefs.getDndStartHour() == null) return false;

    int currentHourUtc = ZonedDateTime.now(ZoneOffset.UTC).getHour();
    int start = prefs.getDndStartHour();
    int end   = prefs.getDndEndHour();

    if (start <= end) {
        // e.g. DND 22:00–08:00: wraps midnight
        return currentHourUtc >= start || currentHourUtc < end;
    } else {
        // e.g. DND 08:00–20:00: same day
        return currentHourUtc >= start && currentHourUtc < end;
    }
}
// Marketing notifications during DND: queued until DND window ends (scheduled_at update)
// NOT dropped — user still gets the notification, just later
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K notifications/sec):
  Single Kafka topic; single worker per channel
  PostgreSQL for all storage
  No deduplication (low volume; manual retry is acceptable)

Phase 2 (1K → 10K/sec):
  Priority Kafka topics (transactional vs marketing)
  Redis idempotency keys
  Redis frequency cap
  Retry with exponential backoff

Phase 3 (10K → 100K/sec):
  Horizontal scaling of channel workers
  Push: FCM batch API (1,000 tokens per call; not 1 call per notification)
  Email: batch via SendGrid bulk API
  SMS: still individual (regulatory requirement for delivery receipt per message)

Phase 4 (marketing blasts: 1M+):
  Segmented blast: read user segment from DB → batch Kafka produce
  Rate-limit outbound channel calls to avoid provider rate limits
    (SendGrid: 100 emails/sec per API key; use multiple keys)
  Pre-warm: for scheduled blasts, pre-enqueue to Kafka 10 min before send time
```

---

## Reliability Strategy
- **Kafka RF=3**: no notification lost on single broker failure
- **At-least-once delivery**: commit Kafka offset after successful DB write + channel delivery
- **Idempotency**: exactly-once to user despite at-least-once from Kafka
- **DLQ monitoring**: alert if DLQ message count > 100; indicates channel provider issue
- **Provider failover**: dual SMS provider (Twilio primary + Sinch secondary); switch on error rate > 5%

---

## Security Considerations
- **OTP notifications**: 6-digit OTP never stored in notification DB (only template variable); OTP validity managed by auth service
- **Phone/email PII**: store hashed in notification log; decrypt only for delivery
- **Provider webhooks**: validate HMAC signature on delivery status callbacks (prevent spoofed delivery confirmations)
- **Unsubscribe**: one-click unsubscribe required (CAN-SPAM, GDPR); honored within 10 days (CAN-SPAM) or immediately
- **Rate limiting on inbound API**: prevent internal service from accidentally flooding notifications

---

## Observability
```
Metrics:
  notifications_ingested_total          (by type, priority)
  notifications_deduplicated_total      (idempotency working)
  delivery_latency_p99 by channel       (push: target < 5s; SMS: < 10s; email: < 30s)
  channel_success_rate by channel       (push: ~85%; SMS: ~95%; email: ~98%)
  frequency_cap_rejections_total        (marketing volume check)
  dlq_message_count                     (alert if > 100; channel provider issue)
  retry_count_distribution              (how often retries are needed)

Business metrics:
  notification_delivery_rate            (% successfully delivered)
  push_token_staleness_rate             (% of push tokens returning "invalid token")
  sms_cost_per_day                      (cost monitoring)

Alerts:
  dlq_message_count > 100              → Channel provider issue; alert on-call
  push_success_rate < 70%             → FCM/APNS connectivity issue
  sms_success_rate < 85%              → SMS provider issue; failover to secondary
  transactional_delivery_latency > 30s → Transactional queue backed up; scale workers
  dedup_rate > 5%                      → Upstream service retrying too aggressively
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| SMS provider (Twilio) down | SMS delivery fails | Automatic failover to Sinch; DLQ for manual retry |
| Push token expired/invalid | Push fails silently | FCM/APNS return error → mark token invalid → fallback to SMS |
| Kafka consumer lag spike | Transactional notifications delayed | Scale workers; priority topics prevent marketing from blocking transactional |
| Redis (idempotency) down | Risk of duplicate sends during window | Fail-open: accept without dedup; alert; DB uniqueness constraint as backup |
| Email provider rate limit | Marketing emails delayed | Queue backpressure; rate-limit outbound; multiple API keys |
| DLQ grows unbounded | Permanent failures piling up | Alert at 100; manual review; provider issue investigation |

---

## Chaos Testing
```
Experiment 1: SMS Provider Down (Twilio)
  Inject:    Return 503 from all Twilio API calls
  Expected:  SMS worker retries (exponential backoff: 30s, 2min, 10min)
  Expected:  After retry limit: automatic failover to Sinch secondary
  Expected:  Delivery latency increases; no notification lost
  Red flag:  Notifications go to DLQ without trying Sinch

Experiment 2: Idempotency Duplicate Suppression
  Inject:    Submit same notification twice with same idempotency_key (simulates upstream retry)
  Expected:  First call: 202 Accepted; notification processed; SMS sent
  Expected:  Second call: 202 Accepted, { deduplicated: true }; no second SMS sent
  Red flag:  Two SMSs delivered to user

Experiment 3: Marketing Blast vs Transactional OTP
  Inject:    Queue 1M marketing notifications; simultaneously trigger 100K transactional OTPs
  Expected:  OTPs delivered within 10s despite marketing backlog
  Expected:  Marketing consumers have no effect on transactional consumers (separate topics)
  Red flag:  OTP delivery delayed because marketing is consuming all Kafka consumer threads

Experiment 4: Frequency Cap Enforcement
  Inject:    Submit 10 marketing notifications for same user (cap=3/day)
  Expected:  First 3: delivered; next 7: frequency_cap_rejections metric increments
  Expected:  Transactional notifications for same user: unaffected (cap is per category)
  Red flag:  Transactional OTP blocked by marketing frequency cap
```

---

## Monthly Cost Estimate
```
Scale assumption: 10K transactional/sec peak; 1M marketing/day

Kafka Cluster:
  3 × kafka.m5.large ($150/mo)                  = $450/mo

Channel Workers (8 instances):
  8 × t3.medium ($34/mo)                        = $272/mo

Redis (idempotency + prefs + freq cap):
  cache.r6g.large ($130/mo)                     = $130/mo

PostgreSQL (notifications + delivery attempts):
  db.r6g.large ($200/mo)                        = $200/mo

External Provider Costs:
  SMS: 600K/day × 30 × $0.004 = $72,000/mo     (major cost driver at scale)
  Email: 300K/day × 30 × $0.001 = $9,000/mo
  Push: Free (FCM/APNS)
  Total provider costs:                          = $81,000/mo

Infrastructure only:                             = $1,052/mo
Total (infra + providers):                       = ~$82,000/mo

SMS is 88% of total cost. Cost optimisation:
  Prefer push over SMS wherever possible (push is free)
  Use email for non-urgent marketing (1/4 the SMS cost)
  Aggregate notifications: 3 events → 1 summary notification
  Regional SMS routing (local SIM routing can cut SMS cost by 30–50% in some markets)
```

---

## Migration Story

### Current State
Each service (payment-service, auth-service) directly calls Twilio/SendGrid APIs synchronously in-request path. No deduplication. Payment confirmation SMS being sent twice on retry. No user preferences. No unsubscribe.

### Migration Plan

```
Step 1: Build notification service with passthrough (zero downtime)
  Action:
    Build Notification API; it just calls Twilio/SendGrid (same as before)
    Services switch from direct provider call to Notification API call
    Benefit: centralized logging, retry logic, delivery tracking
  Rollback: services revert to direct provider calls

Step 2: Add idempotency (zero downtime)
  Action:
    Add idempotency_key field to API
    Callers add idempotency_key to every notification request
    Redis dedup: duplicate sends suppressed
  Monitor: dedup_rate (should be low for normal operations)
  Rollback: disable dedup check; all requests pass through

Step 3: Make async via Kafka (zero downtime, dual-path)
  Action:
    Ingestion API returns 202 immediately; pushes to Kafka
    Kafka consumer calls provider
    Run dual-path (sync + async) for 1 week; compare delivery rates
    Switch to async-only
  Rollback: disable Kafka path; revert to sync delivery in API

Step 4: Add user preferences and frequency caps (zero downtime)
  Action:
    Deploy preference API and table
    Import existing opt-outs from provider blocklists
    Add DND and frequency cap checks in dispatcher
  Rollback: disable preference checks; all notifications pass through

Step 5: Add channel fallback and priority topics (zero downtime)
  Action:
    Split Kafka topics: transactional vs marketing
    Add fallback logic in channel workers
    Monitor: delivery rates per channel
  Rollback: merge back to single Kafka topic; disable fallback
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL: notification definitions and delivery attempts (authoritative).
  → Redis: idempotency keys (ephemeral; TTL 24h), frequency caps, user pref cache.
  → Kafka: in-flight notifications (temporary buffer; not authoritative after delivery).

□ Consistency model?
  → At-least-once delivery from Kafka → idempotency key ensures exactly-once to user.
  → User preferences: eventual (5-min Redis cache); changes take up to 5 min to propagate.
  → Frequency cap: strong (Redis Lua atomic); accurate within Redis session.

□ Failure mode?
  → SMS provider down → fallback to secondary → DLQ if all fail.
  → Redis (idempotency) down → fail-open (accept without dedup); risk brief duplicates.
  → Transactional: never dropped; retried up to 5× before DLQ with alert.
  → Marketing: dropped on frequency cap or DND window (intentional, not a failure).

□ Hot partitions?
  → Marketing blast: all notifications queued simultaneously → Kafka partition backlog.
  → Mitigation: separate topics for marketing; scale marketing workers independently.
  → Transactional is unaffected: dedicated topic with dedicated consumers.

□ Cost driver?
  → SMS provider: 88% of total cost. Push is free; use push first.
  → Optimise: prefer push; batch email; aggregate marketing into digest.

□ Security risk?
  → OTP in notification log: never store OTP value; only template reference.
  → Delivery status webhook forgery: validate HMAC on all provider callbacks.
  → Bulk unsubscribe: someone forging unsubscribes for other users.
    Mitigation: unsubscribe link contains signed token with user_id + expiry.
```

---

## Staff Engineer Discussion Points

**"How do you guarantee OTP is not sent twice to the user?"**
Idempotency key. The auth service generates the idempotency key as `otp-{user_id}-{session_id}-{attempt_number}`. If the OTP delivery fails and the auth service retries, it uses the same idempotency key. The notification service detects the duplicate via Redis SET NX and returns `deduplicated: true` without sending again. The OTP is only sent once — the first successful enqueue. This is the same pattern as payment idempotency keys (Case Study 36+).

**"How do you handle a marketing blast to 10M users?"**
Batch Kafka production: the marketing service reads user IDs in chunks of 10K from the DB (streaming cursor) and produces to Kafka at a controlled rate (e.g., 50K messages/sec). Kafka buffers the full 10M messages. The marketing channel workers consume at 1,000 SMS/sec (limited by Twilio rate limits) and 10,000 emails/sec (SendGrid bulk API). Total delivery time: 10M emails / 10,000/sec = ~17 minutes. The key is separating ingestion rate (fast) from delivery rate (provider-limited) via Kafka as the buffer.

**"What happens when a user's push token is stale?"**
FCM/APNS returns a specific error code: `NotRegistered` (FCM) or `BadDeviceToken` (APNS). On receiving this: (1) mark the push token as invalid in Redis, (2) trigger fallback to SMS or email for this specific delivery, (3) update the user's device record in the DB to remove the invalid token. The user will get a fresh token next time they open the app (standard push token refresh). We never retry with a known-invalid token.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Multiple channels with different cost/reliability"  → Channel Fallback Chain
  "Transactional must not be delayed by marketing"    → Priority Kafka Topics
  "OTP sent twice on retry"                           → Idempotency Key (SET NX)
  "User complained about too many notifications"      → Frequency Cap (Redis Lua)

SCC LENS:
  STATE:
    Authoritative: PostgreSQL (notification records + delivery attempts)
    Ephemeral: Redis (idempotency keys, freq caps, user prefs cache)
    Buffer: Kafka (in-flight; not authoritative; Kafka is the pipe, not the store)
  COORDINATION:
    Idempotency: SET NX in Redis — atomic claim for first sender
    Frequency cap: Lua script — atomic INCR + check
    Channel fallback: sequential try → success stops chain; failure escalates
  CONCENTRATION:
    Marketing blast → Kafka backlog → separate topic prevents transactional starvation
    SMS provider rate limits → Kafka absorbs burst; workers consume at provider rate

INTERVIEW TRIGGER: "Design a notification service" →
  1. Priority queues (transactional ≠ marketing)
  2. Idempotency key (SET NX) for exactly-once OTP/payment confirmation
  3. Channel fallback chain (push → SMS → email; cheapest first)
  4. Frequency cap per user per category
  5. DLQ + retry with backoff; DLQ alert for manual review
  6. Cost insight: SMS is 100× more expensive than push → prefer push
```

---

*Next: Case Study 12 — News Feed*
