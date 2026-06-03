# Case Study 24 — Distributed Ledger / Double-Entry Accounting

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Double-Entry invariants, Immutability, Balance computation at scale.

---

## Business Context

A financial ledger is the core of any payment platform. It records every movement of money in a way that is auditable, immutable, and always balanced. Stripe's ledger, Coinbase's ledger, and India's banking core all solve the same fundamental problem: track money with absolute correctness, at scale, forever.

Unlike a regular database where you update balances, a ledger is append-only. You never update a balance — you record a new entry that changes the balance. This immutability is what makes audits possible.

**Business goals driving architecture:**
- Absolute correctness: every rupee accounted for; zero loss
- Immutability: historical entries never change; corrections are new entries
- Sub-100ms balance reads for checkout flows
- Regulatory: 7-year retention; RBI and SEBI require complete audit trail
- Scale: 1B entries/day across 100M accounts

---

## Recognition Framework

### Signals from the Problem
```
- Write-once semantics: ledger entries are never updated or deleted
- Balance = aggregate of historical entries → expensive to compute from scratch
- High read/write: both balance reads and entry writes are in payment critical path
- Multi-currency: entries in different currencies; balances per currency
- Multi-entity: accounts for users, merchants, platform, tax, fees
- Strong consistency: balance must never show insufficient funds for committed transaction
- Immutability for audit: 7-year retention with ability to replay and verify
```

### Balance Computation Options

```
Option A: Compute balance from all entries on every read
  SELECT SUM(amount) FROM entries WHERE account_id = ? AND entry_type = 'CREDIT'
  MINUS
  SELECT SUM(amount) FROM entries WHERE account_id = ? AND entry_type = 'DEBIT'
  
  Pros: always correct; no cached state
  Cons: at 1B entries/account: this query takes seconds
  Verdict: ONLY viable for accounts with < 1K entries

Option B: Cached balance (materialised view)
  Maintain accounts table with current_balance
  On each entry INSERT: UPDATE accounts SET balance = balance + delta
  
  Pros: O(1) balance read
  Cons: balance update must be atomic with entry insert → transactional requirement
        if balance update fails: ledger entry exists but balance wrong
  Solution: same PostgreSQL transaction (entry INSERT + balance UPDATE)
  Verdict: CHOSEN for most accounts

Option C: Balance checkpoints (snapshot + delta)
  Every N entries: record a balance checkpoint
  Balance = last_checkpoint.balance + SUM(entries since checkpoint)
  
  Pros: bounded computation regardless of history length
  Cons: checkpoint creation must be atomic; checkpoint lag means slightly stale
  Use: offline balance reporting; large accounts (millions of entries)

CHOSEN: Hybrid — materialised balance for operational reads + checkpoints for audit
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Double-Entry Bookkeeping | Every debit has equal credit; sum of all entries = 0 |
| Append-Only Log | Immutability; corrections are new entries; complete history |
| Materialised Balance | O(1) balance read for payment authorisation |
| Optimistic Locking | Prevent concurrent balance update races |
| Account Types (Chart of Accounts) | Asset / Liability / Revenue / Expense / Equity |
| Idempotency | Same entry must not be inserted twice |

---

## Problem Statement

Distributed financial ledger for a payment platform. Tracks money movement across user, merchant, platform, and fee accounts. Double-entry bookkeeping. Immutable entries. Sub-100ms balance reads.

---

## Functional Requirements
- Record financial transactions as double-entry ledger entries
- Query account balance (current + historical at any point in time)
- Query transaction history per account (paginated)
- Multi-currency support
- Idempotent entry creation (same transaction cannot create duplicate entries)
- Balance verification: sum of all account balances = 0
- Statement generation: account statement for any date range

## Non-Functional Requirements
- Entry creation latency: **< 50ms** (in critical payment path)
- Balance read latency: **< 10ms** (in checkout flow)
- Immutability: **no UPDATE or DELETE** on ledger_entries
- Write throughput: **1B entries/day** (2 entries per payment × 500M payments)
- Retention: **7 years**

---

## Capacity Estimation
```
Entries:
  500M payments/day × 2 entries each = 1B entries/day
  Entry size: ~200 bytes
  Daily storage: 1B × 200 bytes = 200 GB/day
  7-year hot storage: 200 GB × 2555 = ~500 TB (too large for single PostgreSQL)
  Solution: 90-day hot (PostgreSQL) + cold archive (S3 Parquet)

Accounts:
  Users: 100M × 1 currency = 100M accounts
  Merchants: 10M × 2 currencies avg = 20M accounts
  Platform accounts: ~1,000 (fee, tax, reserve, suspense)
  Balance table: ~120M rows × 100 bytes = 12 GB → fits in DB with indexing

Balance reads:
  At checkout: every payment reads balance to verify sufficient funds
  50K TPS × 1 balance read = 50K reads/sec
  → Must be cache-backed or materialised (not computed from entries)
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Append-Only Ledger (Cassandra) | Immutable; time-series; high-write; 90-day hot |
| Account Balance (PostgreSQL) | Materialised; O(1) read; transactional update with entry |
| Double-Entry Enforcement | Application invariant: every transaction = 2+ entries; SUM = 0 |
| Idempotency Key | Same payment_id → same ledger entries; no duplicate |
| Balance Checkpoint | Periodic snapshot; fast historical balance computation |
| Optimistic Lock | account.version → prevent race condition on concurrent balance updates |

---

## Data Model

```sql
-- Accounts (PostgreSQL — materialised balances)
CREATE TABLE accounts (
  account_id       VARCHAR(64)  PRIMARY KEY,   -- user:u-12345:INR / merchant:m-xyz:INR
  account_type     VARCHAR(32)  NOT NULL,       -- ASSET / LIABILITY / REVENUE / EXPENSE
  entity_type      VARCHAR(16)  NOT NULL,       -- USER / MERCHANT / PLATFORM
  entity_id        VARCHAR(64)  NOT NULL,
  currency         CHAR(3)      NOT NULL,
  balance          BIGINT       NOT NULL DEFAULT 0,  -- in smallest unit (paise); NEVER NEGATIVE for assets
  available_balance BIGINT      NOT NULL DEFAULT 0,  -- balance minus holds
  hold_balance     BIGINT       NOT NULL DEFAULT 0,  -- authorised but not captured
  version          BIGINT       NOT NULL DEFAULT 0,  -- optimistic lock
  last_entry_id    VARCHAR(64),                -- last ledger entry ID (for verification)
  created_at       TIMESTAMP    DEFAULT NOW(),
  updated_at       TIMESTAMP    DEFAULT NOW()
);

-- Balance holds (for UPI/card authorisations)
CREATE TABLE balance_holds (
  hold_id          UUID         PRIMARY KEY,
  account_id       VARCHAR(64)  NOT NULL,
  amount           BIGINT       NOT NULL,
  reason           VARCHAR(64),                -- payment:pay-abc / mandate:man-xyz
  expires_at       TIMESTAMP,                  -- auto-release after this time
  created_at       TIMESTAMP    DEFAULT NOW()
);
```

```cql
-- Ledger entries (Cassandra — append-only; immutable; high-write)
CREATE TABLE ledger_entries (
  account_id       TEXT,
  entry_time       TIMESTAMP,
  entry_id         UUID,         -- globally unique
  transaction_id   TEXT,         -- payment_id or batch_id (for grouping)
  idempotency_key  TEXT,         -- dedup: transaction_id + account_id + entry_type
  entry_type       TEXT,         -- DEBIT / CREDIT / HOLD / HOLD_RELEASE / REVERSAL
  amount           BIGINT,       -- always positive (sign determined by entry_type)
  running_balance  BIGINT,       -- balance AFTER this entry (computed at write time)
  description      TEXT,
  metadata         TEXT,         -- JSON: counterparty, reference, notes
  created_at       TIMESTAMP,
  PRIMARY KEY (account_id, entry_time, entry_id)
) WITH CLUSTERING ORDER BY (entry_time DESC, entry_id DESC)
  AND default_time_to_live = 7776000;  -- 90 days hot; archived to S3

-- Idempotency table for ledger
CREATE TABLE entry_idempotency (
  idempotency_key  TEXT PRIMARY KEY,
  entry_id         UUID,
  created_at       TIMESTAMP
) WITH default_time_to_live = 86400;  -- 24h dedup window
```

---

## API Design

```
POST /api/v1/ledger/transactions
Body: {
  "transaction_id": "pay-abc-ledger",    // idempotency key for this set of entries
  "description": "Payment from Alice to Merchant X",
  "entries": [
    {
      "account_id": "user:u-12345:INR",
      "entry_type": "DEBIT",
      "amount": 50000,                   // 500 INR in paise
      "description": "Payment to merchant"
    },
    {
      "account_id": "merchant:m-xyz:INR",
      "entry_type": "CREDIT",
      "amount": 47500,                   // 475 INR (after 5% platform fee)
      "description": "Payment received"
    },
    {
      "account_id": "platform:fees:INR",
      "entry_type": "CREDIT",
      "amount": 2500,                    // 25 INR platform fee
      "description": "Transaction fee"
    }
  ]
}
Response 201: {
  "transaction_id": "pay-abc-ledger",
  "entries_created": 3,
  "debit_total": 50000,
  "credit_total": 50000,             // must equal debit_total
  "timestamp": "2024-01-01T14:00:00Z"
}
Response 200: (idempotency: same transaction_id → return existing entries)
Response 400: { "error": "DOUBLE_ENTRY_VIOLATION", "debit_total": 50000, "credit_total": 47500 }

GET /api/v1/accounts/{account_id}/balance
Response 200: {
  "account_id": "user:u-12345:INR",
  "balance": 500000,          // 5000 INR in paise
  "available_balance": 450000, // balance minus active holds
  "hold_balance": 50000,
  "currency": "INR",
  "as_of": "2024-01-01T14:00:01Z"
}

GET /api/v1/accounts/{account_id}/entries?from=2024-01-01&to=2024-01-31&cursor=&limit=50
Response 200: {
  "entries": [...],
  "opening_balance": 200000,   // balance at start of period
  "closing_balance": 500000,   // balance at end of period
  "total_credits": 600000,
  "total_debits": 300000
}
```

---

## High-Level Architecture

```
Ledger Service (stateless; multiple instances)
  │
  ├── POST /ledger/transactions (create journal entry):
  │
  │   Step 1: Idempotency check
  │     Cassandra: SELECT from entry_idempotency WHERE key = transaction_id
  │     HIT: return existing entries
  │     MISS: proceed
  │
  │   Step 2: Validate double-entry invariant
  │     SUM(DEBIT entries) must == SUM(CREDIT entries)
  │     If violated: return 400 DOUBLE_ENTRY_VIOLATION
  │
  │   Step 3: Balance check (for DEBIT entries on ASSET accounts)
  │     SELECT balance FROM accounts WHERE account_id = ?
  │     If balance - debit_amount < 0: return 402 INSUFFICIENT_BALANCE
  │
  │   Step 4: PostgreSQL transaction (ATOMIC):
  │     BEGIN;
  │       INSERT INTO balance_entries (idempotency_key, entry_id) -- dedup record
  │       For each DEBIT entry on ASSET account:
  │         UPDATE accounts SET
  │           balance = balance - amount,
  │           available_balance = available_balance - amount,
  │           version = version + 1
  │         WHERE account_id = ? AND version = {expected_version}  -- optimistic lock
  │         IF 0 rows updated: ROLLBACK; retry (concurrent modification detected)
  │       For each CREDIT entry:
  │         UPDATE accounts SET
  │           balance = balance + amount,
  │           version = version + 1
  │         WHERE account_id = ?
  │     COMMIT;
  │
  │   Step 5: Cassandra write (AFTER PostgreSQL commit)
  │     For each entry: INSERT INTO ledger_entries
  │     Write running_balance (current balance after this entry)
  │
  │   Return 201
  │
  └── GET /accounts/{id}/balance:
        SELECT balance, available_balance FROM accounts WHERE account_id = ?
        → Single row lookup; < 10ms
        → No Cassandra query needed for current balance
```

---

## Detailed Components

### Double-Entry Enforcement
```java
@Service
public class LedgerService {

    @Transactional
    public LedgerTransaction createTransaction(TransactionRequest req) {
        // Enforce double-entry invariant
        long debitTotal = req.getEntries().stream()
            .filter(e -> e.getType() == DEBIT)
            .mapToLong(LedgerEntry::getAmount).sum();

        long creditTotal = req.getEntries().stream()
            .filter(e -> e.getType() == CREDIT)
            .mapToLong(LedgerEntry::getAmount).sum();

        if (debitTotal != creditTotal) {
            throw new DoubleEntryViolationException(
                "Debit total " + debitTotal + " ≠ Credit total " + creditTotal
            );
        }

        // Idempotency
        if (idempotencyRepo.exists(req.getTransactionId())) {
            return idempotencyRepo.get(req.getTransactionId());
        }

        // Balance checks and updates (optimistic locking)
        for (LedgerEntry entry : req.getEntries()) {
            if (entry.getType() == DEBIT) {
                Account account = accountRepo.findById(entry.getAccountId());

                if (account.getBalance() < entry.getAmount()) {
                    throw new InsufficientBalanceException(entry.getAccountId());
                }

                int updated = accountRepo.updateBalance(
                    entry.getAccountId(),
                    -entry.getAmount(),
                    account.getVersion()  // optimistic lock
                );
                if (updated == 0) throw new OptimisticLockException();
            } else {
                accountRepo.updateBalance(entry.getAccountId(), +entry.getAmount(), null);
            }
        }

        // Write immutable audit entries to Cassandra
        for (LedgerEntry entry : req.getEntries()) {
            cassandraRepo.insertEntry(entry);
        }

        return new LedgerTransaction(req.getTransactionId(), req.getEntries());
    }
}
```

### Balance Verification (Invariant Check)
```sql
-- The golden rule: sum of ALL account balances = 0
-- (Platform accounts offset user/merchant accounts)
-- Run daily as part of reconciliation

-- Check 1: Sum of all balances must equal 0
SELECT SUM(
  CASE WHEN account_type = 'ASSET' THEN balance
       WHEN account_type = 'LIABILITY' THEN -balance
       WHEN account_type = 'REVENUE' THEN -balance
       WHEN account_type = 'EXPENSE' THEN balance
  END
) as total_balance
FROM accounts
WHERE currency = 'INR';
-- Must equal 0; if not: data integrity violation

-- Check 2: Individual account balance = sum of its ledger entries
-- (Verify materialised balance against append-only log)
SELECT a.account_id,
       a.balance as materialised_balance,
       SUM(CASE WHEN e.entry_type = 'CREDIT' THEN e.amount
                WHEN e.entry_type = 'DEBIT'  THEN -e.amount
           END) as computed_balance,
       a.balance - SUM(...)  as discrepancy
FROM accounts a
JOIN ledger_entries e ON e.account_id = a.account_id
GROUP BY a.account_id, a.balance
HAVING ABS(a.balance - SUM(...)) > 0;
-- Any row returned = ledger corruption → CRITICAL alert

-- Run this check nightly; alert if any discrepancy
```

### Historical Balance (Point-in-Time)
```sql
-- "What was Alice's balance on 2024-01-15 at 23:59:59?"
-- Used for: statements, dispute resolution, regulatory reporting

-- Method 1: from Cassandra ledger entries
-- Find the last entry before the target timestamp → running_balance
SELECT running_balance
FROM ledger_entries
WHERE account_id = 'user:u-12345:INR'
  AND entry_time <= '2024-01-15 23:59:59'
ORDER BY entry_time DESC, entry_id DESC
LIMIT 1;
-- running_balance is stored per entry at write time → O(1) lookup

-- Method 2: Balance checkpoints + delta
-- If entry history is cold-archived (> 90 days), use checkpoints:
-- 1. Find latest checkpoint before target date
-- 2. Sum entries from checkpoint to target date
-- 3. Add delta to checkpoint balance
```

---

## Scaling Strategy
```
Phase 1 (0 → 100M entries/day):
  PostgreSQL for both accounts + ledger entries
  Simple double-entry enforcement in application layer
  Daily balance verification job

Phase 2 (100M → 1B entries/day):
  Cassandra for ledger entries (time-series; high-write)
  PostgreSQL for materialised account balances only
  Idempotency check in Cassandra before write

Phase 3 (1B → 10B entries/day):
  PostgreSQL sharding by account_id or entity_type
  Cassandra cluster with more nodes
  Balance checkpoint service (nightly)
  S3 Parquet cold archive for entries > 90 days

Phase 4 (global multi-currency):
  Regional ledger services (INR in India, USD in US)
  FX conversion service with lock-in rates
  Cross-currency transfers: dual entries (DEBIT INR + CREDIT USD equivalent)
  Netting service: aggregate interbank settlements
```

---

## Reliability Strategy
- **PostgreSQL HA**: Patroni; primary + 2 standbys; < 30s failover; brief pause on failover
- **Cassandra RF=3**: entry writes continue if 1 node fails; quorum reads
- **Optimistic lock retry**: on version conflict → retry (3×) with backoff; rare in practice
- **Balance integrity**: nightly verification job; CRITICAL alert on any discrepancy
- **Idempotency TTL**: 24h for same-day; for multi-day retries, use longer TTL or DB unique constraint

---

## Security Considerations (Audit + Regulatory)
- **No DELETE on ledger_entries**: DB role has no DELETE privilege; Cassandra TTL for archival only
- **Maker-checker for large entries**: any entry > ₹1M requires dual approval (CS21 pattern)
- **Audit log**: every ledger transaction → Case 10 audit service
- **Account access control**: OPA policy — service X can only read/write accounts of type Y
- **Balance tampering detection**: running_balance in Cassandra is cross-checked vs materialised balance nightly
- **Segregation of duties**: engineers cannot modify live ledger entries; only ops with 4-eyes approval

---

## Observability
```
Financial integrity metrics:
  balance_verification_result        (daily; PASS or FAIL with discrepancy amount)
  double_entry_violation_rate        (should be 0; any > 0 = application bug)
  ledger_write_success_rate          (target > 99.99%; any failure = payment failed)
  entry_count_per_day                (trend; alert on unusual spike or drop)
  optimistic_lock_retry_rate         (high rate = hot accounts under contention)

Operational metrics:
  balance_read_latency_p99           (target < 10ms; critical for checkout)
  entry_write_latency_p99            (target < 50ms)
  cassandra_write_lag                (entries committed to Cassandra vs PostgreSQL)
  account_balance_cache_hit          (if using Redis L1 for balance reads)

Compliance metrics:
  ledger_retention_completeness      (all entries accessible for audit date range)
  statement_generation_accuracy      (opening + txns = closing for every statement)
```

---

## Chaos Testing
```
Experiment 1: Concurrent Balance Update (Race Condition)
  Inject:    100 concurrent requests all debiting from same account simultaneously
  Expected:  Optimistic locking: only 1 update per version; others retry
  Expected:  Final balance = initial_balance - SUM(all debit amounts)
  Expected:  No debit exceeds available balance (no overdraft)
  Red flag:  Race condition: balance goes negative despite lock

Experiment 2: Double-Entry Violation Injection
  Inject:    Submit transaction with DEBIT=500, CREDIT=400 (different totals)
  Expected:  Application validates: 500 ≠ 400 → return 400 DOUBLE_ENTRY_VIOLATION
  Expected:  No entries created; no balance changes
  Red flag:  Unbalanced entry accepted and written to ledger

Experiment 3: Idempotency Under Retry Storm
  Inject:    Submit same transaction_id 100 times concurrently
  Expected:  First call: creates entries; returns 201
  Expected:  All subsequent calls: idempotency check → return same 201 response
  Expected:  Cassandra ledger: exactly 2 entries (not 200)
  Red flag:  Multiple entries created for same transaction_id

Experiment 4: Balance Integrity Verification
  Inject:    Manually UPDATE one account balance in PostgreSQL (bypass application)
  Expected:  Nightly balance_verification job detects discrepancy:
             materialised_balance ≠ computed_from_entries
  Expected:  CRITICAL alert fires
  Expected:  Case created for manual investigation
  Red flag:  Verification job reports PASS despite manual tampering
```

---

## Monthly Cost Estimate
```
Scale: 1B entries/day, 120M accounts

PostgreSQL (account balances — critical path):
  Primary: db.r6g.8xlarge ($3,200/mo)
  2 read replicas: $3,200/mo each
  Total PostgreSQL:                                  = $9,600/mo

Cassandra (ledger entries — time-series):
  20 nodes × r6g.4xlarge ($800/mo)                  = $16,000/mo

S3 (cold archive > 90 days):
  200 GB/day × 90 days = 18 TB × 5× factor = 90 TB × $0.023/GB = $2,070/mo

Balance verification job (EMR Spark; daily 2h):
  50 nodes × 2h × $0.043/hr × 30 days               = $6,450/mo (one-time monthly)
  Wait — 50 nodes × 2h × $0.043 × 30 = $129/mo     = $129/mo

Ledger Service API (12 instances):
  12 × c5.4xlarge ($560/mo)                          = $6,720/mo

Redis (balance L1 cache; optional):
  3 × cache.r6g.2xlarge ($500/mo)                    = $1,500/mo
─────────────────────────────────────────────────────────────────
Total: ~$33,949/month
Cassandra (47%) and PostgreSQL (28%) dominate — financial data is expensive to store correctly.
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL accounts.balance: operational truth (for balance checks in payments).
  → Cassandra ledger_entries: historical truth (for audits, statements, verification).
  → S3 Parquet: cold archive truth (for 7-year regulatory retention).
  → If PostgreSQL and Cassandra disagree: Cassandra is authoritative (append-only log wins).

□ Double-entry invariant:
  → Enforced at application layer before any DB write.
  → Verified nightly: SUM(all account balances) = 0.
  → Any violation = CRITICAL alert + halt new transactions until resolved.

□ Immutability:
  → No DELETE privilege on any ledger table for application role.
  → TTL (Cassandra) for archival only → entries move to S3, not deleted.
  → Corrections are new REVERSAL entries + new CORRECT entries.
  → Historical balance point-in-time computed from entry log; never from "corrected" state.

□ Failure mode?
  → PostgreSQL failover: balance reads/writes pause ~30s; optimistic lock retry handles.
  → Cassandra node down: RF=3; writes continue to 2 remaining nodes; quorum read.
  → Entry written to PostgreSQL but not Cassandra: entry log incomplete; reconciliation catches.
    Mitigation: Cassandra write is synchronous (before response to caller); if fails → rollback PG transaction.

□ SCC LENS:
  STATE: PostgreSQL (materialised balances — fast), Cassandra (immutable entries — authoritative)
  COORDINATION: optimistic lock (concurrent balance updates), idempotency key (duplicate entries)
  CONCENTRATION: hot accounts (high-volume merchants) → contention on optimistic lock → shard by account or batch entries
```

---

*Next: Case Study 25 — API Rate Limiter at Stripe Scale*
