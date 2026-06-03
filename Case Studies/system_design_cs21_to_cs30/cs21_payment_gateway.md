# Case Study 21 — Payment Gateway

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Idempotency, Two-Phase Commit alternative, PCI-DSS scope reduction.

---

## Business Context

A payment gateway is the most critical system in a fintech platform. It authorises, captures, settles, and reconciles money movement. Every design decision has financial and regulatory consequences. A duplicated charge is a fraud incident. A dropped authorisation is lost revenue. An unreconciled settlement is a regulatory violation.

The core design principles: **money never disappears and never duplicates**. Every rupee that enters must exit exactly once, to the right place, at the right time, with an audit trail.

**Business goals driving architecture:**
- Zero duplicate charges (idempotency is non-negotiable)
- Sub-second payment authorisation (user waits at checkout)
- 99.99% availability (downtime = revenue loss at ₹1M+/sec)
- PCI-DSS compliance (no card data in our systems)
- Reconciliation: every payin matches every payout within T+1

---

## Recognition Framework

### Signals from the Problem
```
- Money movement requires exactly-once semantics (no duplicate charges)
- External dependency (card network, bank) that can timeout → idempotency key required
- Two-phase: authorise now, capture later (reservation model)
- Distributed state: payment spans multiple systems (our DB + bank + card network)
- Reconciliation: must verify settlement matches what we expected
- PCI-DSS: card data must be tokenised; never stored in plaintext
- Audit trail: every state change is a financial event; immutable
- Retry safety: upstream service can retry; must produce same outcome
```

### Two-Phase Commit vs Saga Pattern

```
Problem: payment spans multiple systems
  1. Debit user's bank account (via card network)
  2. Credit merchant's account (in our system)
  3. Record transaction in our ledger

If step 2 succeeds but step 3 fails → money moved but no record → financial loss

Option A: Two-Phase Commit (2PC)
  Phase 1 (Prepare): all participants lock resources and vote COMMIT/ABORT
  Phase 2 (Commit/Abort): coordinator tells all to commit or all to abort
  
  Problem for payments:
    Card network does NOT participate in 2PC (external; no 2PC protocol)
    2PC requires all participants to implement the protocol (impossible with banks)
    2PC coordinator failure = all participants stuck waiting (blocking protocol)
  Verdict: IMPOSSIBLE with external parties; REJECTED

Option B: Saga Pattern (Chosen)
  Each step has a compensating transaction
  If step 3 fails: run compensating action for step 1 and 2 (refund/reversal)
  
  Choreography (event-driven): each step publishes event; next step subscribes
  Orchestration (command-driven): saga orchestrator calls each step
  
  For payments: ORCHESTRATION (chosen)
    Reason: payment flow must be deterministic; orchestrator has full visibility
    Compensating transactions: authorisation void, refund, ledger reversal
  Verdict: SAGA ORCHESTRATION is the correct pattern for cross-system money movement
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Idempotency Key | Exactly-once charge despite retries; every payment API call carries idempotency key |
| Saga Orchestration | Cross-system money movement; compensating transactions on failure |
| Payment State Machine | Authorised → Captured → Settled → Reconciled; immutable transitions |
| Tokenisation | PCI-DSS: card data never in our system; token replaces card number |
| Ledger (Double-Entry) | Every credit has corresponding debit; balance always zero-sum |
| Reconciliation Job | Daily batch: verify settlements match expected amounts |

---

## Problem Statement

Payment gateway for authorisation, capture, settlement, and reconciliation. Integrates with card networks (Visa, Mastercard) and bank partners. PCI-DSS compliant. Exactly-once charge guarantee.

---

## Functional Requirements
- Payment authorisation: charge customer's card (reserve funds)
- Payment capture: confirm charge (move funds)
- Payment refund: full or partial refund
- Payment void: cancel authorised but uncaptured payment
- Webhook: notify merchant of payment status changes
- Reconciliation: verify daily settlement with bank
- Dashboard: merchant views payment history, exports statements

## Non-Functional Requirements
- Authorisation latency: **< 2 seconds p99** (card network adds ~500ms)
- Availability: **99.99%** (< 53 minutes downtime/year)
- Throughput: **50,000 transactions/sec** peak (India payment festival)
- Idempotency: **zero duplicate charges**
- Data: **no plaintext card data** (PCI-DSS scope reduction)

---

## Capacity Estimation
```
Peak throughput: 50,000 TPS
  Each transaction: ~2 KB state (authorisation request, response, metadata)
  Daily: 50K × 86400 × 10% active hours = ~432M transactions/day
  Storage: 432M × 2 KB = ~864 GB/day (hot tier)

Idempotency keys:
  50K/sec × 86400 = 4.32B keys/day
  Redis: TTL 24h; at 100 bytes/key = 432 GB → too large
  Optimization: TTL 2h for most payments (retry window); 24h for dispute window
  At 2h TTL: 50K × 7200 = 360M keys × 100 bytes = 36 GB → fits in Redis Cluster

Ledger entries:
  2 entries per payment (debit + credit) = 864M entries/day
  PostgreSQL (append-only): 864M × 100 bytes = ~86 GB/day
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Idempotency Key | Exactly-once; caller retries safely; idempotency key = same result |
| Outbox Pattern | Reliable webhook delivery; atomically write payment + webhook event |
| Saga Orchestration | Multi-step money movement; compensating transactions on failure |
| Payment State Machine | Immutable transitions; no money lost in state transitions |
| Double-Entry Ledger | Every debit has credit; balance always reconciles to zero |
| Tokenisation (Vault) | Store card token; never raw PAN; PCI-DSS scope reduction |

---

## Data Model

```sql
-- Payments (core payment record)
CREATE TABLE payments (
  payment_id       UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key  VARCHAR(128)  UNIQUE NOT NULL,
  merchant_id      VARCHAR(64)   NOT NULL,
  customer_id      VARCHAR(64),
  amount           BIGINT        NOT NULL,    -- in smallest unit (paise for INR)
  currency         CHAR(3)       NOT NULL,    -- ISO 4217: INR, USD
  status           VARCHAR(32)   NOT NULL DEFAULT 'CREATED',
  -- CREATED → AUTHORISED → CAPTURED → SETTLED → RECONCILED
  -- CREATED → AUTHORISED → VOIDED
  -- CAPTURED → REFUNDED (partial or full)
  payment_method   VARCHAR(32)   NOT NULL,    -- CARD / UPI / NETBANKING / WALLET
  card_token       VARCHAR(128),             -- vault token (never PAN)
  network_txn_id   VARCHAR(128),             -- card network's transaction ID
  authorised_at    TIMESTAMP,
  captured_at      TIMESTAMP,
  settled_at       TIMESTAMP,
  failure_code     VARCHAR(64),
  failure_message  TEXT,
  metadata         JSONB,                    -- merchant's custom fields
  created_at       TIMESTAMP     DEFAULT NOW(),
  updated_at       TIMESTAMP     DEFAULT NOW()
);

-- Payment events (immutable audit trail — every state change)
CREATE TABLE payment_events (
  event_id         BIGSERIAL     PRIMARY KEY,
  payment_id       UUID          NOT NULL REFERENCES payments(payment_id),
  event_type       VARCHAR(64)   NOT NULL,   -- AUTHORISATION_REQUEST, AUTHORISATION_SUCCESS, etc.
  old_status       VARCHAR(32),
  new_status       VARCHAR(32),
  amount           BIGINT,
  actor            VARCHAR(64),              -- which service/user triggered
  network_response JSONB,                    -- raw response from card network (masked)
  created_at       TIMESTAMP     DEFAULT NOW()
);
-- Append-only: never UPDATE or DELETE; full history of every payment

-- Double-entry ledger
CREATE TABLE ledger_entries (
  entry_id         BIGSERIAL     PRIMARY KEY,
  payment_id       UUID,
  entry_type       VARCHAR(32)   NOT NULL,   -- DEBIT / CREDIT
  account_id       VARCHAR(64)   NOT NULL,   -- merchant / customer / platform / card_network
  amount           BIGINT        NOT NULL,   -- always positive
  currency         CHAR(3)       NOT NULL,
  description      VARCHAR(256),
  created_at       TIMESTAMP     DEFAULT NOW()
);
-- Invariant: SUM(credits) = SUM(debits) for every payment_id

-- Outbox (for reliable webhook delivery)
CREATE TABLE outbox_events (
  outbox_id        BIGSERIAL     PRIMARY KEY,
  payment_id       UUID          NOT NULL,
  event_type       VARCHAR(64)   NOT NULL,
  payload          JSONB         NOT NULL,
  destination_url  TEXT,                    -- merchant webhook URL
  status           VARCHAR(16)   DEFAULT 'PENDING',  -- PENDING / SENT / FAILED
  attempt_num      INTEGER       DEFAULT 0,
  next_retry_at    TIMESTAMP     DEFAULT NOW(),
  created_at       TIMESTAMP     DEFAULT NOW()
);
```

---

## API Design

```
POST /api/v1/payments
Headers: Idempotency-Key: {uuid-from-merchant}
Body: {
  "amount": 50000,                    // 500.00 INR in paise
  "currency": "INR",
  "payment_method": "CARD",
  "card_token": "tok_abc123",         // vault token; never raw card number
  "merchant_id": "merch-xyz",
  "customer_id": "cust-456",
  "capture_mode": "AUTO",             // AUTO = authorise+capture; MANUAL = authorise only
  "metadata": { "order_id": "ord-789" }
}
Response 201: {
  "payment_id": "pay-uuid",
  "status": "CAPTURED",               // AUTO capture mode: authorised + captured immediately
  "amount": 50000,
  "currency": "INR",
  "network_txn_id": "VISA-123456",
  "authorised_at": "2024-01-01T14:00:00Z",
  "captured_at": "2024-01-01T14:00:00Z"
}
Response 200: (if idempotency key already seen → same response as original)
Response 402: { "error": "payment_declined", "decline_code": "INSUFFICIENT_FUNDS" }
Response 409: { "error": "duplicate_payment" }  (if idempotency key collision with different amount)

POST /api/v1/payments/{payment_id}/capture
Body: { "amount": 50000 }  // can be ≤ authorised amount (partial capture)
Response 200: { "payment_id": "...", "status": "CAPTURED", "captured_amount": 50000 }

POST /api/v1/payments/{payment_id}/refund
Body: { "amount": 25000, "reason": "customer_request" }
Response 200: { "refund_id": "ref-uuid", "status": "PENDING" }

POST /api/v1/payments/{payment_id}/void
Response 200: { "payment_id": "...", "status": "VOIDED" }

GET /api/v1/payments/{payment_id}
Response 200: full payment object + event history

Webhook (sent to merchant):
  POST {merchant_webhook_url}
  Headers: X-Signature: HMAC-SHA256({payload}, {merchant_webhook_secret})
  Body: { "event": "payment.captured", "payment_id": "...", "amount": 50000 }
```

---

## High-Level Architecture

```
Merchant / Client
  │
  ▼
API Gateway (Case 05) → Payment API
  │
  ├── Step 1: Idempotency check
  │     GET idempotency:{idempotency_key} from Redis
  │     HIT:  return cached response immediately (200 OK)
  │     MISS: proceed; SET idempotency key with lock (SET NX)
  │
  ├── Step 2: Payment creation
  │     INSERT payment (status=CREATED) + INSERT payment_event
  │     INSERT outbox_event (status=PENDING) — SAME transaction
  │     (Outbox pattern: atomic write ensures no payment without event)
  │
  ├── Step 3: Card Network call (Saga Step 1)
  │     Vault: exchange card_token → temporary card data (never stored in our DB)
  │     POST to Visa/Mastercard/Rupay API with card data
  │     Card data is TRANSIENT: exists in memory for this call only
  │     Response: AUTHORISED (with network_txn_id) or DECLINED
  │
  ├── Step 4: Update payment state
  │     UPDATE payments SET status=AUTHORISED, network_txn_id=...
  │     INSERT payment_event (AUTHORISATION_SUCCESS)
  │     INSERT ledger_entries (customer DEBIT, merchant CREDIT pending)
  │
  ├── Step 5: Capture (if capture_mode=AUTO)
  │     POST capture to card network
  │     UPDATE payments SET status=CAPTURED
  │     INSERT payment_event (CAPTURE_SUCCESS)
  │     UPDATE ledger_entries (CREDIT confirmed)
  │
  └── Step 6: Set idempotency response + trigger webhooks
        SET idempotency:{key} {response_json} EX 86400
        UPDATE outbox_events SET status=READY (outbox worker picks up for delivery)
        Return 201 to merchant

Card Network Integration (isolated behind CardNetworkAdapter):
  Timeout: 5 seconds (card network SLA)
  Retry: 2× on timeout (with same idempotency key to network)
  Failure handling:
    Network timeout: update payment status=AUTHORISATION_TIMEOUT
    Decline: update payment status=DECLINED with decline_code
    Unknown/ambiguous: mark status=AUTHORISATION_UNKNOWN → manual review queue

Outbox Worker (async; separate process):
  Poll outbox_events WHERE status='PENDING' AND next_retry_at <= now
  HTTPS POST to merchant webhook URL with HMAC-signed payload
  On success: UPDATE status=SENT
  On failure: exponential backoff; max 24h retry window
  After 24h: status=PERMANENTLY_FAILED; alert merchant; manual intervention
```

---

## Detailed Components

### Payment State Machine
```
                    ┌─────────────────────────────────────────────────────┐
                    │                                                     │
CREATED ──────► AUTHORISED ──────► CAPTURED ──────► SETTLED ──────► RECONCILED
    │                │                  │
    │          (card network            │ ──► PARTIALLY_REFUNDED
    │           declined)               │
    ▼                ▼                  ▼
 FAILED           VOIDED            REFUNDED
    
State transition rules (enforced in application layer + DB constraint):
  CREATED → AUTHORISED:    only on successful card network response
  CREATED → FAILED:        on card network decline
  AUTHORISED → CAPTURED:   only if AUTHORISED (cannot capture FAILED payment)
  AUTHORISED → VOIDED:     cancel before capture
  CAPTURED → SETTLED:      batch settlement job (bank batch)
  SETTLED → RECONCILED:    reconciliation job confirms settlement
  CAPTURED → REFUNDED:     full refund
  CAPTURED → PARTIALLY_REFUNDED: partial refund

Immutability principle: every transition is an INSERT to payment_events (never UPDATE status directly without event).
```

### Outbox Pattern — Reliable Webhook Delivery
```java
// In PaymentService — ATOMIC write: payment + event + outbox in ONE transaction
@Transactional
public Payment createAndAuthorise(PaymentRequest req) {
    // 1. Create payment record
    Payment payment = paymentRepo.save(new Payment(req));

    // 2. Insert outbox event (SAME TRANSACTION as payment)
    outboxRepo.save(OutboxEvent.builder()
        .paymentId(payment.getId())
        .eventType("payment.created")
        .payload(toJson(payment))
        .destinationUrl(merchantWebhookUrl)
        .build());

    // 3. Call card network (outside transaction — external call)
    CardNetworkResponse netResp = cardNetworkAdapter.authorise(payment);

    // 4. Update payment state + another outbox event (same transaction)
    payment.setStatus(netResp.isSuccess() ? AUTHORISED : FAILED);
    paymentRepo.save(payment);
    outboxRepo.save(OutboxEvent.of("payment." + payment.getStatus().lower(), payment));

    return payment;
}
// If the DB transaction commits: both payment AND outbox event are atomically saved
// If the machine crashes after DB commit: outbox worker delivers webhook on restart
// If the machine crashes before DB commit: neither saved → safe to retry
// This eliminates "payment saved but webhook never sent" scenario
```

### Double-Entry Ledger
```
Payment: customer pays merchant ₹500

Ledger entries created atomically with payment capture:

  DEBIT:  customer_receivable_account   ₹500 (customer owes us)
  CREDIT: merchant_payable_account      ₹450 (we owe merchant after fees)
  CREDIT: platform_fee_account          ₹50  (our revenue)

Invariant check (runs after every batch):
  SELECT SUM(CASE WHEN entry_type='DEBIT' THEN amount ELSE -amount END)
  FROM ledger_entries WHERE payment_id = ?
  → Must equal 0 (every debit has equal credits)

Settlement (T+1 batch):
  DEBIT:  merchant_payable_account      ₹450 (paying out to merchant)
  CREDIT: nostro_account                ₹450 (our bank account decreases)
  
  After settlement: merchant's balance = 0 (paid out)
  Platform fee: accumulates in platform_fee_account (revenue recognition)
```

### PCI-DSS Scope Reduction via Tokenisation
```
PCI-DSS scope: any system that touches raw card data (PAN, CVV, expiry)
  → Requires quarterly audits, penetration testing, strict access controls

Scope reduction strategy:
  Raw card data path (IN SCOPE, heavily controlled):
    Browser/App → [HTTPS] → Card Vault Service → Card Network
    Card Vault: dedicated service; HSM-backed; separate network segment
    Raw card data NEVER reaches payment-service, DB, or logs
  
  Token path (OUT OF SCOPE, normal security):
    Browser/App → gets card_token from vault
    payment-service: receives card_token (not raw card data)
    DB: stores card_token only
    card_token is useless to attacker (cannot use to make charges without vault key)

  Tokenisation process:
    1. Client sends card data to vault (vault is in PCI scope)
    2. Vault stores encrypted card data in HSM-backed storage
    3. Vault returns opaque token: tok_abc123
    4. Client uses tok_abc123 for all subsequent payments
    5. At payment time: payment-service sends tok_abc123 to vault
    6. Vault exchanges token for temporary single-use credential
    7. Vault (not payment-service) calls card network with credential
  
  Result: payment-service, DB, logs, all other services → OUT OF PCI scope
  PCI-DSS audit scope reduced from "entire platform" to "vault service only"
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K TPS):
  Single payment-service instance
  PostgreSQL for all state
  Synchronous webhook delivery
  Synchronous card network call (blocking)

Phase 2 (1K → 10K TPS):
  Multiple payment-service instances (stateless; horizontal scale)
  Redis for idempotency keys
  Outbox worker for async webhook delivery
  Card network calls with timeout + retry

Phase 3 (10K → 50K TPS):
  Read replicas for payment history queries
  Separate ledger service (high-write; separate DB)
  Kafka for payment events (audit log, analytics, fraud signals)
  Priority queue: retry failed payments without blocking new ones

Phase 4 (50K+ TPS, festival scale):
  Payment DB sharding by merchant_id (tenant isolation)
  Pre-authorisation warming (pre-authorise tokens before peak)
  Circuit breaker per card network (Visa, Mastercard, Rupay separate)
  Regional deployment (Mumbai, Delhi, Singapore)
```

---

## Reliability Strategy
- **Idempotency**: SET NX before external call; prevents double charge on retry
- **Outbox pattern**: atomic payment + webhook event; no missed webhooks
- **Card network circuit breaker**: open after 10% failure rate; fall back to alternate network
- **Payment timeout handling**: AUTHORISATION_UNKNOWN status → manual review; never assume success or failure on ambiguous timeout
- **DB HA**: PostgreSQL Patroni (primary + standby); < 30s failover; payment briefly pauses
- **Graceful degradation**: if card network unavailable → queue payment request → retry when available (for non-real-time flows like batch payouts)

---

## Security Considerations
- **PCI-DSS DSS 4.0**: card vault is sole PCI scope; separate network, HSM, strict access
- **Idempotency key collision**: if same key with different amount → return 409 (not silently accept)
- **SSRF on webhooks**: validate merchant webhook URLs against allowlist; no internal URLs
- **Webhook HMAC**: every webhook signed with HMAC-SHA256 using merchant's secret key
- **SQL injection on payment metadata**: use parameterised queries; never concatenate JSONB
- **Audit log**: every payment state change → immutable payment_events + Case 10 audit service
- **Rate limiting**: per-merchant: 1,000 payment attempts/min (prevents brute-force)
- **CVV**: never stored; used once for authorisation; not even in vault after auth

---

## Observability
```
Golden signals per payment type (card, UPI, netbanking):
  payment_authorisation_rate         (authorised / attempted; target > 90%)
  payment_success_rate               (captured / attempted; target > 85%)
  payment_latency_p99                (end-to-end; target < 2s)
  decline_rate_by_reason             (insufficient_funds, fraud, network)
  card_network_latency_p99           (external dependency; target < 500ms)
  refund_rate                        (% of payments refunded; alert on spike)

Financial health metrics:
  ledger_balance_check               (daily: SUM(debits) = SUM(credits) for every payment)
  settlement_reconciliation_gap      (expected vs actual settlement amount; alert if > 0)
  unreconciled_payments_count        (payments not yet reconciled after T+2)

Operational alerts:
  authorisation_timeout > 5% in 5min → card network issue; circuit breaker
  decline_rate spike > 20% vs baseline → fraud wave or bank connectivity issue
  outbox_webhook_failure_rate > 10%  → merchant webhook issue; check destination URLs
  ledger_imbalance detected          → CRITICAL: financial integrity issue
```

---

## Failure Scenarios
| Failure | Impact | Mitigation |
|---|---|---|
| Card network timeout | Payment authorisation hangs | 5s timeout → AUTHORISATION_UNKNOWN → manual review queue |
| DB commit fails after card auth | Card charged; no record in our DB | Outbox ensures event saved atomically; idempotency prevents double-auth |
| Duplicate payment attempt | Risk of double charge | SET NX idempotency key; second attempt returns cached first response |
| Webhook delivery failure | Merchant not notified of payment | Outbox pattern + 24h retry window; alert after max retries |
| Card network decline storm | High payment failure rate | Circuit breaker; fallback network; decline reason analysis |
| Reconciliation mismatch | Bank settled different amount than expected | Alert immediately; pause settlement for affected merchant; manual review |
| PAN exposed in logs | PCI violation | Log scrubber strips card patterns; vault never logs PAN |

---

## Chaos Testing
```
Experiment 1: Card Network Timeout During Authorisation
  Inject:    Card network API returns 504 after 6 seconds
  Expected:  Payment-service times out at 5s (before client timeout)
  Expected:  Payment status = AUTHORISATION_TIMEOUT (not FAILED; not AUTHORISED)
  Expected:  Client receives 504 with "gateway_timeout" error code
  Expected:  Retry with same idempotency key: new attempt (not cached timeout)
  Red flag:  Payment shows AUTHORISED despite no network confirmation

Experiment 2: Double Submit (Merchant Retries on Timeout)
  Inject:    Send same payment twice with same Idempotency-Key
  Expected:  First call: acquires Redis lock; calls network; stores result
  Expected:  Second call (concurrent): Redis SET NX fails → wait for first to complete
  Expected:  Second call (sequential): Redis hit → return first call's cached response
  Expected:  Only ONE charge appears on customer's card
  Red flag:  Customer charged twice (idempotency not working)

Experiment 3: Payment DB Failure After Card Authorisation
  Inject:    Kill PostgreSQL connection after card network authorisation succeeds
             but before DB write completes
  Expected:  Transaction rolled back; payment not saved
  Expected:  Idempotency key NOT set (Redis SET was after DB commit)
  Expected:  Card authorisation is now "dangling" (authorised at network but no record in our DB)
  Expected:  Reconciliation job detects: network shows authorised; our DB shows nothing
  Expected:  Manual review: void the dangling authorisation
  Red flag:  Duplicate charge on retry (idempotency key was set before DB commit)

Experiment 4: Reconciliation Mismatch
  Inject:    Manually alter one ledger entry amount (simulate data corruption)
  Expected:  Daily reconciliation job: SUM(our_ledger) ≠ SUM(bank_statement)
  Expected:  CRITICAL alert fires immediately
  Expected:  Mismatch isolated to specific merchant and date
  Red flag:  Reconciliation job passes despite data corruption (query bug)
```

---

## Monthly Cost Estimate
```
Scale: 50K TPS peak, 432M transactions/day

Payment API + Saga Orchestrator:
  20 × c5.4xlarge ($560/mo)                         = $11,200/mo

PostgreSQL (payments + ledger + outbox):
  Primary: db.r6g.4xlarge ($1,600/mo)
  2 read replicas: $1,600/mo each
  Total PostgreSQL:                                  = $4,800/mo

Redis (idempotency keys):
  3 × cache.r6g.2xlarge ($500/mo) [36 GB cluster]   = $1,500/mo

Card Vault (PCI scope; HSM-backed):
  2 × dedicated instances + HSM ($5,000/mo HSM lease) = $7,000/mo

Kafka (payment events for audit + analytics):
  6 × kafka.m5.2xlarge ($300/mo)                    = $1,800/mo

Outbox Worker:
  4 × t3.large ($60/mo)                             = $240/mo

Monitoring + security tooling:
                                                     = $3,000/mo
─────────────────────────────────────────────────────────────────
Total infrastructure: ~$29,540/month

Note: Card network fees (1.5–3% of transaction volume) dwarf infrastructure costs.
At 432M × avg ₹500 = ₹216B/day processed → network fees >> infrastructure costs.
PCI compliance audit: ~$50,000–$100,000 annually (additional).
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL payments table: canonical payment state.
  → payment_events: immutable audit trail; never UPDATE or DELETE.
  → ledger_entries: double-entry bookkeeping; financial truth.
  → Redis idempotency: operational dedup cache; DB is authoritative.

□ Consistency model?
  → Payment state: strong (PostgreSQL; every state change in explicit transaction).
  → Webhook delivery: at-least-once (outbox worker retries).
  → Ledger balance: strongly consistent (written in same transaction as payment capture).

□ Failure mode?
  → Card network timeout: AUTHORISATION_UNKNOWN → manual review (never assume success).
  → DB write fails after network auth: dangling auth detected by reconciliation.
  → Redis loss: idempotency cache cold-starts; brief window of possible duplicate check-failure.
    Mitigation: DB UNIQUE constraint on idempotency_key as backup (slower but correct).

□ PCI-DSS:
  → Raw card data: vault only (network-isolated; HSM-backed).
  → Payment service, DB, logs, webhooks: card tokens only (OUT of PCI scope).
  → Vault compromise: immediately rotate HSM keys; re-tokenise all stored tokens.

□ Financial correctness:
  → Every payment: exactly 2+ ledger entries (debit + credit(s)).
  → Invariant: SUM(debits) = SUM(credits) for every payment_id.
  → Verified by: reconciliation job + real-time balance check on every write.

□ SCC LENS:
  STATE: PostgreSQL (payments/ledger/events), Redis (idempotency cache)
  COORDINATION: idempotency (SET NX), outbox (atomic event), saga orchestration
  CONCENTRATION: card network (external SPOF) → circuit breaker; multiple network fallback
```

---

## Staff Engineer Discussion Points

**"How do you prevent the exact scenario: card charged, our DB write fails?"**
This is the "dangling authorisation" problem. Two defences: (1) Idempotency key set AFTER DB commit — if DB fails, key is not set; retry makes fresh attempt to card network. The first attempt's authorisation at the card network expires (typically 7 days) without capture — customer never charged. (2) Reconciliation job: daily comparison of card network's authorisation report vs our DB. Any network-authorised payment with no DB record → automatic void request to network. The combination prevents the customer being charged for something that has no record in our system.

**"What's the difference between authorisation and capture in card payments?"**
Authorisation: card network puts a hold on the customer's funds (reserves ₹500). Money has not moved yet. Capture: confirms the charge; money actually moves from customer's bank to acquiring bank. Two-step flow is important for e-commerce: authorise at checkout (verify card works, reserve funds), capture at shipping (when goods leave warehouse). If order is cancelled: void the authorisation (release the hold; customer never charged). Hotels and car rentals heavily use this: authorise at check-in, capture at check-out with actual amount.

**"How does your payment gateway IAM work — who can access what?"**
Multi-layer: (1) Merchants authenticate with API keys (HMAC-signed); scope: create payments, read own payment history. (2) Internal services use mTLS + JWT (service identity via SPIFFE/SPIRE). (3) Finance team: read-only access to ledger; no payment creation. (4) Admin: payment override (refund, void) requires dual-approval (maker-checker pattern) + MFA. (5) PCI vault: strict access log; every card data access logged with reason code; quarterly access review. This maps directly to your RBAC + ABAC background — payment scopes are fine-grained (payments:create, payments:refund, ledger:read) enforced by OPA.

---

## Cheat Sheet Tie-in

```
PROBLEM SIGNALS → PATTERNS:
  "Cannot duplicate a charge on retry"         → Idempotency Key (SET NX before network call)
  "Payment spans multiple systems"             → Saga Orchestration (not 2PC)
  "Never lose a webhook delivery"             → Outbox Pattern (atomic event + DB write)
  "Card data must not touch our DB"           → Tokenisation (vault + token in DB)
  "Every debit must have a credit"            → Double-Entry Ledger

IDEMPOTENCY KEY PLACEMENT (critical):
  SET idempotency key AFTER DB commit (not before)
  If DB fails: key not set → retry makes fresh attempt → safe
  If key set before DB: DB failure → key set but no record → next retry is blocked → money lost
  Rule: idempotency key is the RESULT cache; set it AFTER the result exists

SAGA vs 2PC:
  2PC: requires all participants to implement protocol → IMPOSSIBLE with external banks
  Saga: compensating transactions → void auth on failure → money movement reversed
  Payment saga: AUTHORISE → (CAPTURE or VOID) → SETTLE → RECONCILE

PAYMENT STATE MACHINE (memorize):
  CREATED → AUTHORISED → CAPTURED → SETTLED → RECONCILED
  AUTHORISED → VOIDED (cancel before capture)
  CAPTURED → REFUNDED (return after capture)
  Any → FAILED (decline or unrecoverable error)
  TIMEOUT → AUTHORISATION_UNKNOWN → MANUAL REVIEW (never assume)

SCC LENS:
  STATE: PostgreSQL (payments/ledger/events — authoritative), Redis (idempotency — operational)
  COORDINATION: idempotency SET NX, outbox atomic write, saga orchestrator step sequencing
  CONCENTRATION: card network (external) → circuit breaker; multiple network providers

IAM PLATFORM TIE-IN:
  Payment API auth: OAuth2 client_credentials for merchants (machine-to-machine)
  Token scopes: payments:create, payments:read, refunds:create — RBAC enforced by OPA
  Vault access: ABAC — only payment-service with correct service identity can call vault
  Dual-approval (refunds > ₹50K): CIAM workflow — maker submits, checker approves
  PCI audit log: every payment state change → Case 10 audit service
```

---

*Next: Case Study 22 — Fraud Detection System*
