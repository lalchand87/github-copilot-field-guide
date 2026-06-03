# Case Study 23 — UPI Payment System

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: NPCI integration, VPA resolution, Collect flow vs Pay flow.

---

## Business Context

UPI (Unified Payments Interface) is India's real-time interbank payment system. It settles payments instantly between any two bank accounts using a Virtual Payment Address (VPA like alice@okhdfc). UPI processed 14B+ transactions in Jan 2024. Third-party apps (PhonePe, Google Pay, Paytm) are TSPs (Third-Party Application Providers) that front-end UPI via NPCI's infrastructure.

For a fintech TSP: the core technical challenge is orchestrating real-time payment flows through NPCI's infrastructure while guaranteeing exactly-once payment, handling bank timeouts gracefully, and reconciling settlement correctly.

**Business goals driving architecture:**
- Transaction completion < 10 seconds (UPI mandate)
- Zero duplicate debits (exactly-once via NPCI transaction ID)
- Handle bank downtime gracefully (some banks are frequently slow)
- Real-time reconciliation with NPCI settlement files
- PCI and RBI compliance

---

## Recognition Framework

### Signals from the Problem
```
- External dependency (NPCI + issuer bank + acquirer bank) with timeouts
- Exactly-once via external transaction reference (NPCI txn ID)
- Push payment (pay) vs pull payment (collect) — different flows and trust models
- VPA resolution: alice@okhdfc → actual bank account → happens at NPCI
- Bank downtime: NPCI reports bank success/failure; must handle all combinations
- Settlement: NPCI settles on T+0 or T+1; must reconcile our records
- Mandate payments: recurring UPI mandates require pre-approved consent framework
- Deep link: UPI deep links (upi://pay?pa=alice&pn=Alice&am=100) from QR codes
```

### UPI Transaction Flow

```
Pay Flow (payer initiates):
  1. Payer enters VPA or scans QR code
  2. TSP resolves VPA: NPCI lookup → bank + masked account number
  3. Payer enters UPI PIN on app (NPCI/bank validates pin in encrypted form)
  4. TSP sends Pay request to NPCI (UPI 2.0 API)
  5. NPCI debits payer's bank (issuer bank) → credits payee's bank (acquirer bank)
  6. NPCI returns transaction result to TSP
  7. TSP updates payment status; sends push notification to payer + payee
  
  Timeout handling:
    NPCI may return TIMEOUT (bank not responding)
    TSP must CHECK STATUS (not retry immediately — could double debit)
    Status check: GET /status/{npci_txn_id} → PENDING / SUCCESS / FAILURE

Collect Flow (payee requests payment from payer):
  1. Payee sends collect request (pay me ₹500, expires in 30 min)
  2. NPCI notifies payer's TSP: "you have a collect request"
  3. Payer approves or rejects in their UPI app
  4. On approval: payer enters UPI PIN; payment flows same as Pay flow
  5. TSP notifies payee of approval/rejection

Key difference:
  Pay: payer controls; immediate; push payment
  Collect: payee controls; requires payer action; pull payment
  Collect expiry: payee sets expiry; NPCI auto-expires unpaid requests
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| NPCI Transaction ID (Idempotency) | Never re-initiate; always check status on timeout |
| Status Check Pattern | NPCI timeout → poll status, not retry → prevents double debit |
| VPA Resolution Cache | VPA→bank lookup is slow; cache for frequently-used VPAs |
| Debit/Credit State Machine | UPI payment has defined states per RBI mandate |
| Reconciliation (T+0 NPCI file) | NPCI sends settlement file; verify against our DB |
| Mandate Consent Store | Recurring payments require pre-authorised consent |

---

## Problem Statement

UPI TSP (Third-Party Application Provider) backend. Handles pay, collect, mandate, and QR code payments. Integrates with NPCI UPI switch. Handles bank downtimes gracefully. Reconciles settlement daily.

---

## Functional Requirements
- Pay: send money to VPA (instant, < 10s)
- Collect: request payment from another VPA
- QR code: generate and parse UPI QR codes (BharatQR standard)
- Mandate: create, modify, revoke UPI AutoPay mandates
- VPA management: register and manage user VPAs
- Balance inquiry: check UPI-linked bank account balance
- Transaction history: last 90 days of UPI transactions
- Dispute: flag a transaction for dispute with NPCI

## Non-Functional Requirements
- Pay latency: **< 10 seconds** (UPI regulation mandate)
- Availability: **99.95%** (UPI is critical payment infrastructure)
- Exactly-once: **zero duplicate debits** (RBI mandate)
- Reconciliation: **daily T+0** (match NPCI settlement file)
- Scale: **50,000 UPI TPS** (peak festival traffic)

---

## Capacity Estimation
```
UPI transactions:
  50K TPS × 86400s × 10% active fraction = ~432M UPI txns/day
  Each transaction: ~1 KB state
  Daily storage: 432M × 1 KB = ~432 GB

VPA resolution:
  50K TPS → 50K VPA resolutions/sec (one per payment)
  NPCI VPA API: 200ms per resolution
  With cache (90% hit rate): 5K uncached/sec × 200ms = 1K concurrent NPCI calls
  Cache size: top 10M VPAs × 100 bytes = 1 GB (fits in Redis)

NPCI connection pool:
  50K TPS × 10s max duration = 500K concurrent connections needed?
  No: NPCI is synchronous; payment completes in < 5s for most banks
  Realistically: 50K concurrent NPCI calls at peak → NPCI has per-TSP rate limits
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| NPCI Txn ID as idempotency | External reference; check status, never retry blindly |
| VPA Resolution Cache | 90% cache hit; reduces NPCI API load |
| Timeout + Status Check | Bank timeout → query NPCI status before any retry |
| Payment State Machine | NPCI-defined states; RBI-mandated state transitions |
| Reconciliation Engine | Daily NPCI file comparison; flag mismatches |
| Mandate Store | Pre-authorised consent with expiry and amount limits |

---

## Data Model

```sql
-- UPI Transactions
CREATE TABLE upi_transactions (
  txn_id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
  npci_txn_id          VARCHAR(64)  UNIQUE,      -- NPCI's transaction reference
  txn_ref_id           VARCHAR(64)  UNIQUE,       -- our reference; in QR/deep link
  txn_type             VARCHAR(16)  NOT NULL,     -- PAY / COLLECT / MANDATE_EXEC
  payer_vpa            VARCHAR(128) NOT NULL,
  payee_vpa            VARCHAR(128) NOT NULL,
  payer_user_id        VARCHAR(64),              -- our user (if payer is our user)
  payee_user_id        VARCHAR(64),              -- our user (if payee is our user)
  amount               BIGINT       NOT NULL,     -- in paise
  status               VARCHAR(32)  NOT NULL DEFAULT 'INITIATED',
  -- INITIATED → PENDING → SUCCESS / FAILURE / TIMEOUT / DEEMED_SUCCESS
  failure_code         VARCHAR(16),              -- NPCI error code
  failure_message      TEXT,
  bank_response_code   VARCHAR(8),               -- issuer bank's response
  initiated_at         TIMESTAMP    DEFAULT NOW(),
  completed_at         TIMESTAMP,
  npci_response        JSONB                     -- full NPCI response (masked)
);

-- VPA registry
CREATE TABLE vpa_registrations (
  vpa                  VARCHAR(128) PRIMARY KEY,  -- alice@okhdfc
  user_id              VARCHAR(64)  NOT NULL,
  bank_account_hash    VARCHAR(64),              -- masked
  issuer_bank          VARCHAR(32),              -- HDFC / ICICI / SBI etc.
  is_primary           BOOLEAN      DEFAULT FALSE,
  is_active            BOOLEAN      DEFAULT TRUE,
  registered_at        TIMESTAMP    DEFAULT NOW()
);

-- UPI Mandates (recurring payment consent)
CREATE TABLE upi_mandates (
  mandate_id           UUID         PRIMARY KEY,
  npci_mandate_id      VARCHAR(64)  UNIQUE,
  payer_vpa            VARCHAR(128) NOT NULL,
  payee_vpa            VARCHAR(128) NOT NULL,
  mandate_type         VARCHAR(32), -- PERIODIC / ON_DEMAND
  frequency            VARCHAR(16), -- MONTHLY / WEEKLY / DAILY / ONE_TIME
  max_amount           BIGINT,      -- per execution cap in paise
  start_date           DATE,
  end_date             DATE,
  status               VARCHAR(16)  DEFAULT 'ACTIVE',  -- ACTIVE / PAUSED / REVOKED
  consent_hash         VARCHAR(64), -- NPCI consent signature
  created_at           TIMESTAMP    DEFAULT NOW()
);
```

---

## API Design

```
POST /api/v1/upi/pay
Body: {
  "payer_vpa": "alice@okhdfc",
  "payee_vpa": "merchant@paytm",
  "amount": 50000,             // 500.00 INR in paise
  "remarks": "Coffee",
  "txn_ref_id": "TXN123456",  // our reference; in notification to payee
  "upi_pin_encrypted": "..."  // encrypted UPI PIN (never stored; passed to NPCI)
}
Response 200: {
  "txn_id": "uuid",
  "npci_txn_id": "NPCI123456",
  "status": "SUCCESS",
  "payer_account_masked": "HDFC****1234",
  "payee_account_masked": "PAYTM****5678",
  "timestamp": "2024-01-01T14:00:00Z"
}
Response 200 with status=PENDING: { "txn_id": "...", "status": "PENDING", "check_after_ms": 5000 }
Response 200 with status=FAILURE: { "txn_id": "...", "status": "FAILURE", "failure_code": "U002" }

GET /api/v1/upi/transactions/{txn_id}/status
→ Poll for PENDING transactions

POST /api/v1/upi/collect
Body: { "payer_vpa": "alice@okhdfc", "amount": 50000, "remarks": "...", "expiry_minutes": 30 }
Response 202: { "collect_id": "...", "status": "PENDING_PAYER_ACTION" }

POST /api/v1/upi/vpa/validate?vpa=alice@okhdfc
Response 200: { "vpa": "alice@okhdfc", "payee_name": "Alice", "bank": "HDFC" }

QR Code generation:
GET /api/v1/upi/qr?amount=50000&payee_vpa=merchant@paytm&remarks=Coffee
Response 200: { "qr_string": "upi://pay?pa=merchant@paytm&pn=Merchant&am=500.00&tn=Coffee",
               "qr_image_url": "s3://qr-codes/qr-uuid.png" }
```

---

## High-Level Architecture

```
Mobile App (PhonePe/Google Pay equivalent)
  │
  ▼
UPI API Gateway (Rate limited; per-user throttle)
  │
  ├── VPA Validation (synchronous; before payment)
  │     1. Check VPA cache: GET vpa:{vpa} from Redis (TTL 1h)
  │        HIT: return cached bank name + masked account
  │        MISS: NPCI VPA Resolution API → cache result
  │
  ├── Pay Flow (synchronous; up to 10s):
  │     1. Validate payer VPA and payee VPA
  │     2. Create txn record (status=INITIATED)
  │     3. Call NPCI UPI API:
  │          POST /upi/pay { payer_vpa, payee_vpa, amount, encrypted_pin, txn_ref_id }
  │          Timeout: 8 seconds (2s reserve for processing)
  │     4. NPCI Response handling:
  │          SUCCESS:  update status=SUCCESS; notify both parties; return 200
  │          FAILURE:  update status=FAILURE; return 200 with failure_code
  │          TIMEOUT:  update status=PENDING; return 200 with "check_after_ms=5000"
  │     5. For TIMEOUT: start async status check loop
  │          GET /upi/status/{npci_txn_id} every 5s for up to 2 minutes
  │          If SUCCESS within 2 min: update status; notify
  │          If still pending at 2 min: status=DEEMED_SUCCESS (RBI rule: credit payee)
  │
  └── Collect Flow (async; up to 30 min):
        1. Create collect request → POST to NPCI
        2. NPCI notifies payer's TSP (our app or competitor's app)
        3. Payer approves/rejects on their device
        4. On approval: identical to Pay flow from step 3
        5. Webhook to payee: payment received/rejected

Settlement Reconciliation (daily, 02:00 UTC):
  1. Download NPCI settlement file (T+0 settlement report)
  2. For each entry in NPCI file:
     a. Find matching transaction in our DB by npci_txn_id
     b. Compare: amount, VPAs, status
     c. Mismatch: alert + create dispute ticket
     d. In NPCI file but not our DB: ghost transaction → investigate
     e. In our DB but not NPCI file: failed transaction our DB shows success → critical alert
```

---

## Detailed Components

### Timeout and Deemed Success (RBI Mandate)
```
RBI Circular: if a UPI transaction is in PENDING state and no response from banks
for > T+2 minutes, TSP must credit the payee (deemed success).

Implementation:
  Transaction initiated at T=0
  Bank did not respond by T=8s → status=PENDING
  
  Status check loop (async):
    T+5s:  GET /upi/status/{npci_txn_id} → PENDING
    T+10s: GET /upi/status/{npci_txn_id} → PENDING
    T+60s: GET /upi/status/{npci_txn_id} → PENDING
    T+120s: T+2 min elapsed → SET status=DEEMED_SUCCESS
            → CREDIT payee immediately (RBI mandate)
            → Notify payer: "Payment pending; if debited, payee credited"
  
  If bank later confirms FAILURE (T+3 days):
    → Initiate refund to payer (RBI mandates within 5 days)
    → This is a chargeback scenario; rare but must be handled

  Why DEEMED_SUCCESS instead of just waiting:
    RBI says: payee must receive credit; user experience must not be degraded
    Banks can be delayed hours due to core banking maintenance
    TSP assumes liability and credits payee from its own float
    Reconciliation later: bank confirms debit → TSP is square
    If bank confirms NO DEBIT: TSP absorbs loss or disputes with bank
```

### VPA Resolution and Caching
```python
async def resolve_vpa(vpa: str) -> VPAInfo:
    # Check cache first (Redis; TTL 1 hour for active VPAs)
    cached = await redis.get(f"vpa:{vpa}")
    if cached:
        return VPAInfo.from_json(cached)

    # Cache miss: call NPCI VPA resolution API
    try:
        response = await npci_client.resolve_vpa(vpa, timeout=3.0)
        info = VPAInfo(
            vpa=vpa,
            payee_name=response.payee_name,
            bank=response.issuer_bank,
            is_active=True
        )
        await redis.setex(f"vpa:{vpa}", 3600, info.to_json())  # TTL 1h
        return info

    except NpciVpaNotFound:
        # Cache negative result to prevent hammering NPCI
        await redis.setex(f"vpa:{vpa}:invalid", 300, "1")  # 5 min negative cache
        raise VPANotFoundException(vpa)

    except NpciTimeout:
        # Cannot resolve; fail fast
        raise VPAResolutionTimeoutException(vpa)

# Cache strategy:
#   Active VPAs: 1h TTL (VPAs rarely change)
#   Invalid VPAs: 5 min negative cache (prevent repeated NPCI calls for typos)
#   Changed VPA (bank switch): user must invalidate manually or wait for TTL
```

### Reconciliation Logic
```python
def reconcile_npci_settlement(date: date):
    """Run daily after NPCI settlement file is available."""

    # Load NPCI settlement file (CSV/XML)
    npci_records = parse_npci_settlement_file(date)

    # Load our DB records for same date
    our_records = {
        txn.npci_txn_id: txn
        for txn in db.query_by_date(date)
        if txn.npci_txn_id
    }

    mismatches = []
    for npci_rec in npci_records:
        our_txn = our_records.get(npci_rec.txn_id)

        if not our_txn:
            # NPCI shows txn; we have no record → CRITICAL
            mismatches.append(ReconciliationMismatch(
                type="GHOST_TRANSACTION",
                npci_txn_id=npci_rec.txn_id,
                severity="CRITICAL"
            ))

        elif our_txn.amount != npci_rec.amount:
            # Amount mismatch → investigate
            mismatches.append(ReconciliationMismatch(
                type="AMOUNT_MISMATCH",
                our_amount=our_txn.amount,
                npci_amount=npci_rec.amount,
                severity="HIGH"
            ))

        elif our_txn.status == "SUCCESS" and npci_rec.status == "FAILED":
            # We think it succeeded; NPCI says failed → refund customer
            mismatches.append(ReconciliationMismatch(
                type="STATUS_MISMATCH",
                action="INITIATE_REFUND",
                severity="CRITICAL"
            ))

    if mismatches:
        alert_on_call(f"Reconciliation found {len(mismatches)} mismatches for {date}")
        for m in mismatches:
            fraud_ops.create_case(m)

    return ReconciliationReport(date=date, mismatches=mismatches)
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K TPS):
  Single UPI service; single NPCI connection
  PostgreSQL for all state
  Synchronous everything

Phase 2 (1K → 10K TPS):
  Multiple UPI service instances
  Redis VPA cache + idempotency store
  Async notification delivery (Kafka → push notification worker)
  Connection pooling to NPCI

Phase 3 (10K → 50K TPS):
  NPCI gives higher rate limits (negotiated; based on transaction volume)
  Redis Cluster for VPA cache
  Read replicas for transaction history queries
  Async reconciliation (streaming, not just daily batch)

Phase 4 (50K+ TPS, UPI 3.0):
  Pre-validated VPA sessions (session token after first VPA validation)
  Streaming NPCI status updates (NPCI Webhook → we don't poll status)
  Real-time fraud signals fed from CS22 fraud engine
  Multi-bank connection: direct bank APIs for top-10 banks
```

---

## Reliability Strategy
- **NPCI rate limits**: TSPs have per-second limits; exceeded → queue payments with backpressure
- **Bank downtime detection**: track per-bank success rate; if < 80% → inform user "HDFC experiencing delays"
- **Circuit breaker per bank**: NPCI routes via bank; if bank circuit open → fail fast with "try another account"
- **Deemed success**: RBI-mandated safety net for persistent bank downtime
- **Reconciliation as final audit**: any system failure eventually caught by daily reconciliation

---

## Security Considerations (RBI + PCI)
- **UPI PIN**: NPCI-encrypted; never stored or logged at TSP layer; transient in memory only
- **Device binding**: UPI mandate requires device binding; second device requires re-KYC
- **VPA creation**: must verify mobile number ownership (OTP) before creating VPA
- **Collect fraud**: malicious collect request (fake merchant) → show verified merchant name + logo (NPCI provides)
- **QR code tampering**: BharatQR has HMAC signature; validate before displaying payee details
- **RBI regulations**: mandatory 4-digit MPIN, mandatory 2FA for amounts > ₹2,000

---

## Observability
```
UPI-specific metrics:
  upi_success_rate                    (target > 97%; RBI minimum)
  upi_latency_p99                     (target < 8s; UPI mandate)
  bank_success_rate by bank           (identify lagging banks)
  vpa_resolution_cache_hit_rate       (target > 90%)
  deemed_success_rate                 (% of txns needing deemed success; alert > 2%)
  collect_expiry_rate                 (% of collect requests that expire unpaid)
  reconciliation_mismatch_count       (should be 0; CRITICAL alert if > 0)
  npci_api_latency_p99                (target < 3s; if > 5s → degradation alert)

RBI compliance metrics:
  success_rate_24h                    (RBI requires > 99.5%; alert if drops)
  complaint_resolution_rate           (must resolve 95% of complaints within 5 days)
  chargeback_resolution_rate          (T+5 day deadline for refunds)
```

---

## Chaos Testing
```
Experiment 1: Bank Timeout on Pay Flow
  Inject:    HDFC bank API returns 504 after 9 seconds
  Expected:  NPCI times out; our service sets status=PENDING at 8s
  Expected:  Status check loop begins; at T+120s → DEEMED_SUCCESS
  Expected:  Payee credited (deemed success); payer's bank debit TBD
  Expected:  Alert fires: "HDFC bank timeout rate > 10%"
  Red flag:  Transaction retried with new NPCI txn_id → duplicate debit

Experiment 2: Duplicate Pay Submission (Network Retry)
  Inject:    Mobile app submits same payment twice (same txn_ref_id)
  Expected:  First: NPCI txn created; npci_txn_id stored
  Expected:  Second: txn_ref_id already in DB → return existing status (no new NPCI call)
  Expected:  NPCI never receives second request → zero duplicate debit risk
  Red flag:  Two NPCI transactions created for same txn_ref_id

Experiment 3: Reconciliation Mismatch Detection
  Inject:    Manually insert a transaction into DB that is NOT in NPCI settlement file
  Expected:  Reconciliation job: finds "in our DB, not in NPCI file"
  Expected:  Raises CRITICAL mismatch alert
  Expected:  Case created for ops team investigation
  Red flag:  Reconciliation passes despite data inconsistency

Experiment 4: VPA Cache Miss Storm (Cache Flush)
  Inject:    Flush all VPA cache entries from Redis
  Expected:  All VPA resolutions hit NPCI (100% cache miss)
  Expected:  NPCI API call rate spikes → may hit rate limit
  Expected:  If rate limited: queue VPA resolutions; return 429 to users
  Expected:  Cache warms back up within minutes as resolutions are cached
  Red flag:  Service crashes or drops requests during cache miss storm
```

---

## Monthly Cost Estimate
```
Scale: 50K TPS peak, 432M UPI transactions/day

UPI API Services:
  20 × c5.2xlarge ($280/mo)                        = $5,600/mo

Redis (VPA cache + idempotency):
  3 × cache.r6g.large ($130/mo)                    = $390/mo

PostgreSQL (transactions + mandates):
  db.r6g.4xlarge ($1,600/mo) + 2 read replicas    = $4,800/mo

NPCI Connectivity (leased line / direct connect):
  Dedicated NPCI connectivity                      = $5,000/mo
  (NPCI requires dedicated secure connectivity for TSPs)

Kafka (UPI events → fraud + analytics):
  3 × kafka.m5.large ($150/mo)                     = $450/mo

Monitoring + compliance tooling:
                                                    = $1,000/mo
─────────────────────────────────────────────────────────────────
Total: ~$17,240/month
NPCI connectivity (29%) + PostgreSQL (28%) + API compute (32%) are main costs.

Note: UPI transaction charges are ₹0 (per NPCI mandate for peer-to-peer).
Merchant payments: NPCI charges MDR (Merchant Discount Rate) to TSP.
TSP revenue: interchange sharing with banks + merchant MDR.
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL: UPI transactions and mandates (authoritative; immutable after success).
  → NPCI: ultimate financial truth (settlement file reconciled daily).
  → Redis: VPA cache (derived; expendable; rebuilt on miss).
  → If our DB and NPCI disagree: NPCI wins; we initiate refund or accept deemed success.

□ Exactly-once via NPCI txn_id:
  → We never re-initiate a payment; we always check status.
  → txn_ref_id (our reference) dedups on our side before NPCI call.
  → NPCI txn_id dedups at NPCI for any network-level retries.
  → Both defences needed: our dedup prevents extra NPCI calls; NPCI dedup is backup.

□ Failure mode?
  → Bank timeout → PENDING → status check → DEEMED_SUCCESS (RBI rule).
  → NPCI timeout → same as bank timeout.
  → DB write fails after NPCI success → transaction in NPCI but not our DB → ghost txn.
    Reconciliation catches this → create record retroactively → notify parties.

□ UPI-specific regulations (RBI):
  → Every transaction T+2 min rule: must credit or refund.
  → Complaint resolution: T+5 days for refunds.
  → Success rate > 99.5% or NPCI can delist TSP.
  → Device binding: mandatory for UPI PIN security.
  → Mandatory 2FA for amounts > ₹2,000.

□ SCC LENS:
  STATE: PostgreSQL (transactions — authoritative), Redis (VPA cache — operational)
  COORDINATION: txn_ref_id dedup (our side) + NPCI txn_id (NPCI side) + status check (timeout handling)
  CONCENTRATION: NPCI is the single concentration point for all UPI traffic → circuit breaker + retry discipline
```

---

## Staff Engineer Discussion Points

**"How does UPI handle the 'double debit' problem if my app crashes after NPCI confirms debit but before I update my DB?"**
This is the "write after external call" problem — identical to the payment gateway's dangling authorisation. Our defence: when we create the NPCI call, we store `npci_txn_id` and `status=PENDING` in our DB before calling NPCI. If the app crashes after NPCI success: the transaction is in our DB as PENDING. The daily reconciliation job sees: "NPCI settlement shows this npci_txn_id as SUCCESS; our DB shows PENDING → update to SUCCESS; notify parties." No money is lost and no duplicate debit occurs because: (1) we store NPCI txn_id atomically before the call, and (2) we never retry with a new NPCI txn_id for the same txn_ref_id.

**"What's the difference between UPI and IMPS technically?"**
IMPS (Immediate Payment Service): older protocol; uses IFSC + account number; no VPA; no QR; HTTP API with NPCI; slower adoption. UPI: built on IMPS rails but adds VPA abstraction, MPIN authentication (no bank password), QR codes, deep links, collect flow, mandates. UPI is the application layer; IMPS is the settlement layer underneath. When your UPI payment succeeds at night, NPCI settles it via IMPS under the hood. UPI innovation: frictionless UX via VPA + PIN (customer never sees bank account numbers).

---

## Cheat Sheet Tie-in

```
UPI-SPECIFIC PATTERNS:
  TIMEOUT → CHECK STATUS (never retry directly) → prevents double debit
  DEEMED_SUCCESS: T+2 min PENDING → credit payee; reconcile later with bank
  VPA resolution cache: resolve once; cache 1h; prevents NPCI rate limit
  Reconciliation: NPCI settlement file is the financial ground truth

UPI FLOWS (memorize):
  Pay (push): payer initiates → NPCI → SUCCESS/FAILURE/PENDING
  Collect (pull): payee requests → payer approves → same as Pay from step 3
  Mandate: pre-authorised consent → TSP executes periodically without PIN

SCC LENS:
  STATE: PostgreSQL (transactions), Redis (VPA cache), NPCI (ultimate financial truth)
  COORDINATION: txn_ref_id dedup prevents double NPCI calls; status check handles timeout
  CONCENTRATION: NPCI is the bottleneck → rate limits → TSP needs NPCI goodwill + quotas

IAM TIE-IN:
  UPI PIN is NOT managed by our IAM; it is bank-managed via NPCI
  Device binding: our IAM registers device_id → NPCI; deregistration via our IAM
  Collect fraud prevention: ABAC — only verified merchants can initiate collect > ₹5,000
  Mandate consent: OAuth2-like consent model — payer grants mandate; TSP stores consent
```

---

*Next: Case Study 24 — Distributed Ledger / Double-Entry Accounting*
