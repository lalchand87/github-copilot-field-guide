# Case Study 39 — Event Streaming vs Message Queuing

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Kafka vs RabbitMQ decision, Consumer groups, Replay vs point-in-time.

---

## Business Context

Every payment platform needs async communication between services. The classic question: should I use Kafka or RabbitMQ (or SQS, or Pulsar)? The answer depends on whether you need event streaming (durable log; replay; multiple consumers) or task queuing (work distribution; once-processed; acknowledgement). Getting this wrong means either over-engineering with Kafka for a simple work queue, or losing replay capability by using RabbitMQ for an event log.

**Business goals driving architecture:**
- Payment events: multiple consumers (fraud, analytics, notification, audit) — each needs their own copy
- Task queues: webhook delivery, email sending — each message processed by exactly one worker
- Replay: reprocess all payment events for a new fraud model — Kafka enables; RabbitMQ does not
- Order: payments within a merchant must be processed in order

---

## Recognition Framework

### Event Streaming vs Task Queuing

```
Task Queue (RabbitMQ / SQS / ActiveMQ):
  Model: message enqueued → ONE consumer picks it up → acks → message deleted
  Consumer group: multiple workers competing for messages (work distribution)
  Retention: message deleted after ack (not persisted)
  Replay: NOT possible (message gone after consumption)
  Ordering: per-queue FIFO (not guaranteed across queues)
  
  Use when:
    Work distribution: N workers process M tasks
    Each task processed ONCE (idempotent or exactly-once needed separately)
    Don't need to replay or reprocess
    Example: send email, process refund batch, thumbnail generation

Event Streaming (Kafka / Kinesis / Pulsar):
  Model: event appended to log → MULTIPLE independent consumer groups read
  Consumer group: each group maintains its own offset (all groups read all events)
  Retention: events kept for configured period (7 days default; forever possible)
  Replay: new consumer starts from offset 0 → processes all historical events
  Ordering: guaranteed within a partition (by key)
  
  Use when:
    Multiple consumers need the same event (fan-out)
    Need to replay history (new service, bug fix, analytics)
    Event sourcing (events as source of truth)
    High throughput ingestion (Kafka handles millions/sec)
    Example: payment events → fraud + analytics + audit + notifications all consume

Decision framework:
  "Will more than one service consume this?"   → Kafka
  "Do I ever need to reprocess this message?"  → Kafka
  "Is this a task for a pool of workers?"      → Queue (RabbitMQ/SQS)
  "Will I add new consumers in the future?"    → Kafka
  "Do I need strict FIFO globally?"            → Queue (Kafka only guarantees per-partition)
```

---

## Problem Statement

Messaging architecture for a payment platform. Kafka for payment events (fan-out, replay, ordering). RabbitMQ / SQS for task queues (webhook delivery, email). Decision criteria and patterns.

---

## Architecture

```
Payment event flow (Kafka):
  payment-service → Kafka topic "payment.events" → {
    fraud-service consumer group
    analytics-service consumer group
    notification-service consumer group
    audit-service consumer group
    ledger-service consumer group
  }
  
  Each consumer group: independent offset; independent processing speed
  fraud-service: must be fast (< 100ms lag); high-priority consumer group
  analytics-service: can lag (1-5 min is fine); batch consumer
  
  All five get the SAME PaymentCaptured event from the same Kafka topic
  Kafka retention: 7 days (replay window for new consumers or bug fixes)

Task queue flow (SQS or RabbitMQ):
  notification-service (Kafka consumer) → processes event → enqueues webhook task
  Webhook delivery workers: pull from SQS queue → HTTP POST to merchant → ack → delete
  
  Why SQS here, not just more Kafka?
    Webhook delivery: each task has independent retry schedule (exponential backoff)
    Kafka consumer groups can't have per-message retry delay natively
    SQS: visibility timeout per message; dead-letter queue after N attempts
    RabbitMQ: dead letter exchange; per-message TTL and retry scheduling
    This is exactly what SQS/RabbitMQ does better than Kafka for task queues
```

---

## Kafka Patterns for Payment Platform

### Topic Design
```
Topic per event type (preferred for payment platform):
  payment.events                    → PaymentCreated, Authorised, Captured, Failed, Refunded
  fraud.decisions                   → FraudAllowed, FraudBlocked, FraudReview
  user.events                       → UserRegistered, KYCCompleted, ProfileUpdated
  merchant.events                   → MerchantOnboarded, ConfigUpdated
  
  Advantages: schema per topic; consumer reads only relevant events
  
Topic per bounded context (alternative):
  payments.domain.events            → all payment-related events
  users.domain.events               → all user-related events
  
  Advantage: simpler; fewer topics
  Disadvantage: consumer reads all events even if only interested in one type

Topic configuration for payments:
  partitions: 32                    (parallelism; 1 consumer per partition per group)
  replication-factor: 3             (survive 2 broker failures)
  min.insync.replicas: 2            (acks=all requires 2 acknowledgements)
  retention.ms: 604800000           (7 days = 7 × 24 × 3600 × 1000)
  retention.bytes: -1               (unlimited by size; only by time)
  compression.type: lz4             (fast compression; reduces storage 3-5×)
```

### Consumer Group Lag Management
```java
// Consumer group monitoring: lag is the key metric
// Lag = (latest Kafka offset) - (consumer's current offset)
// Lag = 0: consumer is up-to-date
// Lag growing: consumer is falling behind (scaling needed)

// Priority consumer (fraud-service): must stay near-zero lag
@KafkaListener(
    topics = "payment.events",
    groupId = "fraud-service-group",
    containerFactory = "highPriorityKafkaListenerFactory"
)
public void handlePaymentEventForFraud(PaymentEvent event) {
    // Must process in < 100ms (fraud decision latency SLO)
    fraudEngine.evaluate(event);
}

// Analytics consumer: lag of minutes is acceptable
@KafkaListener(
    topics = "payment.events",
    groupId = "analytics-group",
    containerFactory = "batchKafkaListenerFactory"  // batch consume 1000 at a time
)
public void handleBatchForAnalytics(List<PaymentEvent> events) {
    analyticsAggregator.processBatch(events);  // efficient batch insert to ClickHouse
}

// Scale based on lag:
// fraud-service consumer lag > 1000 → add consumers (up to partition count)
// analytics consumer lag > 100K → scale analytics consumers
```

### Exactly-Once Semantics in Kafka
```java
// Kafka transactions: consume-process-produce atomically
// Use case: consume payment event → compute → produce fraud decision
// Must not produce fraud decision twice on retry

@Transactional("kafkaTransactionManager")
public void processPaymentEvent(ConsumerRecord<String, PaymentEvent> record) {
    // Within this transaction:
    // 1. Consume offset committed
    // 2. Process logic
    // 3. Produce to fraud.decisions topic
    // ALL or NOTHING; if any step fails, transaction rolls back
    
    PaymentEvent event = record.value();
    FraudDecision decision = fraudEngine.evaluate(event);
    
    // Produce to fraud.decisions (transactional)
    kafkaTemplate.send("fraud.decisions", event.getPaymentId(), decision);
    
    // Offset commit happens atomically with the send above
}

// Producer config for exactly-once:
// enable.idempotence=true         (dedup at Kafka level)
// acks=all                        (all ISR must acknowledge)
// transactional.id=fraud-service-1 (unique per producer instance)
```

---

## SQS/RabbitMQ Patterns for Task Queues

### Webhook Delivery with Dead Letter Queue
```
SQS queue: webhook-delivery
  VisibilityTimeout: 30s (task invisible while worker processes it)
  MaxReceiveCount: 5 (after 5 failed attempts → DLQ)
  Dead Letter Queue: webhook-delivery-dlq
  
Worker picks up task:
  1. Message becomes invisible (visibility timeout)
  2. Worker: POST to merchant webhook URL
  3. Success: DELETE message from queue
  4. Failure (connection error): don't delete → message reappears after 30s → retry
  5. After 5 failures: message moves to DLQ
  6. DLQ: alert ops team; investigate; manually replay if needed

Exponential backoff with SQS:
  Standard retry: message reappears at VisibilityTimeout (30s)
  Exponential: worker explicitly sets VisibilityTimeout before each attempt:
    Attempt 1: 30s → visible if not deleted in 30s
    Attempt 2: worker sets visibility = 60s (ChangeMessageVisibility API)
    Attempt 3: 120s; Attempt 4: 300s; Attempt 5: 600s
    Gives merchant server time to recover between retries
```

### RabbitMQ: Complex Routing (Exchanges)
```
RabbitMQ advantages over SQS: flexible routing
  Direct exchange: route by routing key (queue name)
  Topic exchange: route by pattern (payment.# matches payment.captured, payment.failed)
  Fanout exchange: broadcast to all bound queues
  Dead letter exchange: failed messages routed to DLQ with metadata

Payment notification routing:
  Producer → exchange: payment-events (topic exchange)
    routing key: payment.captured → notification queue + webhook queue
    routing key: payment.failed   → notification queue only (no webhook for failures)
    routing key: payment.refunded → notification + webhook + accounting queues
  
  This routing: impossible with Kafka (Kafka doesn't have routing keys)
  In Kafka equivalent: consumer decides whether to process based on event type
  In RabbitMQ: broker routes to correct queue
```

---

## Comparison Summary

| Dimension | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Retention | Days/forever | Until consumed | 4–14 days |
| Replay | Yes (offset seek) | No | No |
| Fan-out | Yes (consumer groups) | Yes (fanout exchange) | No (one consumer) |
| Ordering | Per-partition | Per-queue | FIFO queues only |
| Throughput | Millions/sec | Tens of thousands/sec | Thousands/sec |
| Retry scheduling | No (manual) | Yes (per-message TTL) | Yes (visibility timeout) |
| Managed | MSK, Confluent | CloudAMQP | AWS SQS |
| Complexity | High | Medium | Low |

---

## Observability
```
Kafka metrics:
  kafka_consumer_lag_sum by {group, topic}       (alert if fraud-service lag > 1000)
  kafka_producer_request_rate                    (producing health)
  kafka_topic_partition_offset_lag               (per-partition lag detail)
  
SQS metrics:
  ApproximateNumberOfMessagesVisible             (queue depth; alert > threshold)
  ApproximateAgeOfOldestMessage                 (how old is oldest unprocessed message)
  NumberOfMessagesSentToDLQ                     (alert if DLQ growing)
  
RabbitMQ metrics:
  rabbitmq_queue_messages_ready                  (unprocessed; alert > threshold)
  rabbitmq_queue_messages_unacknowledged         (in-flight; alert if growing)
```

---

## Architecture Review Checklist
```
□ Kafka for: multiple consumers, replay needed, event log
□ Queue for: single consumer per message, task distribution, retry scheduling

□ Kafka consumer group discipline:
  → Each logical service has exactly one consumer group
  → Consumer group ID in code, not configuration (prevents accidental reset)
  → Monitor lag per group; scale when lag grows

□ Ordering requirements:
  → "All events for merchant X in order": partition by merchant_id
  → "All payments from user U in order": partition by user_id
  → Ordering across partitions: NOT guaranteed (by design)

□ Exactly-once vs at-least-once:
  → Kafka transactions (exactly-once) add complexity and reduce throughput
  → Prefer: at-least-once + idempotent consumer (Case 36 pattern)
  → Exactly-once Kafka: only for financial aggregations or state machines

□ SCC LENS:
  STATE: Kafka log (append-only, durable), SQS queue (transient tasks)
  COORDINATION: consumer groups (offset management per group), visibility timeouts (task locking)
  CONCENTRATION: Kafka broker is the hot write path → RF=3 + min.insync.replicas=2
```

---

*Next: Case Study 40 — Staff Engineer Leadership and Incident Management*
