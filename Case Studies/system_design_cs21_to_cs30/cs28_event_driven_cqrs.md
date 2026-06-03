# Case Study 28 — Event-Driven Architecture with CQRS and Event Sourcing

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Command vs Event distinction, Event Store vs DB, Eventual consistency trade-offs.

---

## Business Context

A payment platform with multiple bounded contexts (payments, merchants, users, fraud, analytics) starts as a monolith but evolves toward microservices. The communication pattern between these services determines whether the system is brittle (sync RPC) or resilient (event-driven). Event Sourcing makes the payment ledger the single source of truth. CQRS separates write (command) from read (query) to scale them independently.

**Business goals driving architecture:**
- Services communicate without coupling (event-driven decouples producers from consumers)
- Complete audit trail without extra work (event sourcing: state IS the event log)
- Read models optimised per use case (CQRS: payment history != fraud analysis)
- Business process as explicit flow (saga = sequence of events + compensating transactions)

---

## Recognition Framework

### CQRS vs Simple CRUD

```
Simple CRUD:
  One model for reads AND writes
  Same table, same endpoints: GET /payments and POST /payments hit same DB
  Works well when: read and write patterns are similar; low volume
  
  Problem at scale:
    Write: INSERT payment; update ledger; trigger webhook; notify fraud
    Read: complex JOIN across payments + merchants + users for dashboard
    These have completely different shapes; trying to optimise one hurts the other
    Aggregate (COUNT, GROUP BY) for analytics runs on same tables as transactional writes

CQRS (Command Query Responsibility Segregation):
  Write model (Command side): optimised for writes and consistency
    Commands: CreatePayment, CapturePayment, RefundPayment
    Storage: normalised, ACID-compliant (PostgreSQL or event store)
  
  Read model (Query side): optimised for reads and query patterns
    Read models (projections) per use case:
      PaymentList: denormalized for merchant dashboard (includes merchant_name)
      FraudView: payment + device + velocity features (for real-time scoring)
      AnalyticsView: aggregated, pre-computed (for reporting)
    Storage: ClickHouse, Elasticsearch, Redis — whatever fits the query pattern
  
  How read models stay fresh:
    Command side emits domain events on every state change
    Projectors (event consumers) update read models from events
    Eventually consistent (< 1s lag typically)
```

### Event Sourcing — State as Event Log

```
Traditional: store current state
  payments table: { payment_id, status, amount, merchant_id, ... }
  On capture: UPDATE payments SET status = 'CAPTURED'
  History: gone (you only see current state)

Event Sourcing: store the events that led to current state
  payment_events table (append-only):
    { PAYMENT_CREATED, amount=500, merchant=X, timestamp=T1 }
    { PAYMENT_AUTHORISED, network_txn=VISA123, timestamp=T2 }
    { PAYMENT_CAPTURED, captured_amount=500, timestamp=T3 }
  
  Current state = replay events from beginning
    payment.status = last event's resulting status
    payment.amount = from PAYMENT_CREATED event
  
  Performance: don't replay 1M events on every read → snapshots
    Snapshot every 100 events: store current state
    On read: load latest snapshot + replay events since snapshot

Why Event Sourcing for payments?
  1. Complete audit trail without extra work (the event log IS the audit trail)
  2. Temporal queries: "what was this payment's status at T=2024-01-15 12:00?"
  3. Bug fix + replay: fix a projection bug → replay all events → correct read model
  4. Debug: "what sequence of events led to this FAILED payment?" → read the log
  5. Regulatory: RBI requires complete transaction history → event store is that history
```

---

## Problem Statement

Refactor payment platform to event-driven architecture with CQRS and Event Sourcing. Domain events drive service communication and maintain an audit-complete event store.

---

## Functional Requirements
- Commands: CreatePayment, CapturePayment, RefundPayment, VoidPayment
- Events: PaymentCreated, PaymentAuthorised, PaymentCaptured, PaymentFailed, PaymentRefunded
- Event store: append-only; queryable by aggregate_id + sequence
- Read models: PaymentList (dashboard), FraudView (real-time scoring), AnalyticsView (BI)
- Event replay: replay all events for any aggregate from beginning
- Saga: multi-step payment flow orchestrated by events (not RPC)

## Non-Functional Requirements
- Command processing: **< 100ms** (write path)
- Event propagation: **< 1 second** from event to read model update
- Event replay: **100M events replayable within 1 hour**
- Read model queries: **< 10ms** (pre-projected; not computed on query)

---

## Data Model

```sql
-- Event Store (append-only; the single source of truth for aggregate state)
CREATE TABLE payment_events (
  event_id         UUID         DEFAULT gen_random_uuid(),
  aggregate_id     UUID         NOT NULL,   -- payment_id
  aggregate_type   VARCHAR(32)  NOT NULL,   -- "Payment"
  event_type       VARCHAR(64)  NOT NULL,   -- PaymentCreated, PaymentCaptured, etc.
  sequence_num     BIGINT       NOT NULL,   -- per-aggregate sequence; for ordering + optimistic lock
  payload          JSONB        NOT NULL,   -- event data (varies by event_type)
  metadata         JSONB,                   -- correlation_id, causation_id, user_id
  occurred_at      TIMESTAMP    DEFAULT NOW(),
  PRIMARY KEY (aggregate_id, sequence_num),  -- natural PK: one aggregate, sequential
  UNIQUE (event_id)
);

-- Concurrency control: before appending event N, check last event = N-1
-- If not: another command was processed concurrently → conflict → retry command

CREATE INDEX idx_events_aggregate ON payment_events(aggregate_id, sequence_num);
CREATE INDEX idx_events_type ON payment_events(event_type, occurred_at DESC);

-- Snapshots (performance optimisation for large aggregates)
CREATE TABLE payment_snapshots (
  aggregate_id     UUID         PRIMARY KEY,
  snapshot_version BIGINT       NOT NULL,   -- sequence_num at snapshot time
  state            JSONB        NOT NULL,   -- serialised current state
  created_at       TIMESTAMP    DEFAULT NOW()
);
```

```
Kafka topics (one per event type or bounded context):
  payment.events                → all payment domain events; consumed by all projectors
  payment.commands              → commands submitted to command handler (FIFO per payment_id)
  
Read models (separate stores per use case):
  payment_list (PostgreSQL):    denormalised; for merchant dashboard queries
  fraud_view (Redis):           hot features; for real-time fraud scoring (< 2ms read)
  analytics_view (ClickHouse):  aggregated; for business intelligence

Event schema (example - PaymentCaptured):
  {
    "event_id": "uuid",
    "aggregate_id": "pay-123",
    "event_type": "PaymentCaptured",
    "sequence_num": 3,
    "payload": {
      "captured_amount": 50000,
      "captured_at": "2024-01-01T14:00:00Z",
      "network_txn_id": "VISA-123"
    },
    "metadata": {
      "correlation_id": "req-456",    // trace the originating HTTP request
      "causation_id": "cmd-789",      // the command that caused this event
      "triggered_by": "payment-service"
    }
  }
```

---

## API Design

```
Command API (write side):
  POST /api/v1/payments/commands/create
  Body: { command_id: "cmd-uuid", amount, currency, card_token, merchant_id }
  Response 202: { command_id, expected_event_type: "PaymentCreated", correlation_id }
  (202 = command accepted; event will be published when processed)
  
  POST /api/v1/payments/{payment_id}/commands/capture
  Body: { command_id: "cmd-uuid", amount }
  Response 202

  POST /api/v1/payments/{payment_id}/commands/refund
  Body: { command_id: "cmd-uuid", amount, reason }
  Response 202

Query API (read side; from projected read models):
  GET /api/v1/payments/{payment_id}         → from payment_list read model
  GET /api/v1/payments/{payment_id}/events  → event store (audit trail)
  GET /api/v1/merchants/{id}/payments       → merchant dashboard view
  GET /api/v1/payments/{payment_id}/history → temporal history from event store

Event subscription (for integrations):
  Webhooks: merchant subscribes to PaymentCaptured, PaymentRefunded
  WebSocket: real-time updates for payment status page
  Kafka consumer: internal services subscribe to domain events
```

---

## High-Level Architecture

```
COMMAND SIDE (write path):

Client → Command API
  │
  ├── 1. Deserialise and validate command
  ├── 2. Load aggregate from event store:
  │       a. Check snapshot: SELECT * FROM payment_snapshots WHERE aggregate_id = ?
  │       b. Load events since snapshot: SELECT * FROM payment_events
  │             WHERE aggregate_id = ? AND sequence_num > snapshot_version ORDER BY sequence_num
  │       c. Apply events to reconstruct current state (in-memory aggregate)
  │
  ├── 3. Apply command to aggregate:
  │       payment.capture(amount) → produces [PaymentCaptured] event
  │       Validation: cannot capture a VOIDED payment → DomainException
  │
  ├── 4. Persist events (optimistic concurrency):
  │       INSERT INTO payment_events (sequence_num = aggregate.version + 1)
  │       ON CONFLICT: another command was processed concurrently → retry
  │
  ├── 5. Publish events to Kafka:
  │       Kafka producer: payment.events topic
  │       Outbox pattern (Case 21): atomic event + DB write → no lost events
  │
  └── 6. Return 202 Accepted (command processed; read model updates async)

QUERY SIDE (read path; separate from command path):

Client → Query API
  │
  └── Route to correct read model based on query:
        Dashboard (merchant_id + date range) → payment_list (PostgreSQL)
        Fraud features (payment_id) → fraud_view (Redis hash)
        Analytics (aggregations) → analytics_view (ClickHouse)
        Event audit trail → event store (PostgreSQL)
        → Each read model is pre-projected; no join-time computation

PROJECTORS (event-driven read model updates):

Kafka consumer group: payment-projectors
  PaymentListProjector:
    On PaymentCreated: INSERT into payment_list (denormalized row)
    On PaymentCaptured: UPDATE payment_list SET status = 'CAPTURED', captured_at = ...
    On PaymentRefunded: UPDATE payment_list SET status = 'REFUNDED', refunded_amount = ...

  FraudViewProjector:
    On PaymentCreated: HSET fraud_view:{payment_id} {amount, merchant, device, ...} EX 3600
    On PaymentFailed: delete from fraud_view (no longer relevant)

  AnalyticsProjector (same as CS18 analytics pipeline):
    Flink streaming aggregation on payment events
    Writes to ClickHouse hourly metrics
```

---

## Detailed Components

### Payment Aggregate
```java
public class Payment {
    private UUID id;
    private Money amount;
    private PaymentStatus status;
    private String networkTxnId;
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();
    private long version;  // current sequence number

    // Command: process capture request
    public void capture(Money captureAmount) {
        // Business rule validation (on current state)
        if (this.status != AUTHORISED) {
            throw new DomainException("Cannot capture payment in state: " + status);
        }
        if (captureAmount.isGreaterThan(this.amount)) {
            throw new DomainException("Cannot capture more than authorised amount");
        }

        // Produce event (does NOT modify state directly)
        PaymentCaptured event = new PaymentCaptured(id, captureAmount, Instant.now());
        apply(event);             // modifies in-memory state
        uncommittedEvents.add(event);  // queued for persistence
    }

    // Event application: updates in-memory state
    private void apply(PaymentCaptured event) {
        this.status = CAPTURED;
        this.capturedAmount = event.getAmount();
    }

    // Rebuild state from events (called on load from event store)
    public static Payment reconstitute(List<DomainEvent> events) {
        Payment payment = new Payment();
        events.forEach(payment::apply);
        payment.version = events.getLast().getSequenceNum();
        return payment;
    }
}
```

### Optimistic Concurrency on Event Append
```java
public void saveEvents(UUID aggregateId, List<DomainEvent> events, long expectedVersion) {
    try {
        for (int i = 0; i < events.size(); i++) {
            DomainEvent event = events.get(i);
            event.setSequenceNum(expectedVersion + i + 1);

            jdbcTemplate.update(
                "INSERT INTO payment_events (aggregate_id, event_type, sequence_num, payload, metadata) " +
                "VALUES (?, ?, ?, ?::jsonb, ?::jsonb)",
                aggregateId,
                event.getEventType(),
                event.getSequenceNum(),
                toJson(event.getPayload()),
                toJson(event.getMetadata())
            );
            // PRIMARY KEY (aggregate_id, sequence_num) → unique constraint
            // If two commands process same aggregate simultaneously:
            //   Both try to insert sequence_num = 42
            //   One succeeds; other gets UNIQUE violation
        }
    } catch (DuplicateKeyException e) {
        // Optimistic lock failure: another command was processed concurrently
        // Strategy: reload aggregate + retry command (up to 3×)
        throw new ConcurrencyException("Concurrent modification detected; retry command");
    }
}
```

### Saga: Multi-Step Payment Flow via Events
```
Traditional synchronous saga (RPC):
  payment-service → (sync RPC) → fraud-service → (sync RPC) → card-network
  Problem: coupling; if fraud-service is slow → payment-service is slow

Event-driven saga (choreography):
  payment-service: publish PaymentInitiated
  fraud-service: consume PaymentInitiated → evaluate → publish FraudEvaluated (ALLOW/BLOCK)
  payment-service: consume FraudEvaluated
    → if ALLOW: call card network → publish PaymentAuthorised or PaymentFailed
    → if BLOCK: publish PaymentDeclined

  No direct coupling: services only know about events, not each other

Event-driven saga (orchestration — more explicit):
  PaymentSagaOrchestrator:
    State machine: INITIATED → FRAUD_CHECKED → AUTHORISED → CAPTURED
    On PaymentInitiated: send FraudCheckCommand to fraud-service
    On FraudCheckCompleted(ALLOW): send AuthoriseCommand to card network adapter
    On AuthorisationCompleted: send CaptureCommand
    On any failure: publish compensating events (void, refund as needed)

  Orchestration preferred for complex flows (visible state machine; easier to debug)
  Choreography preferred for simple flows (no central coordinator; more resilient)
```

---

## Scaling Strategy
```
Phase 1: Introduce events alongside CRUD (additive)
  Keep existing DB tables; add event publishing on every write
  Events go to Kafka; build first projector (payment_list)
  No event sourcing yet; events are derived from CRUD operations

Phase 2: CQRS without Event Sourcing
  Separate write and read models
  Write: normalised PostgreSQL
  Read: denormalised projections per use case
  Projectors maintain read models from events

Phase 3: Event Sourcing for payments aggregate
  Replace payment update operations with event append
  Load aggregate from event store on each command
  Snapshots for performance on large aggregates
  Existing CRUD becomes read model (migrated via event replay)

Phase 4: Full event-driven microservices
  Each bounded context: payment, merchant, user, fraud has its own event store
  Cross-context communication: events only (no direct DB access)
  Saga orchestrator for complex multi-context flows
```

---

## Reliability Strategy
- **Event store durability**: PostgreSQL with WAL replication; RF=2; events never lost
- **Event idempotency**: projectors check if event already applied (event_id in read model); safe to replay
- **Kafka consumer retry**: failed projection → DLQ; alert; manual replay from event store
- **Snapshot lag**: if snapshot fails → next load replays from oldest event; slower but correct
- **Eventual consistency**: read models < 1s behind; queries that need latest state can read from event store directly (pay read cost)

---

## Observability
```
Event pipeline metrics:
  events_published_total by {event_type}      (command success rate)
  event_consumer_lag by {projector}           (read model freshness)
  projection_latency_p99                      (event → read model update time)
  optimistic_concurrency_conflicts_total      (command retry rate)
  aggregate_load_time_p99                     (snapshot + replay time)
  snapshot_creation_rate                      (background snapshot health)

Business event metrics:
  payment_state_transition_rates              (PaymentCreated → Captured vs → Failed)
  saga_completion_rate                        (successful end-to-end payment flows)
  compensating_transaction_rate              (how often rollback is triggered)
```

---

## Chaos Testing
```
Experiment 1: Concurrent Commands on Same Payment
  Inject:    Two capture commands for same payment submitted simultaneously
  Expected:  One command succeeds (inserts event at sequence N)
  Expected:  Second command: UNIQUE violation on (aggregate_id, sequence_num)
  Expected:  Second command retries → reloads aggregate (sees first capture) → rejects duplicate
  Red flag:  Both captures succeed → payment captured twice

Experiment 2: Projector Failure and Replay
  Inject:    Kill payment_list projector after it has consumed 50% of events for a new payment
  Expected:  Projector restarts; Kafka consumer offset committed after DB write (at-least-once)
  Expected:  Some events replayed; projector is idempotent → same result
  Expected:  Read model eventually consistent; no data corruption
  Red flag:  Read model shows inconsistent state after replay (projector not idempotent)

Experiment 3: Event Store Read at Query Time vs Projection
  Inject:    Artificially delay projectors by 30 seconds
  Expected:  Query API returns stale data from read models (30s behind)
  Expected:  For clients needing fresh state: query event store directly
  Expected:  Monitoring shows event_consumer_lag spike → alert fires
  Red flag:  Queries return errors (not stale data; graceful degradation expected)
```

---

## Monthly Cost Estimate
```
Scale: 500M events/day

Event Store (PostgreSQL):
  Primary + 2 replicas db.r6g.4xlarge ($4,800/mo)   = $4,800/mo

Kafka (event distribution):
  6 × kafka.m5.2xlarge ($300/mo)                     = $1,800/mo

Projectors (multiple consumer groups):
  8 × m5.xlarge ($150/mo)                            = $1,200/mo

Read Model stores (PostgreSQL + Redis + ClickHouse):
  Already costed in respective case studies
  Incremental for CQRS: ~$1,000/mo additional

Saga Orchestrator:
  2 × m5.large ($75/mo)                              = $150/mo
─────────────────────────────────────────────────────────────────
Total: ~$8,950/month
CQRS/ES adds ~$8,950/mo on top of existing infrastructure.
Value: complete audit trail, replay capability, independently scalable read/write.
```

---

## Architecture Review Checklist
```
□ Event Sourcing invariants:
  → Events are immutable facts: never update or delete an event.
  → Commands are requests: they may fail; events are what actually happened.
  → Current state is always derivable from events (+ snapshot for performance).
  → Event schema versioning: backward compatible changes only (new optional fields).

□ CQRS trade-offs:
  → Complexity: two models to maintain; projectors to build and monitor.
  → Eventual consistency: read models lag behind; callers must accept this.
  → Read model proliferation: each use case gets its own model; more stores to operate.
  → Worth it when: read and write patterns are sufficiently different; audit is required.

□ Optimistic concurrency:
  → All commands provide expected_version; event store rejects wrong version.
  → Conflict resolution: retry command (reload state + re-apply); max 3 retries.
  → No pessimistic locks: no SELECT FOR UPDATE on aggregate row.

□ Projector idempotency:
  → Every projector tracks applied event_ids; duplicate events are no-ops.
  → Safe to replay entire event stream; read model converges to same state.

□ SCC LENS:
  STATE: Event store (append-only facts — authoritative), Read models (projections — derived)
  COORDINATION: Optimistic concurrency (sequence_num), Outbox (event + DB atomic), Saga (event choreography)
  CONCENTRATION: Event store is write bottleneck → PostgreSQL sharding by aggregate_id if needed
```

---

*Next: Case Study 29 — Observability Platform (OpenTelemetry, Distributed Tracing)*
