# Case Study 36 — Idempotency Patterns (Deep Dive)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Key placement, Network retry scenarios, Distributed idempotency.

---

## Business Context

Idempotency is the single most important correctness property in payment systems. Every distributed system experiences network failures, timeouts, and client retries. Without idempotency, a retry causes a duplicate charge. With idempotency, a retry returns the same result as the original call — no duplicate effect.

This case study goes deeper than Case 21 (which introduced idempotency in the context of payment gateway). Here we systematically cover all the failure scenarios, the correct key placement, multi-service idempotency, and the database patterns that make idempotency reliable.

**The fundamental contract:** "Call me N times with the same idempotency key → same result, same effect."

---

## Recognition Framework

### All the Ways Retries Cause Problems

```
Scenario 1: Network timeout on client side
  Client → [timeout] → Server (request received and processed)
  Client: "Did the server receive it? Let me retry."
  Without idempotency: second call charges customer twice
  With idempotency: second call returns first call's response

Scenario 2: Server crash after processing, before responding
  Client → Server processes payment → Server crashes → Client never gets response
  Client: "No response; retry."
  Without idempotency: same problem
  With idempotency: server on restart: "I processed this idempotency key already"

Scenario 3: Load balancer retry
  Client → LB → Server A (fails; no response to LB)
  LB retries: Client → LB → Server B
  Server A may have partially processed the request
  Without distributed idempotency: Server B processes again → duplicate

Scenario 4: Message queue redelivery
  Producer → Kafka (at-least-once delivery)
  Consumer receives message twice (partition reassignment, rebalance)
  Without idempotent consumer: processes same payment event twice

Scenario 5: Batch job retried after partial failure
  Job processes 1M records; fails at record 500K
  Retry starts from beginning (no checkpoint)
  Without idempotency: first 500K processed twice → duplicated charges/entries
```

### Idempotency Key Design Principles

```
Principle 1: Caller-generated (not server-generated)
  Server generates key: if first call times out, client can't reuse same key
  Caller generates key: client generates UUID before calling; uses same UUID on retry
  
  Rule: idempotency key is ALWAYS caller-generated

Principle 2: Semantically unique per intent
  Good: "payment-order-{order_id}-attempt-{attempt_number}"
    → If payment fails and customer clicks "Pay Again": new attempt number → new payment
    → If client retries same attempt: same key → same response
  
  Bad: UUID.randomUUID() on every call
    → Every retry generates new UUID → no deduplication
    → Defeats the purpose
  
  Bad: just the user_id or session_id
    → Deduplicates all payments from same user → only one payment ever succeeds

Principle 3: Key scope and TTL
  Scope: unique within the context (payment for order X)
  TTL: how long to keep the result (retry window)
    Short-lived APIs: 24 hours (normal retry window)
    Idempotent batch: 7 days (batch jobs may retry after days)
    Critical financial ops: 30 days (disputes and reconciliation)

Principle 4: Key SET timing (this is the critical one)
  WRONG: set key → call external system → return response
    Network fails after external system processes but before key is set → key not set → retry creates duplicate
  
  CORRECT: call external system → store result in DB → set key pointing to result
    If DB fails: key not set → retry makes fresh external call (safe: first external call timed out)
    If key set fails after DB write: retry hits DB → gets cached response
```

---

## Problem Statement

Idempotency implementation patterns for payments. Key design, storage, retry handling, distributed idempotency, and database patterns.

---

## Idempotency Store Patterns

### Pattern 1: Redis with SET NX (Fast; Ephemeral)
```java
public PaymentResponse createPayment(PaymentRequest req) {
    String key = "idem:" + req.getIdempotencyKey();
    
    // Check: is there already a result for this key?
    String cached = redis.get(key);
    if (cached != null) {
        return deserialize(cached, PaymentResponse.class);
    }
    
    // Acquire lock (prevent concurrent processing of same key)
    // SET NX: only set if key doesn't exist
    // EX 30: lock expires in 30 seconds (prevents deadlock if processing crashes)
    Boolean lockAcquired = redis.setIfAbsent("lock:" + key, "processing", Duration.ofSeconds(30));
    
    if (!lockAcquired) {
        // Another request is processing this key right now
        // Wait and poll (or return 202 Accepted with retry hint)
        return waitForResult(key, timeout = Duration.ofSeconds(10));
    }
    
    try {
        // Process the payment (may call external card network)
        PaymentResponse response = paymentProcessor.process(req);
        
        // Store result AFTER processing
        redis.set(key, serialize(response), Duration.ofHours(24));
        return response;
        
    } finally {
        // Release lock (result is now stored; future requests will hit cache)
        redis.delete("lock:" + key);
    }
}

// Key insight: SET NX for the LOCK, not for the result directly
// Lock prevents concurrent duplicate processing
// Result stored after processing → future retries return cached result
```

### Pattern 2: PostgreSQL UNIQUE Constraint (Durable; Correct)
```sql
-- Idempotency store in PostgreSQL (durable; survives Redis restart)
CREATE TABLE idempotency_keys (
  idempotency_key  VARCHAR(128)  PRIMARY KEY,
  request_hash     VARCHAR(64)   NOT NULL,   -- hash of request body (detect conflicts)
  response_body    JSONB,                    -- cached response
  response_status  INTEGER,                 -- HTTP status code
  payment_id       UUID,                    -- link to created payment (if successful)
  status           VARCHAR(16)   DEFAULT 'PROCESSING',  -- PROCESSING / COMPLETED / FAILED
  created_at       TIMESTAMP     DEFAULT NOW(),
  expires_at       TIMESTAMP     DEFAULT NOW() + INTERVAL '24 hours'
);
CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
-- Cleanup job: DELETE FROM idempotency_keys WHERE expires_at < NOW()
```

```java
@Transactional
public PaymentResponse createPayment(PaymentRequest req) {
    String idemKey = req.getIdempotencyKey();
    String requestHash = hash(req.toCanonicalJson());
    
    // Check for existing record
    Optional<IdempotencyRecord> existing = idemRepo.findById(idemKey);
    
    if (existing.isPresent()) {
        IdempotencyRecord record = existing.get();
        
        // Conflict check: same key but different request body
        if (!record.getRequestHash().equals(requestHash)) {
            throw new IdempotencyConflictException(
                "Idempotency key reused with different request body");
        }
        
        if (record.getStatus().equals("COMPLETED")) {
            return deserialize(record.getResponseBody(), PaymentResponse.class);
        }
        
        if (record.getStatus().equals("PROCESSING")) {
            // In-flight: wait for result or return 202
            return pollForCompletion(idemKey, timeout = 10);
        }
    }
    
    // New key: insert PROCESSING record
    try {
        idemRepo.save(new IdempotencyRecord(idemKey, requestHash, "PROCESSING"));
    } catch (DataIntegrityViolationException e) {
        // Race: another request inserted the same key concurrently
        return createPayment(req);  // retry; will hit the existing record
    }
    
    try {
        // Process (may call external systems)
        PaymentResponse response = paymentProcessor.process(req);
        
        // Update idempotency record with result (in same DB transaction as payment)
        idemRepo.update(idemKey, "COMPLETED", serialize(response), response.getPaymentId());
        
        return response;
        
    } catch (Exception e) {
        idemRepo.update(idemKey, "FAILED", errorResponse(e), null);
        throw e;
    }
}
```

### Pattern 3: Distributed Idempotency Across Services
```
Scenario: payment-service calls ledger-service; both need idempotency

payment-service receives: { idempotency_key: "pay-order-123" }
  → payment-service deduplicates via its idempotency store
  → payment-service calls ledger-service to record ledger entry

ledger-service: needs its own idempotency key
  Cannot reuse payment's idempotency_key (different namespace, different store)
  
  Pattern: derive child key from parent key
    ledger_idempotency_key = hash("ledger:" + parent_idempotency_key + ":" + entry_type)
    = hash("ledger:pay-order-123:DEBIT")
  
  This ensures:
    Same payment retry → same ledger key → ledger deduplicates correctly
    Different payments → different keys → no collision
    Key is deterministic: no state needed between service calls

Example across the payment saga:
  Parent key: "pay-order-123" (from merchant)
  
  payment-service:    idem key = "pay-order-123"
  fraud-service:      idem key = hash("fraud:pay-order-123")
  card-network-call:  idem key = "pay-order-123" (card network has their own dedup)
  ledger-service:     idem key = hash("ledger:pay-order-123:DEBIT")
                      idem key = hash("ledger:pay-order-123:CREDIT")
  webhook-delivery:   idem key = hash("webhook:pay-order-123:CAPTURED")
  
  All are deterministic from the original "pay-order-123"
  All are safe to retry independently
```

---

## Failure Scenarios and Correct Handling

```
Scenario A: Network timeout BEFORE server processes
  Timeline: Client → [timeout at T+5s] → Server still hasn't processed (server is slow)
  Client retries with same key → Server processes first call AND second call
  
  Correct handling: idempotency lock (PROCESSING state prevents second concurrent process)
  If first call completes while second is waiting: second returns first's result
  If first call crashes: lock expires → second call processes → result stored

Scenario B: Partial processing (DB write succeeded; external call failed)
  Timeline: payment-service writes to DB → calls card network → card network times out
  payment-service: what do we do?
  
  Correct: retry the card network call with SAME idempotency key
  Card network: if it received and processed first call → returns same result
  payment-service: stores result → returns to client
  
  Key principle: retrying external call with same idempotency key is safe (idempotent)

Scenario C: Conflicting idempotency key (different request, same key)
  Client 1: key="pay-123", amount=500
  Client 2: key="pay-123", amount=1000 (bug in client: reused key with different amount)
  
  Correct handling: detect conflict via request hash comparison
    If hashes differ: return 409 Conflict (not 200)
    Never silently process the second request as if it were the first

Scenario D: Idempotency key TTL expired; genuine retry after 25 hours
  Client retried after 25 hours (Redis key expired; DB record past TTL)
  
  Options:
    Option 1: Process as new request (key expired → fresh start)
    Option 2: Reject (expired; client must use new key)
  
  For payments: REJECT with 422 "Idempotency key expired; use a new key for a new payment"
  Rationale: a retry after 25 hours is likely a duplicate payment attempt, not a network retry
  The original request either succeeded or failed long ago; the customer should know
```

---

## Idempotent Consumer Pattern (Kafka)

```java
// Kafka consumer: process payment event idempotently
@KafkaListener(topics = "payment.events")
@Transactional
public void handlePaymentEvent(PaymentEvent event) {
    String eventId = event.getEventId();  // globally unique event ID
    
    // Check if this event was already processed
    if (processedEventRepo.existsById(eventId)) {
        log.info("Event {} already processed; skipping", eventId);
        return;  // idempotent: same event processed again = no-op
    }
    
    // Process the event
    updatePaymentStatus(event);
    updateFraudFeatureStore(event);
    triggerWebhook(event);
    
    // Mark as processed (in same transaction as above)
    processedEventRepo.save(new ProcessedEvent(eventId, Instant.now()));
    
    // Kafka offset committed after transaction commits
    // If transaction fails: offset NOT committed → event redelivered → idempotency check catches it
}

// Processed events store:
CREATE TABLE processed_events (
    event_id    VARCHAR(64) PRIMARY KEY,
    processed_at TIMESTAMP  DEFAULT NOW()
);
-- TTL: cleanup events older than 7 days (Kafka won't redeliver after retention period)
```

---

## Observability
```
Idempotency metrics:
  idempotency_hit_rate               (% of requests that hit idempotency cache → indicates retry volume)
  idempotency_conflict_rate          (% rejected due to key reuse with different body → client bug)
  idempotency_lock_wait_time         (time waiting for in-flight duplicate to complete)
  idempotency_store_size             (Redis key count; DB row count)
  
Alert thresholds:
  hit_rate > 10%: unusual retry volume; possible client bug or cascading retry
  conflict_rate > 0.1%: client is reusing keys incorrectly; contact merchant
  lock_wait > 5s: slow payment processing causing concurrent retry waits
```

---

## Architecture Review Checklist
```
□ Idempotency key is caller-generated (not server-assigned):
  Server cannot generate: client times out before server responds → client has no key to retry with

□ Result stored AFTER processing (not before):
  Store key before external call: if external call fails, key is set but result is wrong
  Store key after: if storage fails, key not set → retry makes fresh external call → safe

□ Conflict detection (different request body, same key):
  Hash request body on receipt
  If existing record has different hash → 409 Conflict (never silently process)

□ Distributed idempotency keys:
  Parent key → deterministic child keys via hash
  Each service manages its own idempotency store
  Child keys derived from parent → retries propagate correctly

□ TTL appropriate for retry window:
  24h for normal API retries (sufficient for most network issues)
  7d for batch jobs (batch may retry days later)
  30d for critical financial operations (audit + dispute window)

□ SCC LENS:
  STATE: Redis (fast idempotency cache; ephemeral) + PostgreSQL (durable record; authoritative)
  COORDINATION: SET NX lock (prevents concurrent duplicate processing of same key)
  CONCENTRATION: idempotency store is hot path (every payment reads/writes it) → Redis cluster
```

---

*Next: Case Study 37 — Caching Strategies (Advanced)*
