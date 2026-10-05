# Curriculum · Messaging

[← System Design index](../README.md)

> 8 lessons in **Messaging**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Messaging** (8): [Work queues](#work-queues) · [Publish-subscribe](#publish-subscribe) · [Partitioned event logs](#partitioned-event-logs) · [Consumer groups and rebalancing](#consumer-groups-and-rebalancing) · [Delivery semantics](#delivery-semantics) · [Dead-letter queues](#dead-letter-queues) · [Transactional outbox](#transactional-outbox) · [Schema evolution for events](#schema-evolution-for-events)

## Messaging

### Work queues

*Producers enqueue durable tasks, workers lease and complete them, and every acknowledgement boundary is a decision about duplicates versus loss.*

**Flow:** `Producer` → `Durable queue` → `Worker lease` → `Side effect` → `Acknowledgment`

> **The 30-second version**  
> Producers enqueue durable work; workers lease, process and acknowledge. At-least-once delivery is the practical guarantee, so handlers must be idempotent and dead letters must have an owner.

**The problem**

A user uploads a video and expects a response now, not in four minutes. A payment succeeds and three emails, two webhooks and a ledger update must follow. A batch import arrives with a million rows. In each case the work must happen, but not synchronously, and not at the rate it arrives.

A work queue decouples the arrival of work from its execution: producers write durable task records, workers consume at whatever rate they can sustain, and the queue absorbs the difference. That decoupling is the entire value — and every difficulty that follows comes from the fact that the worker and the queue are now two systems that can fail independently.

> **The unavoidable choice at the acknowledgement boundary**  
> **Ack before doing the work**: if the worker crashes mid-task, the work is lost. **Ack after doing the work**: if the worker crashes after the side effect but before the ack, the task is redelivered and the side effect happens twice. There is no third option, which is why **at-least-once delivery plus idempotent handlers** is the standard design.

**Mental model**

A queue is a durable buffer with a lease protocol. Work is not deleted when handed to a worker — it is hidden for a lease period, and only removed when the worker acknowledges. If the lease expires, the work becomes visible again.

1. **Enqueue** — The producer writes a durable record and returns. This must be transactional with whatever caused it, or work is lost or invented (see transactional outbox).
2. **Lease / visibility timeout** — The worker receives the message and it becomes invisible to others for a bounded period. This is the recovery mechanism for crashed workers.
3. **Process** — The worker performs the side effect. This is where idempotency lives.
4. **Acknowledge** — The worker confirms; the message is deleted. Failure to ack before the lease expires means redelivery.
5. **Dead-letter** — After N failed attempts, the message is moved aside rather than retried forever, so one poison message cannot block a queue.

> **The lease duration is a real design parameter**  
> Too short and a slow task is redelivered while still running, producing concurrent duplicate execution. Too long and a crashed worker's task sits invisible for minutes before anyone retries it. Set it from the p99 task duration with headroom, and extend it explicitly (heartbeat) for genuinely long tasks rather than setting a global timeout that suits the slowest one.

**How it works**

**The lease protocol, and where duplicates come from**

```text
worker receives msg, lease 60s
  |
  |-- performs side effect (charges a card)
  |
  X  worker crashes before acknowledging
  |
lease expires at t+60s
  |
another worker receives the SAME message
  |
  |-- performs side effect AGAIN  <- duplicate charge

THE FIX IS NOT IN THE QUEUE
  exactly-once delivery is not achievable over an
  unreliable network. What IS achievable is
  exactly-once EFFECT:

    idempotency_key = msg.id
    if already_processed(idempotency_key): ack and return
    perform_effect()
    record_processed(idempotency_key)   <- same transaction
    ack

Now redelivery is harmless, and at-least-once is enough.
```

1. **Make handlers idempotent, always** — Use the message id or a business key as an idempotency key, and record completion atomically with the effect. This single discipline makes the whole system tractable.
2. **Size the lease from p99 task duration** — With headroom. For long or variable tasks, heartbeat to extend the lease rather than raising the global timeout.
3. **Bound retries and use a dead-letter queue** — A message that fails deterministically — malformed payload, deleted referent — will fail forever. Move it aside after a few attempts so it cannot block or churn.
4. **Apply backoff with jitter between retries** — Immediate retry of a task failing because a dependency is down simply amplifies load against that dependency.
5. **Separate queues by priority and by latency class** — A single queue means a million-row import blocks the password-reset emails behind it. Separate queues, or at least separate consumer pools.
6. **Monitor queue depth and age, not just throughput** — Depth tells you whether you are keeping up; **oldest message age** tells you the worst-case latency a user is experiencing, which throughput alone hides.

**Queue metrics that actually matter**

```text
THROUGHPUT      messages/s processed
                -> tells you nothing about whether you are
                   keeping up

DEPTH           messages waiting
                -> rising depth means arrival > processing

OLDEST AGE      how long the front message has waited
                -> THIS is user-visible latency
                -> a queue with 1M messages and 2s oldest age
                   is healthy; 100 messages and 20 min is not

DRAIN TIME      depth / (capacity - arrival rate)
                -> if arrival >= capacity, it NEVER drains
                -> the number to compute during an incident

DLQ RATE        messages exhausting retries
                -> a rising DLQ rate is a code or data problem,
                   not a capacity problem
```

> **Ordering is not a property of queues in general**  
> Most work queues make no ordering guarantee across messages, and even those that do (partitioned logs) guarantee it only within a partition. If two tasks for the same entity must happen in order, either route them to the same partition, or make the handler order-independent — for example by carrying a version and ignoring stale updates.

**Worked example**

An order-processing pipeline: send confirmation email, update inventory, notify the warehouse, award loyalty points.

**Queue design and the failure analysis**

```text
NAIVE: one queue, one handler doing all four things
  - one slow step (warehouse API) blocks everything
  - a retry re-sends the email that already succeeded
  - a poison message blocks all four effects

BETTER: one message fanned out to four queues
  order.created -> [email] [inventory] [warehouse] [loyalty]
  each with its own handler, retry policy and DLQ

IDEMPOTENCY PER HANDLER
  email:      dedupe on (order_id, "confirmation")
              -> at most one email per order
  inventory:  conditional update WHERE version = n
              -> reapplying is a no-op
  warehouse:  idempotency key on the outbound API call
  loyalty:    dedupe on (order_id, "points")

RETRY AND DLQ POLICY
  email:      5 attempts, exponential backoff, DLQ
  inventory:  10 attempts (must not be lost), DLQ alerts loudly
  warehouse:  5 attempts; DLQ triggers a manual queue
  loyalty:    3 attempts; DLQ is low priority

LATENCY CLASSES
  email and loyalty are not urgent -> shared pool
  inventory is urgent              -> dedicated pool
```

| Metric | Value | Note |
|---|---|---|
| Queues | 4 | independent failure |
| Idempotency | per handler | **own dedupe key** |
| Retry policy | per criticality | not uniform |
| Pools | separated | by latency class |

> **Fan out to separate queues, not to one handler doing everything**  
> A single handler performing four side effects cannot be retried safely: a failure in step three re-runs steps one and two. Splitting into independent messages with independent retry policies means each effect fails, retries and dead-letters on its own, and idempotency is scoped to a single operation rather than to a compound one. This is almost always the right shape.

**When to use it**

- **Decoupling request latency from work duration** — anything the user should not wait for.
- **Absorbing bursts**, where arrival rate exceeds processing capacity temporarily.
- **Retrying work that touches unreliable dependencies**, with the queue providing durability across attempts.
- **Fan-out**, where one event triggers several independent effects.
- **Rate-limiting work against a constrained downstream**, by controlling consumer concurrency.

**When to avoid it**

- **Do not use a queue when the caller needs the result** — that is a synchronous request, possibly with a timeout.
- **Do not use a database table as a queue at high volume** without care: `SELECT ... FOR UPDATE SKIP LOCKED` works, but delete-heavy access patterns punish some storage engines badly.
- **Do not assume ordering** unless the queue explicitly provides it within a partition and you route accordingly.
- **Do not retry indefinitely**; a deterministic failure will never succeed and will churn forever.
- **Do not mix latency classes in one queue**, or a bulk job will delay interactive work.

**Advantages**

- **Producers and consumers scale and fail independently**, which is the core benefit.
- **Durability across worker failure**, since unacknowledged work is redelivered.
- **Natural backpressure and rate limiting** via consumer concurrency.
- **Retry with backoff is built in**, rather than being reimplemented per call site.
- **Bursts are absorbed** rather than rejected, within the bounds of queue capacity and acceptable latency.

**Disadvantages**

- **At-least-once delivery means duplicates**, so every handler must be idempotent.
- **Ordering is weak or partition-scoped**, complicating anything sequence-dependent.
- **Debugging is harder**: the work happens elsewhere, later, possibly several times.
- **Queue depth hides problems** — a system can look healthy while users wait minutes.
- **Another system to operate**, with its own capacity, retention and failure modes.
- **End-to-end latency is unbounded** unless queue age is actively monitored and controlled.

**Trade-offs**

**Delivery guarantees**

| Guarantee | Mechanism | Consequence |
|---|---|---|
| At-most-once | Ack before processing | Work lost on crash; no duplicates |
| At-least-once | Ack after processing | Duplicates on crash; nothing lost |
| Exactly-once delivery | Not achievable over a network | — |
| Exactly-once effect | At-least-once + idempotent handler | The practical target |

Systems that advertise exactly-once are providing exactly-once *processing within their own boundary* — transactional reads and writes inside one system. The moment a side effect leaves that boundary, such as charging a card or calling a partner API, you are back to at-least-once plus idempotency.

> **The sentence that demonstrates understanding**  
> “I'd use at-least-once delivery with an idempotency key derived from the order id, recorded in the same transaction as the effect. That way redelivery after a worker crash is a no-op, and I don't need the queue to promise something it can't deliver.”

**How it fails**

**Work queue failures**

| Symptom | Cause | Fix |
|---|---|---|
| Duplicate side effects | Redelivery after a crash, non-idempotent handler | Idempotency key recorded atomically with the effect |
| Work silently lost | Acking before processing, or an unbounded DLQ nobody watches | Ack after; alert on DLQ depth |
| Queue never drains | Arrival rate ≥ processing capacity | Compute net drain rate; add consumers or shed input |
| One bad message blocks everything | Infinite retry on a poison message | Bounded retries plus a dead-letter queue |
| Concurrent duplicate execution | Lease shorter than task duration | Size lease from p99; heartbeat to extend |
| Retry storm against a failing dependency | Immediate retries with no backoff | Exponential backoff with jitter; circuit breaker |
| Interactive work delayed by bulk jobs | Shared queue across latency classes | Separate queues and consumer pools |
| Healthy dashboards, angry users | Monitoring throughput instead of oldest-message age | Alert on queue age |

**Limits**

> **Operating numbers**
>
> - **Lease duration**: p99 task duration × 2–3, with heartbeat extension for long tasks.
> - **Retry policy**: 3–10 attempts with exponential backoff and jitter, then dead-letter.
> - **Drain time = depth ÷ (capacity − arrival rate)** — if arrivals meet capacity, it never drains.
> - **Alert on oldest-message age**, not depth: a million messages with a two-second age is healthy.
> - **DLQ rate** should be near zero; any sustained rate is a code or data defect, not a capacity issue.

**Alternatives**

| Mechanism | Best for | Trade |
|---|---|---|
| Managed queue (SQS, Pub/Sub) | General async work | At-least-once; weak ordering |
| Partitioned log (Kafka) | Ordered per key; replay; multiple consumers | Consumer group complexity |
| Database table as a queue | Low volume, transactional with your data | Delete-heavy patterns punish some engines |
| Durable workflow engine | Multi-step processes with state | Heavier; another platform |
| Synchronous call with retry | Caller needs the result | Latency coupling; no burst absorption |
| Scheduled batch | Work that is naturally periodic | Latency; burstiness |

**In real systems**

- **SQS with visibility timeouts and dead-letter queues** is the canonical managed implementation, and its semantics — at-least-once, best-effort ordering — are the industry default.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** turns a database table into a work queue without a separate system, which is often the right choice at moderate volume.
- **Celery, Sidekiq and similar libraries** popularised the pattern but historically defaulted to at-most-once or ambiguous semantics, which caused a generation of silently lost jobs.
- **Email and notification pipelines** universally use per-effect queues with per-effect idempotency keys, because duplicate notifications are user-visible.
- **Dead-letter queues with alerting** are how mature systems distinguish transient failure from a code defect, since a rising DLQ rate is never a capacity problem.

**Common mistakes**

- **Acknowledging before performing the work**, silently losing tasks on crash.
- **Non-idempotent handlers** under at-least-once delivery.
- **Unbounded retries** on a deterministically failing message.
- **A lease shorter than the task duration**, producing concurrent duplicate execution.
- **One queue for interactive and bulk work.**
- **Monitoring throughput rather than oldest-message age.**
- **A dead-letter queue nobody owns or watches.**

**The staff-level view**

Queues make failure asynchronous, which means the failures are quieter and the monitoring has to be deliberate.

- **Make idempotency a framework concern, not a per-handler one.** A shared consumer wrapper that checks and records an idempotency key removes the most common and most damaging bug class.
- **Alert on oldest-message age**, because throughput and depth both hide user-visible latency and teams routinely monitor the wrong one.
- **Require a dead-letter queue with an owner and an alert** for every queue. An unwatched DLQ is silent data loss with extra steps.
- **Separate queues by latency class as a standard**, so a bulk import can never delay an interactive path.
- **Push back on compound handlers.** One message performing four side effects cannot be retried safely; four messages with four idempotency keys can.

**Go deeper**

A work queue decouples arrival from execution: producers write durable records, workers lease them for a bounded period, process, and acknowledge. If a worker crashes, the lease expires and the message is redelivered. That redelivery is unavoidable — acknowledging before processing loses work on crash, acknowledging after duplicates it — so the achievable target is not exactly-once delivery but exactly-once *effect*, via at-least-once delivery plus idempotent handlers.

Idempotency means deriving a stable key from the message or business identity and recording its completion atomically with the side effect, so redelivery becomes a no-op. Lease duration must exceed p99 task time or slow tasks get duplicated while still running. Retries need bounded attempts with jittered backoff and a dead-letter queue, because a deterministically failing message will otherwise churn forever.

Two structural choices matter. Fan out to separate queues per effect rather than having one handler do four things, since a compound handler cannot be retried safely and its failure re-runs earlier steps. And separate queues by latency class so a bulk import cannot delay interactive work. Operationally, alert on oldest-message age rather than depth or throughput — a million messages with a two-second age is healthy, while a hundred messages with a twenty-minute age is not.

A work queue's value is decoupling the arrival of work from its execution. Every difficulty in the pattern comes from the fact that the queue and the worker are separate systems that fail independently.

**The acknowledgement boundary is a forced choice.** Acknowledge before processing and a worker crash loses the task silently. Acknowledge after and a crash between the side effect and the acknowledgement causes redelivery and a duplicate effect. Exactly-once delivery across an unreliable network is not achievable, so the industry converged on at-least-once delivery with idempotent handlers, which delivers exactly-once *effect*. Systems advertising exactly-once are describing transactional processing within their own boundary; as soon as a side effect leaves it — a card charge, a partner API call, an email — you are back to at-least-once plus idempotency.

**Idempotency is the load-bearing discipline.** Derive a stable key from the message id or a business identity plus the effect name, and record completion in the same transaction as the effect itself. Atomicity matters: writing the effect and the record separately reintroduces the crash window you were closing. For external effects, pass the same key as an idempotency key on the outbound call so the remote system deduplicates as well. Because this is the highest-consequence and least reliably remembered discipline in async systems, it belongs in a shared consumer wrapper rather than in each handler.

**Leases, retries and poison messages.** The lease must exceed p99 task duration with headroom, or slow tasks get redelivered while still executing and run concurrently with themselves; long or variable tasks should heartbeat to extend rather than forcing a global timeout suited to the worst case. Retries need exponential backoff with jitter, because immediate retry against a failing dependency amplifies load at exactly the wrong moment. And they must be bounded: a message failing deterministically — malformed payload, deleted referent — will never succeed, so after a few attempts it belongs in a dead-letter queue with a named owner and an alert. A rising dead-letter rate is always a code or data defect, never a capacity problem.

**Shape the pipeline as fan-out, not compound handlers.** One handler performing four side effects cannot be retried safely, because a failure in the third step re-runs the first two and idempotency must then be scoped to a compound operation. Publishing one event to four independent queues gives each effect its own retry policy, its own dead-letter queue and its own single-purpose idempotency key. Separating consumer pools by latency class completes it, so a bulk import cannot sit in front of a password-reset email.

**Monitoring the right quantity.** Throughput tells you nothing about whether you are keeping up. Depth tells you the backlog but not the experience. Oldest-message age is the user-visible latency and is the metric to alert on — a million messages with a two-second age is a healthy high-throughput queue, while a hundred messages with a twenty-minute age means someone has been waiting twenty minutes. During an incident, the derived number is drain time: depth divided by capacity minus arrival rate, which is infinite whenever arrivals meet capacity. And end-to-end tracing must span the enqueue and the dequeue, or debugging an asynchronous failure becomes archaeology across two systems separated by an unknown delay.

**Prove it — interview questions**

1. **[Basic] Why do work queues deliver at least once rather than exactly once?**

   <details><summary>Model answer</summary>

   Because the worker and the queue can fail independently, and the acknowledgement is a separate step from the work. If you acknowledge before processing, a crash loses the task. If you acknowledge after, a crash between the side effect and the acknowledgement causes redelivery. Exactly-once delivery over an unreliable network is not achievable, so the practical target is exactly-once *effect*: at-least-once delivery plus an idempotent handler.

   </details>

2. **[Basic] What is a visibility timeout or lease?**

   <details><summary>Model answer</summary>

   When a worker receives a message, it is not deleted — it becomes invisible to other workers for a bounded period. If the worker acknowledges within that period, the message is removed; if the worker crashes, the lease expires and the message becomes visible again for another worker. It is the recovery mechanism for worker failure, and its duration must exceed the task's p99 processing time or slow tasks get redelivered while still running.

   </details>

3. **[Senior] How do you make a handler idempotent?**

   <details><summary>Model answer</summary>

   Derive a stable key — the message id, or a business key like order id plus effect name — and record its completion in the same transaction as the side effect. On redelivery, the handler checks whether that key is already recorded and acknowledges without re-executing. The critical detail is atomicity: if the effect and the record are written separately, a crash between them reintroduces the problem. For effects in external systems, pass the same key as an idempotency key on the outbound API call so the remote side deduplicates too.

   </details>

4. **[Senior] Why alert on queue age rather than queue depth?**

   <details><summary>Model answer</summary>

   Because depth says nothing about user experience. A queue with a million messages and a two-second oldest-message age is perfectly healthy — it is simply high throughput. A queue with a hundred messages and a twenty-minute oldest age means someone has been waiting twenty minutes. Age is the user-visible latency; depth and throughput are both compatible with both healthy and broken states. During an incident the derived number that matters is drain time, which is depth divided by capacity minus arrival rate — and if arrivals meet capacity, it never drains at all.

   </details>

5. **[Staff] Design the async pipeline for order processing with four downstream effects.**

   <details><summary>Model answer</summary>

   I would publish one `order.created` event fanned out to four independent queues rather than having one handler perform all four effects, because a compound handler cannot be retried safely — a failure in the third step re-runs the first two. Each handler gets its own idempotency key scoped to that single effect, its own retry policy matched to criticality, and its own dead-letter queue with an owner. Inventory would get more attempts and a loud DLQ alert because losing it is a correctness problem; loyalty points would get fewer and a low-priority DLQ. I would also separate consumer pools by latency class so a bulk reconciliation job cannot delay customer-facing confirmations. And the publish itself must be transactional with the order write — otherwise a crash between them either loses the event or emits one for an order that does not exist, which is what the transactional outbox pattern solves.

   </details>

6. **[Principal] What makes asynchronous systems harder to operate, and how do you compensate?**

   <details><summary>Model answer</summary>

   Failures become quiet. In a synchronous system a failure surfaces as an error to a caller who notices; in an asynchronous one it surfaces as a message in a dead-letter queue that nobody is watching, or as work that silently never happened. So the compensation is entirely about making the invisible visible and about defaults that prevent the common mistakes. Every queue needs a dead-letter queue with a named owner and an alert on its rate, because a rising DLQ rate is always a code or data defect rather than a capacity issue. Oldest-message age needs to be the standard latency metric, since throughput and depth both conceal the user experience. Idempotency belongs in a shared consumer wrapper rather than in each handler, because it is the highest-consequence bug class and the least reliably remembered. And end-to-end tracing must span the enqueue and the dequeue, or debugging becomes archaeology across two systems and an unknown delay.

   </details>

---

### Publish-subscribe

*One publisher, many independent subscribers, each with its own copy and its own pace — decoupling producers from consumers they never need to know about.*

**Flow:** `Publisher` → `Topic` → `Subscription A` → `Subscription B` → `Independent handlers`

> **The 30-second version**  
> Publishers announce facts to a topic; each subscription gets its own copy and progresses independently. Adding a consumer needs no producer change — but the schema becomes a contract with consumers you cannot name.

**The problem**

An order is placed. Email needs to know. Inventory needs to know. Analytics, fraud screening, the warehouse, the loyalty programme and, next quarter, three services that do not exist yet. If the order service calls each of them, it accumulates knowledge of every downstream consumer, its availability becomes the intersection of theirs, and adding a consumer means changing the producer.

Publish-subscribe inverts the dependency: the producer announces that something happened and knows nothing about who cares. Consumers subscribe independently, each receiving its own copy, each processing at its own pace, each failing without affecting the others.

> **The distinction from a queue**  
> A **queue** distributes work: each message goes to exactly one consumer, and adding consumers adds throughput. A **topic** broadcasts events: each message goes to every subscription, and adding a subscription adds a new independent consumer of the same events. The difference is not the technology — most brokers do both — it is whether you are dividing work or notifying parties.

**Mental model**

Think of a topic as a noticeboard and each subscription as a separate reader with their own bookmark. The publisher posts once; each reader progresses independently and can fall behind or catch up without affecting anyone else.

1. **Topic** — A named stream of events. The publisher's only dependency.
2. **Subscription** — An independent consumer's view of the topic, with its own delivery state, its own retries and its own backlog.
3. **Event** — A statement of fact about something that happened. Past tense, immutable, owned by the producer.
4. **Fan-out** — One publish becomes N deliveries. The broker handles the multiplication.
5. **Schema** — The contract between one producer and many unknown consumers — the thing that makes or breaks this pattern over time.

> **Events are facts, not commands**  
> `OrderPlaced` is an event: it states what happened and lets each consumer decide what that means for them. `SendConfirmationEmail` is a command dressed as an event: the producer has decided what the consumer should do, which reintroduces the coupling pub/sub exists to remove. When producers start publishing commands, the topic becomes a distributed function call with worse error handling.

**How it works**

**Queue versus topic, concretely**

```text
QUEUE (competing consumers - divide work)
  producer -> [queue] -> worker A   (gets msg 1, 4, 7)
                       -> worker B   (gets msg 2, 5, 8)
                       -> worker C   (gets msg 3, 6, 9)
  adding a worker = more throughput
  each message processed ONCE

TOPIC (pub/sub - notify parties)
  publisher -> [topic] -> subscription "email"     -> all messages
                        -> subscription "inventory"  -> all messages
                        -> subscription "analytics"  -> all messages
  adding a subscription = a new consumer of everything
  each message processed ONCE PER SUBSCRIPTION

COMBINED (the usual production shape)
  topic -> subscription "email" -> [queue] -> 5 email workers
        -> subscription "inventory" -> [queue] -> 3 workers
  fan-out across subscriptions, competing consumers within
```

1. **Publish events, not commands** — Name them in the past tense and describe what happened, not what should be done. This is what keeps the producer ignorant of consumers.
2. **Give every subscription independent failure handling** — Its own retries, its own dead-letter queue, its own backlog. A failing analytics consumer must not delay inventory.
3. **Version the schema and evolve additively** — The producer has no idea who is consuming. Adding optional fields is safe; removing or renaming is a breaking change that you cannot coordinate.
4. **Include enough context in the event** — Consumers should not need to call back to the producer for basic details — that reintroduces the coupling and creates a load amplifier on the producer.
5. **Decide the ordering requirement per consumer** — Most brokers order within a partition only. If a consumer needs per-entity ordering, the partition key must be the entity id.
6. **Publish transactionally with the state change** — An event published without the state change, or a state change with no event, is the most common correctness bug — which is what the transactional outbox exists to prevent.

**Event design: thin versus fat**

```text
THIN EVENT (notification only)
  { type: "OrderPlaced", orderId: "1234", at: "..." }
  + small, stable, no data duplication
  - every consumer calls back to the order service
  - N consumers x every event = load amplification
  - the producer's availability is back in the path

FAT EVENT (event-carried state transfer)
  { type: "OrderPlaced", orderId, customerId, items[],
    total, currency, shippingAddress, at }
  + consumers are fully decoupled; no callback
  + consumers can build their own read models
  - larger payloads; data duplicated in many stores
  - schema changes affect more consumers

PRACTICAL DEFAULT
  fat enough that the common consumers need no callback;
  thin enough not to publish your entire domain model.
  Include ids so a consumer needing more CAN fetch it.
```

> **Make the schema an owned, versioned artifact**  
> In point-to-point integration you can coordinate a change with the one consumer. In pub/sub you cannot, because you do not know who they all are. That makes the event schema a public API: register it, version it, enforce compatibility in CI, and treat a breaking change as a migration project rather than a deploy.

**Worked example**

Adding a fraud-screening consumer to an existing order flow, two years after the events were designed.

**What decides whether this is easy or a project**

```text
GOOD CASE - the event carried state
  OrderPlaced includes customer, items, total, address, IP
  fraud team creates a new subscription
  replays 90 days of history from the topic's retention
  builds its model, deploys, starts consuming
  ORDER SERVICE: unchanged, unaware, unaffected
  -> days of work

BAD CASE - thin events
  OrderPlaced carries only orderId
  fraud service must call the order API for every event
  at 5,000 orders/s that is 5,000 extra reads/s on a
  service sized for its own traffic
  -> order service needs capacity work
  -> fraud service now depends on order service availability
  -> and there is no history to replay: the topic holds
     only ids, and the order service has no point-in-time view
  -> quarters of work

THE LESSON
  the decision that made this cheap or expensive was made
  two years earlier, by whoever designed the event payload.
```

| Metric | Value | Note |
|---|---|---|
| Fat events | days | replay + self-contained |
| Thin events | quarters | **callback amplification** |
| Producer changes | none | in both cases |
| Retention | enables replay | 90 days |

> **Event payload design is a long-lived architectural decision**  
> Because consumers are unknown and arrive later, the payload you publish today determines what future integrations cost. Thin events optimise for today's payload size and pay for it forever in callbacks and coupling. Carrying the state that a reasonable consumer would need — plus ids for anything unusual — is almost always the better trade, and it is nearly impossible to retrofit once consumers exist.

**When to use it**

- **One event, many interested parties**, especially when the set of parties will grow.
- **Decoupling services organisationally**, so teams can consume without coordinating with the producer.
- **Building read models and projections**, where each consumer maintains its own view.
- **Audit and analytics**, which should never be in the producer's critical path.
- **Replay and backfill**, where a retained topic lets a new consumer process history.

**When to avoid it**

- **Do not use it when the producer needs a result** — that is a request, not an event.
- **Do not publish commands**, which reintroduces the coupling and hides a synchronous dependency inside an async mechanism.
- **Do not use it for work distribution**; that is a queue with competing consumers.
- **Do not publish thin events and expect consumers to call back** at scale, which makes the producer a bottleneck again.
- **Do not assume global ordering**, which almost no broker provides across partitions.

**Advantages**

- **The producer has no knowledge of consumers**, so adding one requires no producer change.
- **Independent failure and pace** — a slow or broken consumer affects only its own subscription.
- **Organisational decoupling**: teams integrate by subscribing, without negotiating with the producer team.
- **Replay enables new consumers to process history**, which is what makes late integrations cheap.
- **Natural audit trail**, since the event stream is a record of what happened.

**Disadvantages**

- **Schema becomes a public contract** with unknown consumers, so breaking changes are nearly impossible to coordinate.
- **Debugging spans systems**: what happened to this event is a distributed question.
- **No backpressure to the producer** — a slow consumer builds a backlog rather than slowing the publisher.
- **Ordering is partition-scoped at best**, which complicates per-entity sequencing.
- **Eventual consistency is inherent**, so consumers' views lag and can disagree with each other.
- **Duplicate delivery** means every consumer still needs idempotency.

**Trade-offs**

**Integration styles**

|  | Synchronous call | Queue (work) | Topic (pub/sub) |
|---|---|---|---|
| Producer knows consumer | Yes | Yes (the queue) | No |
| Adding a consumer | Producer change | More workers | New subscription only |
| Failure isolation | None — coupled | Good | Excellent, per subscription |
| Result available | Yes | No | No |
| Ordering | n/a | Weak | Per partition |
| Backpressure to producer | Yes | Via queue depth | None |

> **Framing the trade**  
> “I'd publish `OrderPlaced` as an event carrying the order state, so email, inventory and the fraud service that doesn't exist yet can each subscribe without the order service knowing. The costs I'm accepting are eventual consistency across those views and a schema that becomes a public contract — so it gets registered and compatibility-checked in CI.”

**How it fails**

**Pub/sub failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Breaking change breaks unknown consumers | Schema treated as internal | Schema registry; additive-only evolution; compatibility checks in CI |
| Producer overloaded by consumer callbacks | Thin events requiring lookups | Carry state in the event |
| Event published but state not committed | Publish outside the transaction | Transactional outbox |
| One consumer's backlog affects others | Shared subscription or shared consumer pool | Independent subscriptions and pools |
| Out-of-order processing per entity | Partition key not the entity id | Partition by entity; or make handlers order-independent with versions |
| Duplicate side effects | At-least-once delivery, non-idempotent consumer | Idempotency keys per consumer |
| Cannot onboard a new consumer | No retention, no replay | Retain events long enough to bootstrap a consumer |

> **The dual-write problem**  
> Writing to the database and then publishing an event is not atomic. A crash between them either loses the event — so downstream systems never learn about a real change — or, if published first, announces a change that never committed. This is the single most common correctness bug in event-driven architectures, and the transactional outbox pattern exists specifically to eliminate it.

**Limits**

> **Design guidance**
>
> - **Event payload**: carry what a reasonable consumer needs, plus ids for anything unusual. Typically kilobytes, not megabytes.
> - **Retention**: long enough to bootstrap a new consumer — days to weeks is common, and it is what makes late integrations cheap.
> - **Ordering**: guaranteed per partition only; partition by the entity whose order matters.
> - **Schema evolution**: additive changes only; removals and renames require a versioned topic and a migration.
> - **Every subscription needs its own dead-letter queue and alerting**, since failures are independent.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Pub/sub topic | One event, many unknown consumers | Schema as a public contract; eventual consistency |
| Point-to-point queue | Dividing work among peers | Producer knows the consumer |
| Synchronous API call | Caller needs a result | Availability coupling |
| Change data capture | Consumers need every database change | Exposes internal schema; less semantic |
| Shared database | Simplest integration | Tight coupling; no independence |
| Webhooks | External consumers you do not control | Delivery reliability; no replay |

Change data capture is worth distinguishing: it publishes *row changes* rather than *domain events*, which makes it excellent for replication and analytics and poor as a public contract, because it exposes internal schema to consumers who will then depend on it.

**In real systems**

- **Kafka topics with consumer groups** provide both patterns: fan-out across groups, competing consumers within a group.
- **Google Pub/Sub and AWS SNS+SQS** make the distinction explicit — a topic fans out to subscriptions, each of which is typically a queue.
- **Schema registries with compatibility enforcement** exist because a pub/sub payload is a contract with consumers you cannot enumerate.
- **Event-driven microservice architectures** use this as the primary integration mechanism precisely for the organisational decoupling it provides.
- **Debezium publishing database changes** is the change-data-capture variant, widely used to bootstrap consumers without modifying producers.

**Common mistakes**

- **Publishing commands** instead of facts, recreating the coupling.
- **Thin events** that force every consumer to call back to the producer.
- **Dual writes** — committing state and publishing separately.
- **Treating the schema as internal**, then breaking unknown consumers.
- **Assuming global ordering** where only partition ordering exists.
- **Sharing subscriptions or consumer pools**, coupling independent consumers' failures.
- **Retention too short to bootstrap a new consumer.**

**The staff-level view**

Pub/sub is an organisational tool as much as a technical one: it lets teams integrate without negotiating, which is its main value and the source of its main risk.

- **Treat event schemas as public APIs.** Register them, enforce compatibility in CI, and require a versioned topic plus a migration plan for breaking changes — you cannot coordinate with consumers you cannot name.
- **Push for events that carry state.** The payload decision made today determines what integrations cost for years, and it cannot be retrofitted once consumers exist.
- **Mandate the transactional outbox for publishing.** Dual writes are the most common correctness bug in event-driven systems and the least visible.
- **Insist events are facts, not commands.** The moment producers publish instructions, the coupling is back and the error handling is worse than a synchronous call.
- **Set retention from the bootstrap requirement**, not from storage cost. Replayability is what makes adding a consumer a days-long task rather than a quarter-long one.

**Go deeper**

In publish-subscribe, a producer emits events to a topic and knows nothing about who consumes them. Each subscription receives every message and processes at its own pace with its own retries and backlog, so a failing analytics consumer cannot delay inventory. The contrast with a queue is that a queue divides work — one message, one consumer — while a topic notifies parties, delivering each message once per subscription.

Two design decisions dominate. Events must be facts in the past tense, not commands: publishing `SendConfirmationEmail` rather than `OrderPlaced` puts the producer's knowledge of consumers back into the system. And events should carry enough state that common consumers need no callback — thin events containing only ids turn the producer into a bottleneck, restore availability coupling, and make replay useless because history holds no data.

The costs are a schema that becomes a public contract with consumers you cannot enumerate, which makes additive-only evolution and a registry with CI enforcement necessary rather than nice; eventual consistency between consumers' views; and the dual-write problem — committing state and publishing an event are not atomic, so a crash between them loses events or announces changes that never happened, which is what the transactional outbox exists to solve.

Publish-subscribe inverts integration dependency: the producer announces what happened and remains ignorant of who cares. That inversion is as much an organisational mechanism as a technical one, since it lets teams integrate by subscribing rather than by negotiating.

**Queue versus topic.** A queue distributes work among competing consumers — each message processed once, more consumers meaning more throughput. A topic fans out to subscriptions — each message processed once per subscription, more subscriptions meaning more independent consumers. Most brokers provide both, and the standard production shape composes them: a topic fans out to several subscriptions, each backed by a queue with a pool of workers, giving both independence across consumers and parallelism within each.

**Events are facts.** Naming them in the past tense and describing what occurred, rather than what should be done, is what preserves the decoupling. A producer publishing `SendConfirmationEmail` has encoded knowledge of a consumer and turned the topic into a distributed function call with no return value and worse error handling. This distinction erodes gradually and is worth defending explicitly, because each individual command-shaped event seems harmless.

**Payload design is a long-lived decision.** Thin events carrying only identifiers minimise payload size and maximise long-term cost: every consumer must call back to the producer, which at scale makes the producer a bottleneck sized for everyone else's traffic and restores its availability into everyone's path — and replay becomes worthless, because retained history contains no state to replay. Events carrying the relevant domain state let consumers be genuinely independent and build their own read models, at the cost of larger payloads and a schema more consumers depend on. Because this choice determines what future integrations cost and cannot be retrofitted once consumers exist, the default should lean toward carrying state, with ids included for consumers needing something unusual.

**The schema is a public API.** The defining property of pub/sub — unknown consumers — means a breaking change cannot be coordinated. That makes a schema registry with compatibility enforcement in CI a structural requirement rather than a nicety: additive changes are free, removals and renames require a new versioned topic and a migration period publishing both. Ownership metadata and consumer registration make dependencies discoverable before a deploy rather than after an incident. The cultural corollary is that publishing an event is a commitment, and teams routinely make it casually.

**Two failure modes to design out.** The dual-write problem — committing state and publishing separately is not atomic, so a crash loses events or announces uncommitted changes — is the most common and most silent correctness bug in event-driven systems, and the transactional outbox eliminates it by writing the event in the same transaction and relaying separately. And delivery remains at-least-once, so every consumer independently needs idempotency, scoped to its own effect. Alongside those, retention should be set from the bootstrap requirement rather than from storage cost, because replayability is what makes onboarding a new consumer a matter of days instead of a quarter.

**Prove it — interview questions**

1. **[Basic] What is the difference between a queue and a topic?**

   <details><summary>Model answer</summary>

   A queue distributes work: each message is processed by exactly one consumer, so adding consumers adds throughput. A topic broadcasts events: each message is delivered to every subscription, so adding a subscription adds a new independent consumer of the same events. Most brokers support both, and the common production shape combines them — a topic fans out to several subscriptions, each of which feeds a queue with competing workers.

   </details>

2. **[Basic] Why publish events rather than commands?**

   <details><summary>Model answer</summary>

   Because an event states what happened and lets each consumer decide what it means for them, whereas a command states what should be done and therefore encodes the producer's knowledge of the consumer. Publishing `SendConfirmationEmail` rather than `OrderPlaced` means the order service knows there is an email service, which is exactly the coupling pub/sub exists to remove — and it turns the topic into a distributed function call with worse error handling and no return value.

   </details>

3. **[Senior] How much data should an event carry?**

   <details><summary>Model answer</summary>

   Enough that a reasonable consumer does not need to call back. Thin events carrying only an id force every consumer to query the producer, which at scale turns the producer into a bottleneck and puts its availability back in everyone's path — and it makes replay useless, because the historical events contain no state. Fat events carrying the relevant domain state let consumers be fully independent and build their own read models. The cost is larger payloads and a schema that more consumers depend on. My default is to carry what the common consumers need plus ids for anything unusual, because the payload decision determines what future integrations cost and is nearly impossible to retrofit.

   </details>

4. **[Senior] What is the dual-write problem in event publishing?**

   <details><summary>Model answer</summary>

   Writing to the database and publishing an event are two operations that are not atomic. If the process crashes between them, you either commit a change nobody learns about, or — if you publish first — announce a change that never committed, causing downstream systems to act on something that did not happen. It is the most common correctness bug in event-driven architectures and it is silent. The standard fix is the transactional outbox: write the event to a table in the same transaction as the state change, and have a separate relay publish from that table, which converts two systems into one atomic write plus an at-least-once relay.

   </details>

5. **[Staff] A new team wants to consume your events for fraud detection. What determines whether that is easy?**

   <details><summary>Model answer</summary>

   Three decisions made long before they asked. First, whether the events carry state: if they contain only ids, the fraud service must call back for every event, which at high volume needs capacity work on my service and puts my availability in their path. Second, retention: if the topic retains ninety days, they can replay history to build and validate their model before going live, which turns a quarter of work into days. Third, schema stability and registration, which determines whether they can rely on the payload or must defensively handle unknown shapes. Notably, none of these require any change from my team when they onboard — which is the point of pub/sub — but all of them were determined by choices my team made when designing the events.

   </details>

6. **[Principal] How do you govern event schemas across an organisation?**

   <details><summary>Model answer</summary>

   By treating them as public APIs with enforcement rather than convention, because the defining property of pub/sub is that you cannot enumerate your consumers and therefore cannot coordinate a breaking change. Concretely: a schema registry that every producer publishes to, compatibility checks enforced in CI so a producer physically cannot deploy an incompatible change, and additive-only evolution as the rule — new optional fields are free, removals and renames require a new versioned topic plus a migration during which both are published. I would also require ownership metadata on every topic, so a consumer can find who to talk to, and consumer registration so producers have at least an advisory view of who depends on them. The cultural point that matters more than the tooling is that publishing an event is a commitment, not a side effect: teams routinely publish events casually and then discover three services depend on a field they wanted to remove, and the registry is what makes that discoverable before the deploy rather than after the incident.

   </details>

---

### Partitioned event logs

*An append-only log split into partitions, each strictly ordered, with consumers tracking their own position — durable, replayable, and ordered only where you chose.*

**Flow:** `Producer key` → `Partition` → `Append log` → `Consumer offset` → `Replay`

> **The 30-second version**  
> An append-only log split into ordered partitions, retained regardless of consumption, with consumers tracking their own offsets — so replay is free and ordering is per key.

**The problem**

Traditional message brokers delete a message once it is acknowledged, which makes them a transport rather than a record. That is fine until you need a second consumer of the same data, or need to reprocess after fixing a bug, or need to bootstrap a new service from history. None of those are possible when the messages are gone.

A partitioned log keeps everything. Messages are appended to an immutable ordered sequence, retained for a configured period regardless of consumption, and each consumer tracks its own position. Reading is a sequential scan from an offset, which is why a single node can serve enormous throughput — and why replay is free rather than a feature.

> **The two properties that make it different**  
> **Retention independent of consumption**: the log is a record, not a transport, so new consumers can read history and existing ones can rewind. **Ordering within a partition**: the partition key determines which events are ordered relative to each other, which converts a global ordering problem into a per-entity one you can actually satisfy.

**Mental model**

A topic is a set of append-only files. Each event is assigned to a partition by hashing its key, and within a partition the order is the order of appends. Consumers are readers with bookmarks, not recipients of a delivery.

1. **Partition** — An ordered, immutable sequence. Ordering exists here and nowhere else.
2. **Key** — Determines the partition, and therefore what is ordered with what. The single most important design decision.
3. **Offset** — A consumer's position. Committing it is what makes progress durable; it is not an acknowledgement to the broker.
4. **Consumer group** — A set of consumers sharing the work, with each partition assigned to exactly one member.
5. **Retention** — Time- or size-based. Independent of whether anyone has read the data.

> **Parallelism is bounded by partition count**  
> Within a consumer group, each partition has exactly one consumer, so you cannot have more useful consumers than partitions. Partition count is therefore a scaling ceiling chosen at topic creation and awkward to increase later — because increasing it changes which partition a key hashes to, breaking ordering for existing keys. Over-provision partitions early.

**How it works**

**Partitioning, ordering and consumption**

```text
PRODUCE  key="user-42"
  partition = hash("user-42") % 12  ->  partition 7
  append to partition 7, offset 981,204

PARTITION 7  (ordered, immutable)
  ... 981202  981203  981204  981205 ...
                          |
  consumer group "billing" committed offset 981203
  consumer group "search"  committed offset 640,110  (behind)

GUARANTEES
  all events for user-42 are in partition 7, in order
  no ordering guarantee BETWEEN partitions
  each group progresses independently
  rewinding = setting the offset backwards

CONSUMER GROUP ASSIGNMENT (12 partitions)
  3 consumers -> 4 partitions each
  12 consumers -> 1 each        <- maximum useful parallelism
  15 consumers -> 3 idle        <- partition count is the ceiling
```

1. **Choose the partition key from the ordering requirement** — Whatever must be processed in order must share a key. Per-user, per-account, per-device. If nothing needs ordering, use a random or round-robin key for even distribution.
2. **Over-provision partitions** — Increasing partition count later rehashes keys to different partitions, breaking ordering for in-flight entities. Pick a number well above current parallelism needs.
3. **Commit offsets after processing, not before** — Committing first gives at-most-once semantics and silently loses work on crash. Committing after gives at-least-once, so handlers must be idempotent.
4. **Watch for hot partitions** — Key skew means one partition carries disproportionate traffic, and its single consumer becomes the bottleneck regardless of how many others are idle.
5. **Set retention from replay needs, not storage cost** — Retention is what makes bootstrapping a new consumer cheap; too short and every new integration becomes a migration project.
6. **Use log compaction for state, not events** — Compaction retains only the latest value per key, turning a log into a changelog you can replay to reconstruct current state — a different tool from time-based retention.

**Consumer lag: the metric that matters**

```text
LAG = latest offset in partition - consumer's committed offset

lag in MESSAGES tells you the backlog size
lag in TIME tells you how stale the consumer's view is
  -> time lag is the user-visible number

HEALTHY        lag oscillates near zero
FALLING BEHIND lag growing steadily -> consumption < production
STUCK          lag growing, throughput zero -> poison message
               or a consumer crash-loop

CATCH-UP TIME = lag / (consumer capacity - production rate)
  if consumption <= production, it never catches up

PER-PARTITION LAG matters more than aggregate:
  total lag looks fine while ONE partition is hours behind
  because its key is hot or its consumer is stuck
```

> **Rebalancing is a stop-the-world event**  
> When a consumer joins, leaves or is presumed dead, partitions are reassigned and consumption pauses across the group. A consumer that takes too long between polls is declared dead and triggers a rebalance, which slows everyone, which can cause more consumers to miss their deadline — a rebalance storm. Keep per-poll processing short, use cooperative rebalancing where available, and tune session timeouts to real processing times.

**Worked example**

A payment ledger requiring strict per-account ordering, processed by several downstream consumers.

**Partition key and its consequences**

```text
REQUIREMENT
  per-account event order must be preserved (a debit
  cannot be processed before the credit that funds it)
  50,000 events/s, 20M accounts
  consumers: ledger projection, fraud, analytics, notifications

PARTITION KEY = account_id
  -> all events for an account land in one partition, ordered
  -> different accounts are independent, which is correct

PARTITION COUNT
  current need: 50,000/s / ~5,000 per consumer = 10 consumers
  choose 64 partitions
    -> room to scale to 64 consumers without repartitioning
    -> 20M accounts over 64 partitions: no hot partition
       unless one account is extreme

CONSUMER GROUPS (independent)
  ledger:        64 consumers, lag must stay < 1s
  fraud:         16 consumers, lag < 30s acceptable
  analytics:     4 consumers, lag < 5min acceptable
  notifications: 8 consumers, lag < 10s

WHAT THIS BUYS
  each group tunes its own parallelism and lag target
  a broken analytics consumer does not affect the ledger
  a bug fix means rewinding ONE group's offsets and replaying

WHAT IT COSTS
  no cross-account ordering (correct - we do not need it)
  a single extreme account is capped by one partition's
    consumer throughput
```

| Metric | Value | Note |
|---|---|---|
| Partitions | 64 | ceiling on parallelism |
| Ordering | per account | **where it is needed** |
| Groups | 4 independent | own lag targets |
| Replay | per group | rewind offsets |

> **Replay changes how you handle bugs**  
> In a delete-on-ack broker, a consumer bug that corrupted a projection means restoring from backup and hoping. With a retained log, you fix the code, reset that consumer group's offsets, and reprocess — the source data is still there. This changes the risk profile of downstream consumers enough that it justifies retention on its own, independent of the cost.

**When to use it**

- **Multiple independent consumers of the same stream**, each at its own pace with its own offsets.
- **Replay and reprocessing**, whether for bug fixes, new consumers, or rebuilding projections.
- **Per-entity ordering requirements**, satisfied by partitioning on the entity key.
- **Very high throughput**, where sequential appends and reads outperform per-message broker bookkeeping.
- **Event sourcing and change data capture**, where the log is the system of record.

**When to avoid it**

- **Do not use it as a task queue with per-message acknowledgement** — offsets are sequential positions, so one slow message blocks its partition.
- **Do not expect global ordering**; it exists only within a partition.
- **Do not under-provision partitions**, since increasing them later breaks key-to-partition mapping.
- **Do not use it for request-response**; the consumer cannot reply to the producer.
- **Do not use it where messages must be individually delayed or prioritised** — a log is strictly sequential.

**Advantages**

- **Retention independent of consumption**, enabling replay, late consumers and backfill.
- **Very high throughput**, because appends and reads are sequential and batched.
- **Strict ordering per partition**, which is usually the ordering you actually need.
- **Multiple independent consumer groups** over the same data with no producer involvement.
- **The log is an audit trail**, since nothing is deleted on consumption.
- **Compaction gives a changelog**, allowing state reconstruction from the log alone.

**Disadvantages**

- **Parallelism is capped by partition count**, chosen early and hard to change.
- **Head-of-line blocking within a partition**: one slow or failing message stalls everything behind it.
- **Hot partitions** from key skew cannot be fixed by adding consumers.
- **Rebalancing pauses consumption** across the group and can storm under bad configuration.
- **Storage cost scales with retention**, and retention is what makes it valuable.
- **Offset management is subtle** — commit timing determines your delivery semantics.

**Trade-offs**

**Log versus traditional queue**

|  | Partitioned log | Traditional queue |
|---|---|---|
| Message lifetime | Retention period | Until acknowledged |
| Replay | Built in — reset offsets | Not possible |
| Multiple consumers | Independent groups | Competing consumers only |
| Ordering | Strict per partition | Usually none |
| Per-message ack | No — sequential offsets | Yes |
| One bad message | Blocks its partition | Blocks only itself |
| Parallelism ceiling | Partition count | Unbounded |

The per-message acknowledgement row is the practical dividing line. If individual messages fail independently and must not block others, a queue is the right model. If the stream is ordered and consumers process it sequentially, a log is.

**How it fails**

**Partitioned log failures**

| Symptom | Cause | Fix |
|---|---|---|
| One partition hours behind | Hot key, or a stuck consumer on that partition | Check per-partition lag; re-key; fix the poison message |
| Cannot add consumers to go faster | More consumers than partitions | Increase partitions (accepting the ordering break) or re-key |
| Rebalance storm | Slow processing exceeding session timeout | Shorter poll batches; cooperative rebalancing; tuned timeouts |
| Work lost after a crash | Offsets committed before processing | Commit after processing; accept at-least-once and be idempotent |
| Duplicate processing | At-least-once redelivery after rebalance | Idempotent handlers keyed on the event id |
| Cannot bootstrap a new consumer | Retention too short | Set retention from bootstrap needs; use compaction for state topics |
| Ordering broken after scaling | Partition count increased, keys rehashed | Over-provision initially; or use explicit partition assignment |

> **Head-of-line blocking within a partition**  
> A log has no per-message acknowledgement. If one event cannot be processed — a bug, a malformed payload, a missing referent — the consumer either skips it (losing it), retries forever (blocking the partition and everything behind it), or routes it to a side queue. The third option is correct but must be built deliberately; the default behaviour of a naive consumer is the second, and a single poison message can stall an entire partition indefinitely.

**Limits**

> **Design numbers**
>
> - **Partition count** is the parallelism ceiling and is painful to raise — provision several times current need.
> - **Per-partition throughput** is bounded by one consumer's processing rate; a hot key cannot exceed it.
> - **Retention**: days to weeks for replay; compacted topics retain the latest value per key indefinitely.
> - **Consumer lag in time**, not messages, is the user-visible metric — and per-partition, not aggregate.
> - **Rebalance duration** grows with partition count and consumer count; cooperative protocols reduce the stop-the-world effect.

**Alternatives**

| System | Best for | Trade |
|---|---|---|
| Partitioned log | Replay, multiple consumers, ordered streams | Partition ceiling; HOL blocking |
| Traditional queue | Independent task processing | No replay; no ordering |
| Log + queue bridge | Ordered ingest, independent task retry | Two systems |
| Database change log (CDC) | Capturing every row change | Internal schema exposed |
| Event store | Event sourcing as the system of record | Narrower ecosystem |
| Object storage + batch | Very high volume, high latency tolerance | Not near-real-time |

The log-plus-queue bridge is a common and underrated shape: consume the log sequentially to preserve ordering, and immediately publish each event into a per-consumer queue where individual failures can retry and dead-letter without blocking the stream.

**In real systems**

- **Kafka** established this model — partitions, consumer groups, offsets and retention — and most alternatives follow its vocabulary.
- **Kafka Streams and Flink** colocate per-key state with the partition owning that key, which is why partition choice determines both ordering and state locality.
- **Log compaction** underpins changelog topics used to rebuild materialised state after a failure, which is a distinct use from time-based retention.
- **Debezium and CDC pipelines** publish database changes into partitioned logs, using the primary key as the partition key so per-row ordering is preserved.
- **Event sourcing systems** treat the log as the system of record, with all read models being replayable projections of it.

**Common mistakes**

- **Choosing a partition key for even distribution** when ordering per entity was the requirement, or vice versa.
- **Too few partitions**, capping parallelism at a number you cannot raise without breaking ordering.
- **Committing offsets before processing**, silently losing work on crash.
- **No dead-letter path**, so a poison message blocks a partition indefinitely.
- **Monitoring aggregate lag** while one partition is hours behind.
- **Retention set from storage cost** rather than from replay needs.
- **Long per-poll processing**, triggering rebalances and then rebalance storms.

**The staff-level view**

A partitioned log makes three decisions expensive to change later: the partition key, the partition count and the retention. All three are set on day one, usually by whoever creates the topic.

- **Treat the partition key as an architectural decision.** It determines ordering, state locality, skew and future parallelism — and changing it means a new topic and a migration.
- **Over-provision partitions deliberately.** Raising the count later rehashes keys and breaks ordering for in-flight entities, so the cost of extra partitions now is far below the cost of repartitioning later.
- **Alert on per-partition time lag**, since aggregate lag hides a single stalled partition, which is the most common real failure.
- **Build the poison-message path explicitly.** Without a side queue, one unprocessable event blocks a whole partition, and the naive behaviour is an infinite retry loop.
- **Set retention from bootstrap and replay requirements.** Being able to reset offsets and reprocess after a bug changes the risk profile of every downstream consumer.

**Go deeper**

A partitioned log appends events to immutable ordered sequences, assigns each event to a partition by hashing its key, and lets consumers track their own offsets. Two properties distinguish it from a traditional broker: retention is independent of consumption, so history can be replayed and new consumers can bootstrap; and ordering is strict within a partition, which converts an unachievable global ordering requirement into a per-entity one.

Three decisions are made early and are expensive to revisit. The partition key fixes what is ordered with what, plus state locality and skew. The partition count caps parallelism, because each partition is consumed by exactly one member of a group, and raising it later rehashes keys and breaks ordering. And retention determines whether onboarding a new consumer is days or quarters of work.

The characteristic failure is head-of-line blocking: offsets are sequential positions, not per-message acknowledgements, so one unprocessable event blocks everything behind it in its partition. The naive consumer retries forever; the correct design bounds attempts and side-tracks to a dead-letter topic. Monitor per-partition lag in time rather than aggregate lag in messages, because a single stalled partition is invisible in the aggregate and is the most common real problem.

A partitioned log is a record rather than a transport, and nearly every consequence — good and bad — follows from that plus the decision to order within partitions instead of globally.

**Retention independent of consumption** is what enables replay. A new consumer can read history to bootstrap its state, an existing one can rewind after a bug and reprocess, and several independent consumer groups can read the same stream at different paces. This materially lowers the risk of building downstream consumers: a projection corrupted by a bug is repaired by resetting offsets rather than by restoring backups. That property alone often justifies the storage cost, and retention should therefore be set from bootstrap and replay requirements rather than from cost.

**Partitioning is where the design lives.** The key determines what is ordered relative to what, which partition holds an entity's state in stream processors, and how evenly load spreads. Partition count is the parallelism ceiling, since each partition is assigned to exactly one consumer within a group. Raising the count later rehashes keys so an entity's events can land in two partitions, breaking ordering for anything in flight — which makes over-provisioning at creation far cheaper than repartitioning afterwards. Key skew produces hot partitions whose single consumer is the bottleneck no matter how many others sit idle, and no amount of added capacity fixes it.

**Offsets are not acknowledgements.** Committing offset N asserts that everything up to N is processed, so there is no way to complete message N+2 while N+1 is outstanding. One unprocessable event therefore blocks its partition entirely. The defaults are all unacceptable — skipping loses data, retrying forever stalls the stream, crashing triggers rebalances — so a bounded-retry-then-side-track path must be built deliberately, with the dead-letter topic having an owner and an alert. Commit timing also determines delivery semantics: before processing gives at-most-once and silently loses work on crash; after gives at-least-once and requires idempotent handlers.

**Rebalancing is the operational hazard.** When consumers join, leave, or fail to poll within the session timeout, partitions are reassigned and consumption pauses across the group. A consumer doing long per-poll processing is declared dead, triggering a rebalance that slows everyone, which can push more consumers past their deadline — a rebalance storm. Keeping per-poll batches small, using cooperative rebalancing, and tuning session timeouts to real processing times are the standard mitigations.

**Monitor the right lag.** Lag measured in messages is a backlog size; lag measured in time is the staleness a user experiences. And per-partition lag matters far more than aggregate, because a single stalled partition — from a hot key or a poison message — is invisible when averaged across sixty-four others. The derived figure during an incident is catch-up time: lag divided by consumer capacity minus production rate, which is infinite whenever consumption merely matches production.

**Know when it is the wrong tool.** The absence of per-message acknowledgement makes a log a poor task queue: workloads where items fail independently and must not block each other want a queue, often fed from the log so you get ordered durable ingest with independent per-item retry. Similarly, individual message delay or priority cannot be expressed in a strictly sequential structure. The log is for ordered, replayable streams consumed sequentially; it is not a universal replacement for messaging.

**Prove it — interview questions**

1. **[Basic] What ordering guarantee does a partitioned log provide?**

   <details><summary>Model answer</summary>

   Strict ordering within a partition, and none across partitions. Since the partition is chosen by hashing the message key, everything sharing a key is ordered relative to itself. That is usually the guarantee you actually want — all events for one account, one user, one device in order — and it is achievable at scale in a way that global ordering is not, because global ordering would require a single partition and therefore a single consumer.

   </details>

2. **[Basic] Why is partition count a scaling ceiling?**

   <details><summary>Model answer</summary>

   Because within a consumer group each partition is assigned to exactly one consumer. With twelve partitions you can usefully run at most twelve consumers; additional ones sit idle. Raising the count later is painful because keys rehash to different partitions, so events for an entity can appear in two partitions and ordering breaks for anything in flight. The practical guidance is to over-provision at topic creation, since extra partitions are cheap and repartitioning is a migration.

   </details>

3. **[Senior] How do offsets differ from acknowledgements?**

   <details><summary>Model answer</summary>

   An acknowledgement in a traditional queue removes one specific message. An offset is a sequential position: committing offset N asserts that everything up to N has been processed. That means there is no way to acknowledge message N+2 while N+1 is still outstanding, so a single unprocessable message blocks everything behind it in that partition. It also means commit timing determines delivery semantics — committing before processing gives at-most-once and loses work on crash, while committing after gives at-least-once and requires idempotent handlers.

   </details>

4. **[Senior] What do you do about a poison message in a log?**

   <details><summary>Model answer</summary>

   You need an explicit path, because the defaults are all bad: skipping loses data, retrying forever blocks the partition and everything behind it, and crashing the consumer triggers rebalances. The correct design is to attempt the message a bounded number of times and then publish it to a side topic or dead-letter queue, commit the offset, and continue — accepting that this one event is now out of band and needs an owner and an alert. For consumers where ordering matters strictly, that side-tracking is itself a correctness decision that has to be considered, because subsequent events for the same entity will now be processed without it.

   </details>

5. **[Staff] Design the partitioning for a payment event stream requiring per-account ordering.**

   <details><summary>Model answer</summary>

   Partition by account id, because that is precisely the ordering requirement — a debit must not be processed before the credit that funds it — and different accounts are genuinely independent. I would choose a partition count several times the current parallelism need, perhaps sixty-four for a workload needing ten consumers, because raising it later rehashes keys and breaks ordering for in-flight accounts. With twenty million accounts, skew is unlikely unless a single account is extreme, and I would monitor per-partition lag rather than aggregate to catch that. Each downstream concern becomes its own consumer group with its own parallelism and lag target — a ledger projection needing sub-second lag, fraud tolerating thirty seconds, analytics tolerating minutes — which is the main benefit of the log model, since a broken analytics consumer cannot affect the ledger and a bug fix means rewinding one group's offsets and replaying.

   </details>

6. **[Principal] What makes a partitioned log a different architectural commitment from a queue?**

   <details><summary>Model answer</summary>

   It becomes a system of record rather than a transport, and that changes what the organisation can do. Because retention is independent of consumption, a new team can subscribe and replay history without any change from the producer, and a bug in a downstream projection is recoverable by resetting offsets rather than by restoring backups — which materially lowers the risk of building consumers at all. The cost is a set of decisions that are expensive to revisit: the partition key fixes both ordering and state locality, partition count caps parallelism, and retention determines whether late integrations are cheap. I would also be clear that it is not a general replacement for queues: the absence of per-message acknowledgement means one failing event blocks its partition, so task-like workloads where items fail independently still want a queue — often fed from the log, which gives ordered durable ingest with independent per-item retry.

   </details>

---

### Consumer groups and rebalancing

*Partitions are distributed among group members, and every membership change triggers a reassignment that pauses processing — which is where most operational pain lives.*

**Flow:** `Group membership` → `Assignment` → `Partition owner` → `Checkpoint` → `Rebalance`

> **The 30-second version**  
> A group assigns each partition to exactly one member and reassigns on any membership change. Those reassignments pause processing, so cooperative assignment, static membership and short poll batches are what make it workable.

**The problem**

A stream has sixty-four partitions and you want twenty consumers to share them. Something must decide which consumer owns which partition, detect when a consumer dies, reassign its partitions to survivors, and ensure that no two consumers ever process the same partition simultaneously — because that would break the ordering guarantee the partitioning exists to provide.

Consumer groups solve this, and the solution is where most of the operational difficulty in stream processing lives. Every deploy, every scale event, every slow consumer and every transient network problem triggers a reassignment, and a reassignment pauses processing for the entire group.

> **Why rebalancing hurts more than it should**  
> **Stop-the-world**: in the classic protocol, all consumers revoke all partitions, then receive new assignments. Nobody processes anything during that window. **Cascading**: a slow consumer misses its deadline, is declared dead, triggers a rebalance, whose pause makes other consumers miss theirs. **State loss**: stateful consumers must rebuild local state for newly assigned partitions, which can take minutes.

**Mental model**

A group is a membership protocol plus an assignment strategy. Members heartbeat to prove liveness; a coordinator detects changes and triggers reassignment; each partition ends up owned by exactly one member.

1. **Group coordinator** — A broker-side component tracking membership and driving the assignment protocol.
2. **Heartbeat and session timeout** — Liveness. Missing heartbeats for the session timeout means the member is presumed dead.
3. **Max poll interval** — A separate deadline: a member that does not fetch new records within it is presumed stuck, even if heartbeats continue.
4. **Assignment strategy** — How partitions are distributed — range, round-robin, sticky, or cooperative sticky.
5. **Offset commit** — Progress is durable only once committed. Uncommitted work is reprocessed after reassignment.

> **Two different timeouts, two different failures**  
> **Session timeout** catches a crashed or partitioned consumer, and heartbeats are usually sent by a background thread so they continue even while processing. **Max poll interval** catches a consumer that is alive but stuck — processing one batch for too long. The second is the one that surprises people: a consumer doing legitimate slow work is ejected mid-task, its partitions reassigned, and the work is reprocessed elsewhere.

**How it works**

**The rebalance protocol, and why it pauses**

```text
EAGER (classic) REBALANCE
  1  coordinator detects a membership change
  2  ALL members revoke ALL partitions       <- processing stops
  3  members rejoin and send their subscriptions
  4  leader computes the assignment
  5  members receive assignments and resume
  -> the entire group is idle for steps 2-5

COOPERATIVE (incremental) REBALANCE
  1  coordinator detects a change
  2  leader computes which partitions must MOVE
  3  only those partitions are revoked
  4  everything else keeps processing throughout
  -> pause is limited to the partitions actually moving

STICKY ASSIGNMENT
  prefers to keep each consumer's existing partitions,
  minimising both movement and state rebuild
  -> combine sticky + cooperative for the least disruption
```

1. **Keep per-poll processing well under max poll interval** — Process fewer records per poll, or hand work to a separate thread pool and poll continuously. This is the single most common cause of avoidable rebalances.
2. **Use cooperative sticky assignment where available** — It converts a stop-the-world pause into a partial one, and it minimises state rebuilding by keeping existing assignments.
3. **Set static membership for planned restarts** — A stable member id lets a restarting consumer reclaim its partitions without triggering a full rebalance, which makes rolling deploys far less disruptive.
4. **Commit offsets frequently enough to bound reprocessing** — Everything since the last commit is reprocessed after reassignment; the commit interval is therefore your duplicate-work window.
5. **Handle the revoke callback** — On revocation, commit offsets and flush any partial state. Failing to do so means the new owner reprocesses more, or worse, that in-flight work is silently inconsistent.
6. **Be careful with stateful consumers** — A reassigned partition means rebuilding local state from a changelog, which can dominate the rebalance duration. Standby replicas exist for exactly this.

**Why a slow consumer takes down the group**

```text
max.poll.interval = 5 min
a consumer processes a batch of 500 records, each taking 1s
  -> 8 minutes per poll  -> exceeds the interval

t+5min   coordinator: consumer is stuck -> rebalance
         its partitions move to others
t+5min   OTHER consumers pause during the rebalance
         their in-flight batches now take longer
t+10min  another consumer exceeds its interval -> rebalance
...      rebalance storm; throughput collapses to near zero

THE FIX IS NOT A LONGER TIMEOUT
  raising max.poll.interval to 20 min delays detection of a
  genuinely dead consumer by 20 minutes.
THE FIX IS SMALLER BATCHES
  max.poll.records = 50 -> 50s per poll -> comfortable margin
```

> **Raising timeouts is almost always the wrong fix**  
> When rebalances are triggered by slow processing, the instinct is to increase the timeout. That trades one problem for a worse one: detection of genuinely failed consumers now takes as long as the new timeout, during which their partitions are unprocessed and lag grows. Reduce the work per poll instead, so the deadline is comfortable rather than marginal.

**Worked example**

A rolling deploy of a twenty-instance stateful stream processor, and how to make it not hurt.

**Deploy disruption, before and after**

```text
NAIVE (eager rebalance, no static membership)
  20 instances, rolling deploy one at a time
  each restart = 1 full rebalance
  each rebalance:
    all 64 partitions revoked
    stateful consumers rebuild local state from changelog
    rebuild time ~90s per instance's partitions
  20 restarts x (pause + rebuild) -> ~30 min of degraded
  processing, with lag spiking throughout

IMPROVED
  cooperative sticky assignment
    -> only the restarting instance's partitions move
    -> the other 19 keep processing
  static group membership
    -> a restarting instance reclaims its OWN partitions
    -> for a restart within session.timeout, NO rebalance
       happens at all
  standby replicas for state
    -> a warm copy of each partition's state exists elsewhere,
       so a genuine move rebuilds in seconds not minutes

RESULT
  rolling deploy causes brief per-instance pauses only
  total lag impact: seconds instead of tens of minutes
```

| Metric | Value | Note |
|---|---|---|
| Naive deploy | ~30 min | degraded |
| Cooperative | partial pauses | 19 keep running |
| Static membership | no rebalance | **on fast restart** |
| Standbys | seconds | state rebuild |

> **Static membership is the highest-value setting most teams have not enabled**  
> It makes a consumer restart within the session timeout a non-event: the same member id reclaims the same partitions with no reassignment at all. For rolling deploys — by far the most frequent cause of rebalances in practice — it eliminates the disruption entirely, and it costs nothing but a stable identifier per instance.

**When to use it**

- **Any partitioned stream consumed by more than one process**, which is essentially all production stream consumption.
- **Horizontal scaling of consumers**, where the group protocol distributes work automatically.
- **Fault tolerance**, since a failed member's partitions are reassigned without manual intervention.
- **Independent consumption of the same stream** by different applications, each in its own group.

**When to avoid it**

- **Do not run more consumers than partitions** — the excess sit idle and each one still participates in rebalances.
- **Do not do long-running work inside the poll loop**; hand it off or reduce batch size.
- **Do not raise timeouts to suppress rebalances**, which delays detection of real failures.
- **Do not ignore the revoke callback**, which is where offsets and partial state must be flushed.
- **Do not assume a rebalance is rare** — deploys, autoscaling and transient network issues all trigger them.

**Advantages**

- **Automatic work distribution** across members with no manual assignment.
- **Automatic failover**: a dead member's partitions are reassigned without intervention.
- **Elastic scaling** up to the partition count by adding or removing members.
- **Exactly one owner per partition**, which is what preserves per-partition ordering.
- **Independent groups** over the same topic, each scaling and failing separately.

**Disadvantages**

- **Rebalances pause processing**, and they happen far more often than teams expect.
- **Two timeouts with different semantics** create a confusing failure surface.
- **Stateful consumers must rebuild local state** on reassignment, which can dominate the pause.
- **Rebalance storms** are self-amplifying under slow processing.
- **Parallelism is capped by partition count**, so scaling has a hard ceiling.
- **Reprocessing after reassignment** means duplicates, requiring idempotency.

**Trade-offs**

**Assignment strategies**

| Strategy | Movement on change | Pause | Best for |
|---|---|---|---|
| Range | Large, uneven | Full | Legacy default; avoid |
| Round-robin | Large, even | Full | Stateless consumers |
| Sticky | Minimised | Full | Reducing state rebuild |
| Cooperative sticky | Minimised | Partial | The modern default |
| Static membership | None on fast restart | None | Rolling deploys |

Cooperative sticky assignment plus static membership together address the two most common causes of disruption — reassignment scope and restart-triggered rebalances — and both are configuration rather than code.

**How it fails**

**Consumer group failures**

| Symptom | Cause | Fix |
|---|---|---|
| Frequent rebalances with no deploys | Processing exceeding max poll interval | Smaller poll batches; async processing; check p99 batch time |
| Rebalance storm, throughput collapses | Cascading timeouts across members | Reduce work per poll; cooperative rebalancing; do not raise timeouts |
| Long lag spikes on every deploy | Eager rebalance plus state rebuild | Cooperative sticky; static membership; standby replicas |
| Duplicate processing after a rebalance | Uncommitted offsets reprocessed | Commit in the revoke callback; idempotent handlers |
| Some consumers idle | More consumers than partitions | Reduce consumers, or increase partitions |
| One consumer permanently behind | Hot partition assigned to it | Per-partition lag monitoring; re-key the topic |
| Consumer never rejoins after a network blip | Session timeout too aggressive | Tune session timeout to real network conditions, not to milliseconds |

**Limits**

> **Tuning guidance**
>
> - **Max poll interval** should comfortably exceed p99 batch processing time — aim for 3–5× headroom, achieved by reducing batch size rather than raising the timeout.
> - **Session timeout** of 10–45 s is typical; shorter means faster failure detection and more false positives.
> - **Offset commit interval** bounds duplicate work after reassignment — seconds, typically.
> - **Consumers ≤ partitions**, always; excess members are idle but still participate in rebalances.
> - **Standby replicas** turn a multi-minute state rebuild into seconds, at the cost of extra memory and changelog consumption.

**Alternatives**

| Approach | Assignment | Trade |
|---|---|---|
| Consumer group protocol | Automatic, dynamic | Rebalance pauses |
| Manual partition assignment | Static, explicit | No automatic failover; you own it |
| Single consumer per topic | Trivial | No parallelism |
| External coordination (lease-based) | Custom, fine-grained | You build and operate it |
| Queue with competing consumers | Per-message, no assignment | No ordering; no replay |

Manual assignment is a legitimate choice for small, stable deployments where ordering and state locality matter more than automatic failover — you give up rebalancing entirely and take responsibility for reacting to failures yourself.

**In real systems**

- **Kafka's cooperative sticky assignor** was introduced specifically because eager rebalancing made large stateful deployments impractical.
- **Static group membership** was added to make rolling restarts of large consumer fleets non-disruptive, which is the dominant real-world rebalance trigger.
- **Kafka Streams standby replicas** maintain warm copies of per-partition state so reassignment does not require a full changelog rebuild.
- **Flink's checkpointing model** takes a different approach entirely — periodic distributed snapshots rather than per-partition offset commits — trading a different set of costs.
- **Managed streaming platforms** increasingly hide the protocol, but the underlying constraints — one owner per partition, pauses on change — remain visible in their behaviour.

**Common mistakes**

- **Long processing inside the poll loop**, exceeding the max poll interval.
- **Raising timeouts to stop rebalances**, delaying detection of genuine failures.
- **Eager rebalancing on large stateful deployments**, making every deploy a lag incident.
- **No static membership**, so every rolling restart triggers a full reassignment.
- **Ignoring the revoke callback**, leaving offsets uncommitted and state unflushed.
- **More consumers than partitions**, adding rebalance participants for no throughput.
- **Aggregate lag monitoring only**, hiding a single stuck partition.

**The staff-level view**

Rebalancing is a configuration problem masquerading as an architectural one: the defaults are poor, the right settings are well known, and most teams discover them only after an incident.

- **Enable cooperative sticky assignment and static membership as platform defaults.** Together they eliminate most rebalance disruption and cost nothing but configuration.
- **Treat frequent rebalances as a processing-time defect**, not a tuning problem. The fix is smaller batches; raising timeouts trades a visible problem for a slower failure detection you will meet later.
- **Require per-partition lag monitoring**, since aggregate lag hides the stalled partition that is almost always the real issue.
- **Plan for state rebuild in stateful consumers.** Standby replicas turn a minutes-long reassignment into seconds and are the difference between a tolerable deploy and a lag incident.
- **Make idempotency mandatory**, because reassignment reprocesses everything since the last commit and that window is unavoidable.

**Go deeper**

A consumer group distributes partitions so each is owned by exactly one member, preserving per-partition ordering while allowing parallelism, and reassigns automatically when a member joins or dies. Parallelism is capped at the partition count; extra consumers sit idle while still participating in every rebalance.

Two independent deadlines govern liveness. Session timeout detects a crashed or partitioned consumer via missing heartbeats. Max poll interval detects a consumer that is alive but stuck on a long batch — and this is the one that catches teams out, because a consumer doing legitimate slow work is ejected mid-task and its work reprocessed. The correct response is smaller poll batches, never a longer timeout, since raising it delays detection of genuinely dead consumers by the same amount.

Rebalances pause processing, and under classic eager assignment they stop the entire group while every partition is revoked and reassigned. Cooperative sticky assignment reduces this to only the partitions that must move; static membership means a restart within the session timeout triggers no rebalance at all; and standby replicas turn a minutes-long state rebuild into seconds. Those three settings convert a rolling deploy from tens of minutes of degradation into brief per-instance pauses.

Consumer groups exist to satisfy one invariant — exactly one consumer owns each partition at any moment — because violating it would break the ordering guarantee that partitioning exists to provide. Every operational difficulty follows from maintaining that invariant across membership changes.

**The protocol.** Members heartbeat to a coordinator; when membership changes, partitions are reassigned. Under the classic eager protocol, all members revoke all partitions and then receive new assignments, which means the whole group is idle for the duration. Cooperative incremental rebalancing computes which partitions must actually move and revokes only those, so unaffected consumers keep processing throughout. Sticky assignment additionally prefers to leave each consumer with its existing partitions, which minimises both movement and, crucially, state rebuilding.

**Two timeouts, two failure modes.** Session timeout catches a crashed or network-partitioned consumer through missed heartbeats, which are typically sent from a background thread and therefore continue during processing. Max poll interval catches a consumer that is alive but has not fetched records within the deadline — a batch taking too long. The second causes most avoidable rebalances, and the instinctive fix of raising it is actively harmful: detection of a genuinely dead consumer is then delayed by the new timeout, during which its partitions go unprocessed. The correct fix is to shorten the work per poll, either by reducing batch size or by handing processing to a separate thread pool while the poll loop continues.

**Rebalance storms are self-amplifying.** A rebalance pauses processing, which lengthens in-flight work, which pushes other consumers past their deadlines, which triggers further rebalances. Throughput can collapse to near zero from a cause that was originally marginal, which is why rebalance frequency deserves to be a monitored indicator rather than background noise — a healthy group should rebalance only on deploys and scaling events.

**Stateful consumers change the economics.** When a partition moves, the new owner must rebuild any local state associated with that key range, typically by replaying a changelog, which can take minutes and dominates the pause entirely. Standby replicas — warm copies maintained on other instances — reduce that to seconds at the cost of extra memory and changelog consumption. For a large stateful deployment this is the difference between a tolerable deploy and a lag incident.

**The configuration that matters.** Cooperative sticky assignment, static group membership with a stable identifier per instance, standby replicas for stateful processors, poll batch sizes giving several times headroom against the interval, and offset commits frequent enough to bound reprocessing. None of these require code changes, all of them are well established, and most teams discover them only after an incident — which is precisely the argument for putting them into a platform-level consumer template rather than leaving each team to rediscover them. Finally, because reassignment reprocesses everything since the last commit, idempotent handling is not optional but structural.

**Prove it — interview questions**

1. **[Basic] What does a consumer group do?**

   <details><summary>Model answer</summary>

   It distributes a topic's partitions among a set of consumers so that each partition is owned by exactly one member, which is what preserves per-partition ordering while allowing parallel consumption. It also handles failure: a coordinator detects when a member stops heartbeating and reassigns its partitions to the survivors. Different applications use different groups over the same topic, each with its own offsets and its own parallelism.

   </details>

2. **[Basic] Why can't you have more consumers than partitions?**

   <details><summary>Model answer</summary>

   Because each partition is assigned to exactly one consumer within a group, so once every partition has an owner, additional members have nothing to do. They sit idle — and they still participate in every rebalance, so they add disruption without adding throughput. This makes partition count the parallelism ceiling, chosen when the topic is created and awkward to raise later because increasing it rehashes keys and breaks ordering.

   </details>

3. **[Senior] What is the difference between session timeout and max poll interval?**

   <details><summary>Model answer</summary>

   Session timeout detects a dead or partitioned consumer through missing heartbeats, which are usually sent by a background thread and therefore continue even while the application is processing. Max poll interval detects a consumer that is alive but stuck — one that has not requested new records within the deadline because a batch is taking too long. The second catches people out, because a consumer doing legitimate slow work is ejected mid-task, its partitions reassigned, and the work reprocessed elsewhere. The fix is to reduce work per poll rather than to raise the interval.

   </details>

4. **[Senior] Why is raising the timeout the wrong response to frequent rebalances?**

   <details><summary>Model answer</summary>

   Because it trades a visible problem for an invisible worse one. If rebalances are caused by processing exceeding the deadline, raising the deadline to twenty minutes means a consumer that genuinely dies goes undetected for twenty minutes, during which its partitions are unprocessed and lag grows unchecked. The correct fix is to make each poll's work comfortably short — fewer records per poll, or handing work to a separate thread pool while the poll loop continues — so the deadline has real headroom rather than being marginal.

   </details>

5. **[Staff] How do you make rolling deploys of a large stateful stream processor non-disruptive?**

   <details><summary>Model answer</summary>

   Three settings, all configuration rather than code. Cooperative sticky assignment so that a membership change moves only the partitions that must move, leaving the other consumers processing throughout instead of stopping the world. Static group membership with a stable member id per instance, so a consumer restarting within the session timeout reclaims its own partitions with no rebalance at all — which for rolling deploys, the dominant trigger in practice, eliminates the disruption entirely. And standby replicas for local state, so that when a partition genuinely does move, the new owner has a warm copy rather than replaying a changelog for minutes. Without these, a twenty-instance rolling deploy is twenty full rebalances plus twenty state rebuilds, which is tens of minutes of degraded processing; with them it is a sequence of brief per-instance pauses.

   </details>

6. **[Principal] Why do consumer groups cause so much operational pain relative to their conceptual simplicity?**

   <details><summary>Model answer</summary>

   Because the protocol's correctness requirement — exactly one owner per partition at all times — forces a coordination point, and every coordination point becomes a stop-the-world event unless deliberately designed otherwise. The defaults historically optimised for correctness and simplicity rather than for the common case, which is a rolling restart of a healthy fleet, so teams meet eager rebalancing and full state rebuilds at their worst moment. There is also a genuine feedback loop: a rebalance pauses processing, which lengthens in-flight batches, which causes more consumers to miss their deadlines, which triggers more rebalances. That self-amplification means the failure is not proportional to its cause. The organisational answer is to put the good configuration into the platform's consumer template rather than leaving each team to discover it, and to treat rebalance frequency as a monitored service-level indicator, because a healthy system should rebalance only on deploys and scale events — anything more is a processing-time defect that will eventually cascade.

   </details>

---

### Delivery semantics

*At-most-once loses, at-least-once duplicates, and exactly-once is a property of effects rather than of delivery — which is why idempotency is the real answer.*

**Flow:** `Deliver` → `Apply effect` → `Record progress` → `Acknowledge` → `Retry ambiguity`

> **The 30-second version**  
> At-most-once loses, at-least-once duplicates, and exactly-once delivery does not exist. Use at-least-once with an idempotency key recorded atomically with the effect.

**The problem**

A message is sent, an effect is applied, and progress is recorded. These are three separate operations, and a process can die between any two of them. Whatever ordering you choose, some failure leaves you unable to tell whether the effect happened — and the retry that follows either repeats it or skips it.

This is not an implementation weakness. It is a consequence of the Two Generals problem: two parties communicating over a lossy channel cannot achieve common knowledge of a fact. No protocol removes the ambiguity; it can only be moved.

> **The three regimes, and what each costs**  
> **At-most-once**: acknowledge first, then act. A crash loses the work silently — no duplicates, and no record that anything was missed. **At-least-once**: act first, then acknowledge. A crash redelivers, so effects can repeat. **Exactly-once delivery**: impossible over an unreliable network. What is achievable is exactly-once *effect*, by making repeated delivery harmless.

**Mental model**

Stop asking “how many times will this be delivered?” and start asking “what happens if it is delivered twice?” The first question has no satisfying answer; the second has an engineering solution.

1. **At-most-once** — Fire and forget, or acknowledge before processing. Appropriate only where loss is genuinely cheaper than duplication — some telemetry, some cache warming.
2. **At-least-once** — Acknowledge after processing. The default for anything that matters, and the reason idempotency is a universal requirement.
3. **Exactly-once processing** — Achievable *within* a closed transactional system — read, compute and write offsets atomically. Does not extend past that boundary.
4. **Exactly-once effect** — At-least-once delivery plus a deduplication key recorded atomically with the effect. This is what people actually want.
5. **Ambiguity** — A timeout is not a failure. The request may have succeeded and the response been lost, which is why retrying safely requires idempotency at the receiver.

> **Where the exactly-once claim is and is not true**  
> Systems advertising exactly-once semantics are describing transactional processing inside their own boundary: consume, transform and commit offsets in one atomic unit, so reprocessing cannot double-apply. That is real and useful. But the moment an effect leaves that boundary — charging a card, sending an email, calling a partner API — the transaction cannot cover it, and you are back to at-least-once plus idempotency.

**How it works**

**The three orderings and their failure windows**

```text
AT-MOST-ONCE
  ack()                <- progress recorded
  perform_effect()     <- crash here = work SILENTLY LOST
  no duplicates, no record that anything is missing

AT-LEAST-ONCE
  perform_effect()     <- crash here = redelivered, effect once
  ack()                <- crash here = redelivered, effect TWICE
  nothing lost, duplicates possible

EXACTLY-ONCE EFFECT (the practical target)
  BEGIN
    if seen(key): COMMIT; ack; return      <- already done
    perform_effect()                       <- must be in the txn
    record_seen(key)
  COMMIT
  ack()                <- crash here = redelivered, seen(key)
                          is true, so it is a no-op

the deduplication record and the effect MUST be atomic.
if they are separate writes, the window reopens.
```

1. **Choose the deduplication key carefully** — The message id works for transport-level duplicates. A business key — order id plus effect name — also catches duplicates produced upstream, which message ids do not.
2. **Record the dedupe key in the same transaction as the effect** — Two separate writes reintroduce the crash window you were closing. If the effect is in another system, use its idempotency mechanism instead.
3. **Give dedupe records a retention bound** — They cannot be kept forever. Retain longer than the maximum possible retry window — typically hours to days — and expire them.
4. **Store the result, not just the fact** — A retry of a completed operation should return the original response, not a conflict. Storing the response makes the retry genuinely indistinguishable from the first call.
5. **Push idempotency to the outermost boundary** — If the client can supply an idempotency key, duplicates caused by client retries are caught too — which is where many real duplicates originate.
6. **Do not confuse retries with duplicates** — A retry after a timeout is the *same* logical operation. A genuine duplicate is a second distinct request. Both are handled by the key, but the distinction matters when deciding what the key should be.

**Idempotency key design**

```text
TRANSPORT-LEVEL KEY (message id)
  catches: broker redelivery, consumer restart
  misses:  the producer publishing the same event twice
           (e.g. an application retry before the first
            publish was confirmed)

BUSINESS-LEVEL KEY (order_id + effect)
  catches: everything above, PLUS upstream duplicates
  example: "order:1234:confirmation_email"
  -> at most one confirmation email per order, no matter
     how many events describe it

CLIENT-SUPPLIED KEY (Idempotency-Key header)
  catches: everything above, PLUS client retries after a
           timeout where the response was lost
  -> this is the only level that makes the CLIENT's
     retry safe, which is where many duplicates start

RULE: put the key as far toward the origin of the
      operation as you can.
```

> **Ambiguous outcomes are the normal case, not the edge case**  
> When a call times out, the operation may have succeeded. The caller cannot know. Systems designed as if timeouts mean failure will double-charge, double-ship and double-send under exactly the conditions — network degradation — where retries are most frequent. Every write API that clients may retry needs an idempotency mechanism, and it is far cheaper to add at design time than to retrofit after a financial incident.

**Worked example**

A payment API, tracing every place a duplicate can originate and where it is caught.

**Duplicate sources and their defences**

```text
SOURCE 1  client retry after timeout
  client POSTs /payments, gets no response, retries
  -> WITHOUT a client key: two payments
  -> WITH Idempotency-Key: second request returns the
     first result verbatim

SOURCE 2  gateway or proxy retry
  an intermediary retries a 502
  -> same defence: the client's key travels with the request

SOURCE 3  internal message redelivery
  the payment service publishes payment.captured;
  the ledger consumer crashes after writing and before
  committing its offset
  -> ledger dedupes on (payment_id, "ledger_entry")

SOURCE 4  producer double-publish
  the payment service retries its publish after an
  ambiguous broker response
  -> two identical events, different message ids
  -> a transport-level key would NOT catch this
  -> a business key does

DESIGN
  client -> Idempotency-Key (24h retention, stores response)
  service -> business keys on every downstream effect
  consumers -> dedupe atomically with their own writes
```

| Metric | Value | Note |
|---|---|---|
| Client retries | Idempotency-Key | returns first result |
| Broker redelivery | business key | **catches more** |
| Dedupe retention | 24 h | > max retry window |
| Stored | the response | not just a flag |

> **Store the response, not a boolean**  
> A dedupe record saying “this happened” lets you avoid repeating the effect but leaves you unable to answer the retry properly — returning a conflict tells the client something went wrong when it did not. Storing the original response means a retry is genuinely indistinguishable from the first call, which is what the client needs in order to retry safely without special handling.

**When to use it**

- **At-least-once plus idempotency** as the default for anything whose loss matters, which is most things.
- **At-most-once** only where duplication is worse than loss and the data is genuinely disposable — high-volume metrics samples, cache warming hints.
- **Exactly-once processing** within a single transactional system, where read-process-commit can be atomic.
- **Client-supplied idempotency keys** on any write API that a client may retry — which is all of them.
- **Business-level keys** wherever upstream duplicates are possible, not just transport redelivery.

**When to avoid it**

- **Do not claim exactly-once across a system boundary**; it does not exist and the claim causes designs that omit idempotency.
- **Do not acknowledge before processing** unless loss is genuinely acceptable and stated as a requirement.
- **Do not write the dedupe record separately from the effect**, which leaves the window open.
- **Do not treat a timeout as a failure** — the operation may have succeeded.
- **Do not retain dedupe keys forever**, but do retain them longer than the longest possible retry.

**Advantages**

- **At-least-once plus idempotency is simple and robust**, needing no distributed agreement.
- **Idempotency keys compose across layers**, catching duplicates from clients, proxies, brokers and producers.
- **Storing the response makes retries transparent** to the caller, removing the need for client-side special cases.
- **Transactional processing within a boundary** genuinely removes duplicates for internal state changes.
- **The pattern is uniform**, so once a team internalises it, it applies to queues, APIs and event consumers alike.

**Disadvantages**

- **Every handler must be idempotent**, which is a discipline that is easy to skip and invisible when skipped.
- **Dedupe storage has a cost** in writes, space and retention management.
- **Choosing the key requires thought** — the wrong level catches some duplicates and misses others.
- **Atomicity between effect and dedupe record is not always possible**, particularly for external side effects.
- **Exactly-once claims in vendor documentation mislead teams** into omitting the work.

**Trade-offs**

**Semantics and their consequences**

| Semantics | Mechanism | Failure result | Use for |
|---|---|---|---|
| At-most-once | Ack before processing | Silent loss | Disposable telemetry |
| At-least-once | Ack after processing | Duplicate effects | The default |
| Exactly-once processing | Transactional read-process-commit | None, within the boundary | Stream processing into the same system |
| Exactly-once effect | At-least-once + atomic dedupe | None | Anything with external side effects |

> **The answer that shows understanding**  
> “Exactly-once delivery isn't achievable, so I'd use at-least-once with an idempotency key recorded in the same transaction as the effect. For the client-facing API I'd accept a client-supplied key and store the original response, so a retry after a timeout returns the same answer rather than creating a second payment.”

**How it fails**

**Delivery-semantics failures**

| Failure | Cause | Fix |
|---|---|---|
| Double charge | Client retried an ambiguous timeout with no idempotency key | Client-supplied key; store and return the original response |
| Work silently missing | Acknowledged before processing | Acknowledge after; alert on dead-letter and dedupe anomalies |
| Dedupe record written but effect not applied | Two separate non-atomic writes in the wrong order | Single transaction; effect first if atomicity is impossible |
| Duplicate despite dedupe | Key at transport level missed an upstream double-publish | Use a business-level key |
| Retry returns 409 and the client gives up | Dedupe stores a flag, not the response | Store the original response and replay it |
| Dedupe table grows unbounded | No retention policy | TTL longer than the maximum retry window |
| Exactly-once assumed, idempotency omitted | Vendor claim taken at face value | Confirm where the transactional boundary ends |

> **The non-atomic dedupe write**  
> Writing “processed” and then performing the effect loses work when the process dies between them, because the next delivery sees the key and skips. Performing the effect and then writing “processed” duplicates when it dies between them. Only a single atomic write of both is correct — and when the effect is external and cannot join your transaction, you must instead rely on that external system's own idempotency key, which is precisely why payment APIs offer one.

**Limits**

> **Practical guidance**
>
> - **Dedupe retention**: longer than the maximum retry window; 24 hours is a common default for client-facing keys.
> - **Key scope**: transport-level catches redelivery; business-level also catches upstream duplicates; client-supplied also catches client retries.
> - **Store the response**, not a boolean, so replays are transparent.
> - **Exactly-once processing** holds only where reads, writes and offset commits share one transaction.
> - **Treat every timeout as ambiguous** — assume roughly half of them succeeded.

**Alternatives**

| Approach | Handles | Cost |
|---|---|---|
| Idempotency key + atomic record | All duplicate sources it covers | Storage; discipline |
| Natural idempotency | Operations that are inherently repeatable | Only some operations qualify |
| Conditional write (version check) | Duplicate updates to one entity | Single-entity only |
| Transactional read-process-write | Duplicates within one system | Does not cross boundaries |
| Two-phase commit | Atomicity across systems | Latency; coordinator failure modes |
| Accept duplicates | Cases where they are harmless | Requires proving harmlessness |

Natural idempotency is worth seeking first: setting a field to a value, adding to a set, or writing a specific version are all inherently repeatable, while incrementing a counter or appending to a list are not. Reformulating an operation to be naturally idempotent removes the need for any key at all.

**In real systems**

- **Stripe's `Idempotency-Key` header** is the reference implementation of client-supplied keys with stored responses, precisely because double-charging is unacceptable.
- **Kafka's transactional producer and consumer** provide exactly-once processing within Kafka, which vendor documentation sometimes presents without the boundary caveat.
- **SQS's standard queues are at-least-once**, and its FIFO queues offer a deduplication window rather than an unconditional guarantee.
- **HTTP method semantics** encode this: `PUT` and `DELETE` are defined as idempotent, `POST` is not, which is why retry logic differs between them.
- **Payment processors universally require idempotency keys**, because ambiguous timeouts during network degradation are common and duplicate charges are not recoverable reputationally.

**Common mistakes**

- **Believing a vendor's exactly-once claim extends past its own boundary.**
- **Acknowledging before processing**, losing work with no trace.
- **Writing the dedupe record and the effect separately.**
- **Using a transport-level key** where upstream duplicates exist.
- **Returning a conflict on a retry** instead of the original response.
- **Treating a timeout as a failure**, then retrying a non-idempotent operation.
- **Unbounded dedupe storage**, or retention shorter than the retry window.

**The staff-level view**

The most valuable intervention here is vocabulary: teams that say “exactly-once” stop designing for duplicates, and the resulting bugs are financial.

- **Replace “exactly-once delivery” with “exactly-once effect” in design discussions.** The rename makes the required work — idempotency — visible rather than assumed.
- **Require idempotency keys on every write API** at the outermost boundary, with stored responses, so client retries after timeouts are safe by construction.
- **Make dedupe atomic with the effect a review checkpoint.** Two separate writes is the most common way a correct-looking implementation is wrong.
- **Prefer reformulating operations to be naturally idempotent** where possible, since that removes the mechanism entirely.
- **Audit vendor exactly-once claims for their boundary.** They are usually true and usually narrower than teams assume.

**Go deeper**

Delivering a message, applying an effect and recording progress are three separate operations, and a crash between any two leaves ambiguity. Acknowledge before processing and a crash loses the work silently; acknowledge after and a crash duplicates the effect. Exactly-once delivery cannot be achieved over an unreliable network — this is the Two Generals problem — so the practical target is exactly-once *effect*.

That means at-least-once delivery plus a receiver that recognises repeats: derive a stable key, check it, apply the effect and record the key, all in one transaction. Two separate writes reintroduce the window you were closing. Store the original response rather than a boolean so a retry returns the same answer and needs no client-side special handling.

Key placement matters. A message id catches broker redelivery; a business key such as order id plus effect name also catches a producer publishing twice; a client-supplied key also catches the most damaging case — a client retrying after a timeout where the response was lost. Vendor exactly-once claims are real but bounded to that system's own transactional boundary; anything crossing it, such as a card charge or partner API call, is back to at-least-once plus idempotency.

Delivery semantics look like a transport concern and are actually an application-design concern, because the transport cannot solve the problem and the application can.

**Why exactly-once delivery is impossible.** A sender that receives no acknowledgement cannot distinguish a lost message from a lost acknowledgement. Retrying risks duplication; not retrying risks loss. This is the Two Generals problem, and no protocol removes it — it can only be relocated. Consequently the three real options are at-most-once (acknowledge first, lose work silently on crash), at-least-once (acknowledge after, duplicate on crash), and at-least-once combined with a receiver that makes repeats harmless, which delivers exactly-once *effect*.

**The atomicity requirement.** Idempotency means checking a key, applying the effect, and recording the key. If the record is written before the effect, a crash between them causes the next delivery to skip work that never happened. If it is written after, a crash between them duplicates. Only a single transaction covering both is correct. When the effect is external and cannot join your transaction — a card charge, a partner API — you must delegate to that system's own idempotency mechanism, passing your key through, which is exactly why payment providers offer one.

**Key placement determines coverage.** A transport-level key such as the message id catches broker redelivery and consumer restarts. It does not catch a producer that published the same logical event twice after an ambiguous broker response, because those carry different message ids; a business-level key like `order:1234:confirmation_email` does. Neither catches a client retrying after a timeout where the response was lost, which is where a large share of real duplicates originate — only a client-supplied key can. The rule is to place the key as close to the origin of the operation as you can reach.

**Store the response, not a flag.** A dedupe record that merely asserts “this happened” forces the server to answer a retry with a conflict, which misinforms the client and requires special-case handling on their side. Storing and replaying the original response makes a retry genuinely indistinguishable from the first call, which is the property that lets clients retry freely — and retrying freely is the behaviour you want, because timeouts are ambiguous.

**Reformulation beats mechanism.** Some operations are naturally idempotent: setting a field to a value, adding to a set, writing a specific version. Others are not: incrementing a counter, appending to a list. Where an operation can be restated in the naturally idempotent form, no key, no storage and no retention policy is needed at all. That is always worth attempting before reaching for dedupe infrastructure.

**The vocabulary problem is real.** Vendor documentation advertising exactly-once semantics is usually accurate about processing within that system's transactional boundary and silent about what crosses it. Teams read the claim, conclude idempotency is unnecessary, and meet duplicate external effects during their first network incident. Insisting on the phrase “exactly-once effect”, and asking in review where the transactional boundary ends and what crosses it, prevents a class of expensive and entirely predictable failures.

**Prove it — interview questions**

1. **[Basic] Why is exactly-once delivery impossible?**

   <details><summary>Model answer</summary>

   Because the sender cannot distinguish between a lost message and a lost acknowledgement. If it retries, it may duplicate; if it does not, it may lose. This is the Two Generals problem: two parties over an unreliable channel cannot reach common knowledge of a fact. What is achievable is exactly-once *effect* — at-least-once delivery combined with a receiver that recognises and ignores repeats — which moves the problem from the transport to the application where it can actually be solved.

   </details>

2. **[Basic] How do you make a handler idempotent?**

   <details><summary>Model answer</summary>

   Derive a stable key for the operation, check whether it has already been applied, and if not, perform the effect and record the key — in a single atomic transaction. On redelivery the key is found and the handler returns without repeating. The atomicity matters: recording first and then acting loses work on a crash between them, while acting first and then recording duplicates on a crash between them. Only one transaction covering both is correct.

   </details>

3. **[Senior] Where should the idempotency key come from?**

   <details><summary>Model answer</summary>

   As close to the origin of the operation as possible. A transport-level key such as the message id catches broker redelivery and consumer restarts, but not a producer that published the same event twice after an ambiguous response. A business-level key like order id plus effect name catches those too. And a client-supplied key catches the case that causes the most damage in practice — a client retrying after a timeout where the response was lost, which no server-side key can distinguish from a genuine second request. For any write API that clients may retry, the key has to come from the client.

   </details>

4. **[Senior] Why store the response rather than just a flag?**

   <details><summary>Model answer</summary>

   Because a retry should be indistinguishable from the first call. If the dedupe record only says “this happened”, the best the server can do is return a conflict, which tells the client something went wrong when in fact the operation succeeded — and the client then has to implement special handling to interpret that. Storing the original response and replaying it verbatim means the client's retry logic needs no special case at all, which is the entire point of offering idempotency.

   </details>

5. **[Staff] What does a system mean when it claims exactly-once semantics?**

   <details><summary>Model answer</summary>

   It means exactly-once processing within its own transactional boundary: it can consume a message, apply a state change to its own storage, and commit the consumer offset as a single atomic unit, so reprocessing after a failure cannot double-apply. That is genuine and valuable for stream processing where the output goes back into the same system. What it cannot cover is any effect outside that boundary — charging a card, calling a partner, sending an email — because those cannot participate in the transaction. The failure mode I watch for is teams reading the claim, concluding that idempotency is unnecessary, and then discovering duplicate external effects during the first network incident. So in review I ask specifically where the transactional boundary ends and what crosses it.

   </details>

6. **[Principal] How do you make correct delivery handling the default across an organisation?**

   <details><summary>Model answer</summary>

   Partly through vocabulary and partly through platform defaults. The vocabulary matters more than it sounds: once a team says “exactly-once”, they stop designing for duplicates, so I insist on “exactly-once effect” and on naming the mechanism that achieves it. On the platform side, the shared consumer framework should take a key extractor and handle the check-and-record atomically, so idempotency is inherited rather than remembered — this is the single highest-consequence bug class in async systems and the least reliably implemented. API frameworks should support a client idempotency key with stored responses as a first-class feature, with retention configured centrally. And review should treat two separate writes for effect and dedupe record as a defect, because it is the most common way an implementation that looks correct is not. Underlying all of it is one cultural point: a timeout is ambiguous, not a failure, and engineers who internalise that design differently from those who do not.

   </details>

---

### Dead-letter queues

*Messages that cannot be processed get moved aside after bounded retries, so one poison message cannot block a queue or churn forever.*

**Flow:** `Delivery failure` → `Retry policy` → `Quarantine` → `Repair` → `Controlled redrive`

> **The 30-second version**  
> After bounded retries, move unprocessable messages aside with their failure context so the queue keeps moving — then alert on the rate, fix the cause, and redrive deliberately.

**The problem**

A message arrives whose payload references a deleted entity, or whose schema predates a field the handler now requires, or that triggers a bug. Processing fails. The queue redelivers. Processing fails again. Without a bound, this repeats forever, consuming capacity, filling logs, and — in an ordered stream — blocking everything behind it.

Retries assume failures are transient. Some are not. A dead-letter queue is the mechanism that distinguishes the two: after N attempts, the message is moved aside rather than retried again, so the system makes progress and the problem becomes visible.

> **The core distinction**  
> **Transient failures** — a timeout, a rate limit, a restarting dependency — succeed on retry. **Deterministic failures** — malformed data, a missing referent, a code bug — never will. Retry policy exists for the first; the dead-letter queue exists for the second. A system that only has retries treats every failure as transient and will churn on the ones that are not.

**Mental model**

Think of it as a quarantine with an owner. Messages that repeatedly fail are isolated so they stop affecting healthy traffic, preserved so nothing is lost, and surfaced so someone investigates.

1. **Retry policy** — Bounded attempts with exponential backoff and jitter. The bound is what makes dead-lettering possible at all.
2. **Dead-letter destination** — A separate queue or topic holding failed messages plus the metadata needed to diagnose them.
3. **Context** — Original payload, failure reason, attempt count, timestamps, trace id. Without these the quarantine is a pile of unexplainable messages.
4. **Alerting** — A rising dead-letter rate is always a code or data defect, never a capacity issue. It needs an owner and a page or ticket.
5. **Redrive** — The controlled process of replaying repaired messages back into the main queue after the underlying problem is fixed.

> **An unmonitored dead-letter queue is silent data loss**  
> The whole point is to make failures visible, and a dead-letter queue nobody watches achieves the opposite: it makes them invisible while providing the comforting sense that they are handled. Messages accumulate, retention expires, and work that was supposed to happen simply did not. Every dead-letter queue needs a named owner and an alert on its rate, or it is a slower path to the same loss.

**How it works**

**Retry and dead-letter flow**

```text
message delivered
  |
  v
handler fails
  |
  +-- attempt < max?  -> requeue with backoff
  |                      (1s, 2s, 4s, 8s, 16s, with jitter)
  |
  +-- attempt >= max? -> publish to DLQ with metadata:
                           { original_message,
                             error_type, error_message,
                             attempt_count,
                             first_failed_at, last_failed_at,
                             trace_id, consumer_version }
                         then ACK the original
                         -> the main queue makes progress

LATER
  an engineer inspects the DLQ, groups by error type,
  identifies the cause, deploys a fix, and REDRIVES
  the affected messages back into the main queue.
```

1. **Classify failures before retrying** — A 400-class error will never succeed on retry and should dead-letter immediately; a 503 or timeout should retry. Retrying deterministic failures wastes the retry budget and delays the alert.
2. **Capture enough context to diagnose without reproduction** — The error, the attempt history, the trace id and the consumer version. A DLQ message you cannot explain is a DLQ message nobody will act on.
3. **Alert on rate, not depth** — Depth grows slowly and is easy to ignore; a step change in rate is the signal that something just broke, usually a deploy or an upstream schema change.
4. **Build redrive as a first-class, rate-limited operation** — Replaying ten thousand messages at once after a fix will overwhelm the very dependency that was failing. Redrive with a rate limit and the ability to select a subset.
5. **Give each consumer its own DLQ** — A shared dead-letter queue makes ownership ambiguous and mixes unrelated failures, which is how they end up unowned.
6. **Set retention long enough to act** — Dead-lettered messages expiring before anyone looks is the loss the mechanism exists to prevent. Days at minimum.

**Failure classification drives the policy**

```text
IMMEDIATE DEAD-LETTER (never retry)
  schema validation failure
  referenced entity permanently deleted
  authorisation permanently denied
  payload deserialisation error
  -> retrying cannot change the outcome

RETRY WITH BACKOFF
  dependency timeout / 503
  rate limited (429) - honour Retry-After
  optimistic concurrency conflict
  transient database error
  -> the same input may succeed later

RETRY, THEN DEAD-LETTER
  anything unclassified
  -> bounded attempts, then quarantine

WHY CLASSIFICATION MATTERS
  without it, a malformed payload consumes 10 retries over
  5 minutes before anyone learns about it, and a genuine
  outage's messages dead-letter too early.
```

> **Redrive without a fix recreates the problem**  
> Replaying dead-lettered messages before the underlying cause is resolved simply sends them back around the retry loop and into the dead-letter queue again, consuming capacity and obscuring the original failure timestamps. Redrive is the last step after a fix is deployed and verified, not a first response to a growing queue.

**Worked example**

A dead-letter queue filling after a deploy, worked through as an incident.

**From alert to redrive**

```text
ALERT  order-events DLQ rate: 0/min -> 340/min at 14:02

TRIAGE (group by error type)
  338  NullPointerException in OrderHandler.applyDiscount
    2  ValidationError: unknown currency "XBT"

DIAGNOSIS
  14:01 deploy added a discount field consumers must read.
  Orders published BEFORE the deploy lack the field;
  the new handler assumes it is present.
  -> a deterministic failure for a bounded set of messages

IMMEDIATE ACTION
  roll back the consumer (not the producer) -> rate returns
  to 0; new messages process with the old handler

FIX
  handler tolerates a missing discount field (default 0),
  which is the compatibility rule that should have applied
  to the original change

REDRIVE
  338 messages, rate-limited to 20/s
  verify the first 10 succeed before releasing the rest
  confirm DLQ depth returns to 2

REMAINING 2
  genuinely invalid currency from a partner integration
  -> not redrivable; route to the partner team as a data
     issue and delete after confirmation
```

| Metric | Value | Note |
|---|---|---|
| Alert | rate change | not depth |
| Triage | group by error | **338 vs 2** |
| Fix first | then redrive | never reverse |
| Redrive | rate-limited | verify a sample |

> **The dead-letter queue is a debugging surface, not just a bin**  
> Grouping by error type turned 340 messages into two distinct problems with two different owners in under a minute. That only worked because the messages carried the exception, the handler version and a trace id. A dead-letter queue containing bare payloads would have required reproducing each failure by hand — which is why capturing context is not optional metadata but the feature itself.

**When to use it**

- **Every queue and every stream consumer**, without exception. A queue with no dead-letter path will eventually churn on a poison message.
- **Anywhere retries are bounded**, since a bound implies a destination for messages that exhaust it.
- **Schema-evolving systems**, where producer and consumer versions can diverge and produce deterministic failures.
- **Integrations with third parties**, whose data will eventually violate your assumptions.
- **As a debugging surface** during incidents, grouping failures by type to separate distinct problems.

**When to avoid it**

- **Do not create a dead-letter queue without an owner and an alert** — it becomes silent data loss with extra steps.
- **Do not redrive before fixing the cause**, which recycles the messages straight back.
- **Do not redrive at full speed**, which overwhelms the dependency that was already failing.
- **Do not share one dead-letter queue across consumers**, which makes ownership ambiguous.
- **Do not retry deterministic failures**, which wastes the budget and delays the alert.

**Advantages**

- **The main queue makes progress** regardless of individual message failures.
- **Nothing is lost**, unlike simply dropping failed messages.
- **Failures become visible and attributable**, which is what turns them into fixable defects.
- **Grouping by error type is a fast triage tool** that separates distinct problems immediately.
- **Redrive means a fix can recover the affected work** rather than abandoning it.

**Disadvantages**

- **Requires active ownership**; an unwatched dead-letter queue is worse than none because it creates false confidence.
- **Redrive is an operational procedure** that must be built, rate-limited and tested.
- **Ordering is broken for dead-lettered messages** — they are reprocessed out of sequence relative to their stream.
- **Retention must be managed**, or the messages expire before anyone acts.
- **Context capture adds work** to the consumer, and without it the queue is not useful.

**Trade-offs**

**Handling a message that cannot be processed**

| Option | Progress | Data preserved | Visibility |
|---|---|---|---|
| Retry forever | Blocked (ordered) or churning | Yes | Buried in logs |
| Drop and log | Yes | No | Log only — usually missed |
| Dead-letter queue | Yes | Yes | Explicit, alertable |
| Dead-letter with no alerting | Yes | Temporarily | None — silent loss |
| Halt the consumer | No | Yes | Very high — an outage |

The last row is a legitimate choice for a small class of systems: financial reconciliation pipelines sometimes prefer to stop entirely rather than proceed past a message they cannot process, because silent progress past an unexplained failure is worse than downtime. That should be a deliberate decision, not a default.

**How it fails**

**Dead-letter queue failures**

| Failure | Cause | Fix |
|---|---|---|
| Messages expire unnoticed | No alert, retention shorter than response time | Alert on rate; retention measured in days |
| Redrive causes a second incident | Full-speed replay into a still-failing dependency | Rate-limited redrive; verify a sample first |
| Cannot diagnose dead-lettered messages | No error context captured | Store exception, attempt history, trace id, consumer version |
| Ownership unclear | Shared DLQ across consumers | One dead-letter destination per consumer |
| Deterministic failures burn retry budget | No failure classification | Immediate dead-letter for non-retryable error classes |
| Ordering violated after redrive | Messages reprocessed out of sequence | Accept it, or redrive per key in original order |
| Dead-letter queue itself fills the broker | Unbounded growth from a persistent defect | Alert on rate early; cap and page rather than accumulate |

**Limits**

> **Operating parameters**
>
> - **Retry attempts**: 3–10 with exponential backoff and jitter, depending on the cost of the operation.
> - **Dead-letter retention**: days to weeks — long enough for a human to respond and redrive.
> - **Alert on rate change**, not on absolute depth; a sustained non-zero rate is a defect.
> - **Redrive rate**: well below the main queue's normal throughput, and verified on a sample first.
> - **Steady-state dead-letter rate should be zero**; anything else indicates a data or code problem awaiting attention.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Dead-letter queue | The general answer | Needs an owner and a redrive path |
| Retry forever | Never appropriate in practice | Churn; blocking in ordered streams |
| Drop with metrics | Genuinely disposable data | Loss; only acceptable if stated |
| Park in a database table | Rich querying and manual repair | Bespoke tooling |
| Halt the consumer | Correctness-critical pipelines | Availability cost |
| Compensating action | Failures with a meaningful fallback | Requires a sensible default |

Parking failures in a database table rather than a queue is underrated for complex cases: it allows querying by error type, customer, or time, supports manual repair of payloads, and makes redrive a controlled selection rather than an all-or-nothing replay.

**In real systems**

- **SQS redrive policies** make dead-lettering declarative — a maximum receive count and a destination queue — plus a built-in redrive operation.
- **Kafka has no native dead-letter concept**, so consumers must implement it explicitly by publishing to an error topic and committing the offset, which is a frequent omission.
- **Debezium and CDC pipelines** route unprocessable change events to error topics, since a single malformed record would otherwise block the whole stream.
- **Payment and billing pipelines** typically park failures in a database with rich metadata, because each one may need individual human judgement.
- **Schema-registry-enforced topics** reduce dead-letter volume substantially by rejecting incompatible messages at publish time rather than at consume time.

**Common mistakes**

- **No dead-letter queue at all**, so poison messages churn or block.
- **No alert on the dead-letter queue**, turning it into silent loss.
- **No error context**, making the messages undiagnosable.
- **Redriving before fixing**, recycling failures and obscuring timestamps.
- **Full-speed redrive**, causing a second incident.
- **Retrying deterministic failures**, wasting budget and delaying detection.
- **Retention shorter than the time it takes anyone to notice.**

**The staff-level view**

The dead-letter queue is the difference between a system that degrades and one that stops — and between failures that are fixed and failures that are quietly abandoned.

- **Require a dead-letter destination, an owner and a rate alert for every consumer**, enforced in the platform's consumer template rather than left to each team.
- **Standardise the failure context payload** — error, attempts, trace id, consumer version — so triage is uniform and grouping by error type works everywhere.
- **Build redrive as shared tooling** with rate limiting and subset selection, because every team otherwise improvises it during an incident.
- **Treat a non-zero steady-state dead-letter rate as a defect**, not as background noise. It is always a code or data problem.
- **Classify failures explicitly.** Retrying a schema validation error ten times delays the alert by minutes and teaches the team that retries are meaningless.

**Go deeper**

Retries assume failures are transient, and some are not: a malformed payload, a deleted referent or a code bug will fail identically every time. Without a bound, those messages churn forever or, in an ordered stream, block everything behind them. A dead-letter queue takes them out of the path after N attempts, preserves them with their failure context, and acknowledges the original so the system makes progress.

Two things make it work. Failure classification, so deterministic errors dead-letter immediately rather than burning the retry budget and delaying the alert, while timeouts and rate limits retry with jittered backoff. And context capture — exception, attempt history, trace id, consumer version — because a queue of bare payloads is undiagnosable and therefore unactioned. Grouping by error type is what turns hundreds of failures into two distinct problems in under a minute.

The critical operational point is that an unmonitored dead-letter queue is silent data loss with the added harm of false confidence. Every one needs a named owner and an alert on rate change, not depth. And redrive must come after the fix, rate-limited and verified on a sample — replaying into an unfixed system simply recycles the failures and destroys the original timestamps.

A dead-letter queue is the mechanism that separates transient failures, which retries resolve, from deterministic ones, which retries only amplify.

**Why bounded retries require a destination.** Retrying forever either churns capacity indefinitely or, in an ordered stream where there is no per-message acknowledgement, blocks every subsequent message in the partition. Dropping instead loses data with only a log line that nobody reads. The dead-letter queue provides the third option: quarantine the message with enough context to diagnose it, acknowledge the original so the pipeline progresses, and make the failure explicit enough that someone acts.

**Classification comes first.** Schema validation errors, deserialisation failures, permanently deleted referents and permanent authorisation denials cannot succeed on retry, so retrying them ten times over several minutes only delays the alert while consuming capacity. Timeouts, 503s, rate limits and optimistic concurrency conflicts may well succeed, so they should retry with exponential backoff and jitter. Anything unclassified retries a bounded number of times and then quarantines. Without this split, teams learn that retries are meaningless and alerts are late.

**Context is the feature.** The dead-lettered payload alone is nearly useless. What makes triage possible in seconds is the exception type and message, the attempt count with first and last failure timestamps, a trace id linking to the originating request, and the consumer version. That last field is disproportionately valuable, because a step change in dead-letter rate almost always coincides with a deploy, and knowing which version failed collapses the search space immediately. Grouping by error type then separates a single alert into distinct problems with distinct owners.

**Redrive is a procedure, not a button.** It must follow the fix, because replaying into an unfixed system recycles the messages and overwrites the original failure timestamps that would have explained them. It must be rate-limited well below normal throughput, since the dependency that failed may still be fragile and a bulk replay will re-break it. It must support subset selection, because a mixed dead-letter queue contains unrelated problems. And the first few should be verified before releasing the rest. Building this once as shared tooling is far better than each team improvising it under incident pressure.

**The failure mode of the mechanism itself.** An unmonitored dead-letter queue produces exactly the loss it was built to prevent, with the additional harm of creating confidence that failures are handled. Messages accumulate, retention expires, and work that should have happened simply did not. So the platform's consumer configuration should make a dead-letter destination, a named owning team and a rate-change alert mandatory rather than optional, with retention sized against realistic human response time. The framing worth repeating is that a dead-letter queue with no owner is worse than none — and that a non-zero steady-state dead-letter rate is always a code or data defect, never a capacity problem.

**Prove it — interview questions**

1. **[Basic] What is a dead-letter queue for?**

   <details><summary>Model answer</summary>

   For messages that cannot be processed after a bounded number of attempts. Rather than retrying forever — which churns capacity and, in an ordered stream, blocks everything behind it — the message is moved to a separate destination with its failure context, and the original is acknowledged so the main queue makes progress. The data is preserved, the failure becomes visible, and someone can fix the cause and replay the affected messages.

   </details>

2. **[Basic] Why classify failures before retrying?**

   <details><summary>Model answer</summary>

   Because retries only help with transient failures. A timeout, a 503 or a rate limit may succeed on the next attempt. A schema validation error, a deleted referent or a deserialisation failure never will, so retrying it ten times over five minutes simply delays the alert while consuming capacity. Classifying lets deterministic failures dead-letter immediately, which surfaces the problem faster and keeps the retry budget for failures that can actually benefit from it.

   </details>

3. **[Senior] What must a dead-lettered message carry?**

   <details><summary>Model answer</summary>

   Enough to diagnose it without reproducing the failure: the original payload, the exception type and message, the attempt count with first and last failure timestamps, a trace id linking to the wider request context, and the consumer version that failed. That last one matters more than people expect, because a step change in dead-letter rate usually coincides with a deploy, and knowing which version failed immediately narrows the cause. Without this metadata the queue is a pile of payloads that nobody can explain and therefore nobody acts on.

   </details>

4. **[Senior] How should redrive work?**

   <details><summary>Model answer</summary>

   Only after the cause is fixed and deployed, because replaying into an unfixed system just cycles the messages back and destroys the original failure timestamps. It should be rate-limited well below normal throughput, since the dependency that was failing may still be fragile and ten thousand replayed messages at once will overwhelm it. It should support selecting a subset, so a mixed dead-letter queue containing two unrelated problems can be redriven separately. And the first few should be verified before releasing the rest, because a fix that works in testing and not in production is exactly what redrive will discover at scale.

   </details>

5. **[Staff] Your dead-letter rate jumps from zero to 340 per minute. Walk through your response.**

   <details><summary>Model answer</summary>

   First, group by error type, because a step change usually contains one dominant cause plus noise, and separating them immediately gives distinct problems with distinct owners. Then correlate with deploys, since a sudden onset almost always coincides with a code or schema change — in practice the most common cause is a consumer that now requires a field older messages do not have, which is a compatibility rule that should have been applied to the original change. The immediate action is to stop the bleeding, usually by rolling back the consumer rather than the producer, so new messages process correctly while the fix is prepared. Then fix the handler to tolerate the missing field, deploy, and redrive the affected messages rate-limited with a verified sample. The residual messages that fail for genuinely different reasons — malformed partner data, for example — are not redrivable and need to go to whoever owns that integration as a data issue.

   </details>

6. **[Principal] How do you ensure dead-letter queues do not become silent data loss?**

   <details><summary>Model answer</summary>

   By making ownership and alerting structural rather than optional. The consumer framework should refuse to configure a queue without a dead-letter destination, an owning team and an alert on rate change, so the absence of monitoring is impossible rather than merely discouraged. The alert should fire on rate, not depth, because depth accumulates slowly enough to be normalised while a rate change is unambiguous and coincides with the cause. Retention should be measured in days, sized against realistic human response time rather than storage cost, since expiry is precisely the loss the mechanism exists to prevent. And redrive should be shared tooling with rate limiting and subset selection, because every team otherwise improvises it under pressure during an incident, which is when mistakes are most expensive. The framing I use is that a dead-letter queue with no owner is worse than no dead-letter queue at all — it produces the same loss while creating confidence that failures are being handled.

   </details>

---

### Transactional outbox

*Write the event to a table in the same transaction as the state change, then relay it to the broker — turning an impossible dual write into one atomic write plus a retry.*

**Flow:** `Business transaction` → `Outbox row` → `Relay` → `Event broker` → `Consumer`

> **The 30-second version**  
> Insert the event into an outbox table in the same transaction as the state change, and let a relay publish it. Atomicity without distributed transactions, at the cost of asynchronous delivery.

**The problem**

An order service commits an order to its database and publishes an `OrderPlaced` event. These are two systems and two operations, and there is no transaction spanning them. A crash between the commit and the publish means the order exists but nobody downstream learns about it. Publishing first and then committing means an event announcing an order that does not exist.

Neither ordering is safe, and neither failure is detectable. The order service sees a successful commit; the broker sees nothing; the consumer sees nothing; and the inconsistency surfaces weeks later as a missing shipment or an unreconciled ledger.

> **The dual-write problem is the defining correctness bug of event-driven systems**  
> It is silent on both sides, it occurs only during crashes and network partitions, it cannot be reproduced in testing, and its effects are discovered downstream and long afterwards. Retrying the publish does not fix it, because the process that would retry is the one that died.

**Mental model**

Make the event part of the state it describes. If the event is a row in the same database, written in the same transaction, then either both exist or neither does — and a separate process can deliver it later, retrying as long as necessary.

1. **Business transaction** — Write the order and the outbox row atomically. This is the only step that must be exactly right.
2. **Outbox table** — A durable queue inside your database: event id, aggregate id, type, payload, created_at, and a published marker or offset.
3. **Relay** — A separate process reading unpublished rows and delivering them to the broker, marking or deleting on success.
4. **At-least-once delivery** — The relay may publish and crash before marking, so the same event can be delivered twice. Consumers must be idempotent.
5. **Ordering** — Rows can be relayed in insertion order per aggregate, which preserves per-entity ordering downstream.

> **You have not removed the two-system problem — you have moved it**  
> The event still has to get from your database to the broker, and that step can still fail. What changed is that the failure is now *recoverable*: the event is durably recorded, so the relay simply retries until it succeeds. The dual write became an atomic write plus an at-least-once delivery, which is a problem with a known solution.

**How it works**

**The outbox write and the relay**

```text
BUSINESS TRANSACTION (atomic)
  BEGIN
    INSERT INTO orders (...) VALUES (...);
    INSERT INTO outbox (id, aggregate_id, type, payload,
                        created_at)
      VALUES (uuid, order_id, 'OrderPlaced', json, now());
  COMMIT
  -> either both rows exist, or neither does

RELAY (separate process, at-least-once)
  loop:
    rows = SELECT * FROM outbox
            WHERE published_at IS NULL
            ORDER BY id
            LIMIT 100
            FOR UPDATE SKIP LOCKED;
    for row in rows:
        broker.publish(row.type, row.payload, key=row.aggregate_id)
        UPDATE outbox SET published_at = now() WHERE id = row.id;
    COMMIT

CRASH BETWEEN PUBLISH AND UPDATE
  -> the row is still unpublished -> relayed again
  -> duplicate delivery -> consumers must be idempotent
```

1. **Use the event id as the deduplication key** — The outbox row's id travels with the event, so a consumer can recognise a redelivery even though the broker assigned a new message id.
2. **Partition by aggregate id** — Publishing with the aggregate id as the partition key preserves per-entity ordering, which is usually the ordering guarantee that matters.
3. **Prefer log-based relay at scale** — Polling the outbox table costs queries and adds latency. Reading the database's replication log via change data capture picks up outbox inserts with no polling and no additional load.
4. **Delete or archive published rows** — An outbox that grows forever becomes a large table with an index nobody prunes. Delete after publishing, or move to an archive partition.
5. **Keep the payload self-contained** — The relay publishes what the row holds. If the event only carries an id, consumers must call back — reintroducing coupling the outbox was meant to decouple.
6. **Monitor relay lag** — The age of the oldest unpublished row is the delay between a state change and the world learning about it, and it is the metric that matters.

**Polling relay versus log-based relay**

```text
POLLING RELAY
  SELECT ... WHERE published_at IS NULL ... SKIP LOCKED
  + simple; no extra infrastructure
  + works with any database
  - query load proportional to polling frequency
  - latency bounded by the poll interval
  - the outbox table takes both inserts and updates,
    which means index churn and vacuum/compaction load

LOG-BASED RELAY (CDC)
  a connector reads the database's replication log and
  publishes every outbox INSERT it sees
  + no polling, no query load, sub-second latency
  + no UPDATE needed - the insert itself is the signal
  + outbox rows can be deleted immediately or never read
  - a CDC pipeline to operate
  - ordering and offset semantics to understand

AT SCALE, LOG-BASED WINS
  the polling relay's UPDATE-per-row is the part that
  stops scaling; CDC removes it entirely.
```

> **The outbox does not make delivery exactly-once**  
> The relay can publish an event and crash before recording that it did, so the same event is published again with a different broker message id. Consumers therefore need idempotency keyed on the *event id from the outbox row*, not on the broker's message id. Teams that adopt the outbox and skip consumer idempotency have solved the publishing half and left the consuming half broken.

**Worked example**

Adding the outbox to an existing order service that currently dual-writes.

**Migration and what each step buys**

```text
BEFORE
  @Transactional
  placeOrder():
    orderRepo.save(order)      -- committed
    broker.publish(event)      -- outside the transaction
  -> crash between = lost event, silently

STEP 1  add the outbox table and write to it
  @Transactional
  placeOrder():
    orderRepo.save(order)
    outboxRepo.save(event)     -- same transaction
  -> atomicity achieved; nothing published yet

STEP 2  relay publishes, consumers still see the old path
  run the relay; consumers now receive events twice
  (once from the old direct publish, once from the relay)
  -> consumers must ALREADY be idempotent for this to be safe
  -> if they are not, this step exposes that, which is
     valuable to learn now rather than during an incident

STEP 3  remove the direct publish
  only the relay publishes
  -> single path, atomic, at-least-once

STEP 4  switch the relay to CDC
  remove polling load and the published_at UPDATE
  -> latency drops, database load drops

MEASURABLE OUTCOME
  events lost per crash: some -> zero
  publish latency: synchronous -> ~50-500ms (poll) or
                                  ~10-50ms (CDC)
```

| Metric | Value | Note |
|---|---|---|
| Lost events | zero | atomicity |
| Added latency | 10–500 ms | relay delay |
| Consumer change | idempotency | **required** |
| At scale | CDC relay | no polling |

> **The outbox trades synchronous publish latency for correctness**  
> Before, the event was published in-line and appeared immediately; after, it appears when the relay gets to it — tens to hundreds of milliseconds later. That delay is the price of never losing an event, and it is almost always worth paying, because a missing event is a silent correctness failure while a slightly delayed event is an eventual-consistency window you already had.

**When to use it**

- **Any service that changes state and publishes an event about it**, which is most services in an event-driven architecture.
- **Where losing an event causes downstream inconsistency** — orders, payments, inventory, entitlements.
- **When you cannot use a distributed transaction**, which is essentially always across a database and a broker.
- **With change data capture**, where the outbox provides semantic events rather than raw row changes.
- **Migrating from dual writes**, as a mechanical and incrementally deployable fix.

**When to avoid it**

- **Do not use it when the event is genuinely optional** and loss is acceptable — the machinery is not free.
- **Do not use it without consumer idempotency**; you will have fixed publishing and left duplicate handling broken.
- **Do not let the outbox table grow unbounded**, which turns it into a large, churning, unpruned table.
- **Do not put a thin event in the outbox**, forcing consumers to call back and reintroducing coupling.
- **Do not use a polling relay at very high volume** — the update-per-row becomes the bottleneck.

**Advantages**

- **Atomicity between state change and event** without any distributed transaction.
- **Events are never lost**, because they are durably recorded before delivery is attempted.
- **Per-aggregate ordering is preserved**, since rows can be relayed in insertion order.
- **Works with any database and any broker**, needing no special support from either.
- **Incrementally deployable**, so a dual-writing system can migrate without a rewrite.
- **The outbox is an audit trail** of what the service intended to publish.

**Disadvantages**

- **Publishing becomes asynchronous**, adding latency between the state change and downstream visibility.
- **At-least-once delivery** still requires idempotent consumers.
- **A relay process to build, deploy and monitor**, or a CDC pipeline to operate.
- **Extra write per business transaction**, plus the outbox table's storage and maintenance.
- **Polling relays add query load** and the update-per-row limits throughput.
- **Two representations of the same fact** — the state and the event — which must be kept semantically consistent.

**Trade-offs**

**Approaches to the dual-write problem**

| Approach | Guarantees | Cost |
|---|---|---|
| Dual write (publish after commit) | None — silent loss on crash | Zero, and wrong |
| Transactional outbox + polling relay | Atomic; at-least-once delivery | Extra write; poll load; latency |
| Transactional outbox + CDC relay | Atomic; at-least-once; low latency | CDC pipeline to operate |
| CDC on the business tables directly | Atomic; no outbox needed | Exposes internal schema as the contract |
| Two-phase commit across DB and broker | Atomic delivery | Rarely supported; coordinator failure modes |
| Event sourcing | The event *is* the state | Architectural commitment |

Change data capture on business tables directly is tempting because it removes the outbox entirely, but it publishes row changes rather than domain events — which means your internal schema becomes the public contract, and a column rename becomes a breaking change for consumers you cannot enumerate. The outbox exists partly to keep that boundary.

**How it fails**

**Outbox failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Events delayed by minutes | Relay stopped or falling behind | Alert on oldest unpublished row age |
| Duplicate events downstream | Relay crashed between publish and mark | Expected — consumers dedupe on the outbox event id |
| Outbox table enormous | Published rows never deleted | Delete or archive after publishing; partition by time |
| Ordering broken downstream | Relay publishing out of insertion order, or wrong partition key | Order by id; partition by aggregate id |
| Relay becomes the bottleneck | Polling with an update per row at high volume | Move to CDC-based relay |
| Consumers call back for data | Thin events written to the outbox | Put the state in the event payload |
| Outbox writes slow the transaction | Large payloads or excessive indexing | Keep payloads reasonable; index only what the relay queries |

**Limits**

> **Operating numbers**
>
> - **Relay latency**: 50–500 ms for polling depending on interval; 10–50 ms for CDC.
> - **Alert on oldest unpublished row age**, which is the delay between a state change and the world learning about it.
> - **Retention**: delete published rows promptly, or partition by time and drop old partitions.
> - **Polling relay throughput** is limited by the update-per-row; CDC removes that ceiling.
> - **Consumer dedupe key** must be the outbox event id, which is stable across relay retries — the broker's message id is not.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| Transactional outbox | The general answer for reliable publishing | Latency; relay to operate |
| CDC on business tables | Replication and analytics | Internal schema becomes the contract |
| Event sourcing | The event stream is the source of truth | Whole-architecture commitment |
| Listen-to-yourself | Publish first, then consume your own event to write state | Inverts the problem; state lags |
| Idempotent republish from state | Periodically re-derive events from current state | Cannot express transitions, only current values |
| Accept the risk | Events that are genuinely optional | Silent inconsistency |

The listen-to-yourself pattern is worth knowing: publish the event first, and have the service consume its own event to apply the state change. That makes the publish the atomic step and the state change the derived one — appropriate when the event stream is authoritative, but it means the service's own state is eventually consistent with its own commands.

**In real systems**

- **Debezium's outbox event router** is purpose-built for this: it reads outbox table inserts from the replication log and publishes them as domain events, which is the standard production implementation.
- **Microservice architectures at scale** treat the outbox as a default rather than an option, because the dual-write failure is silent and its cost is discovered late.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** is what makes a multi-instance polling relay safe without coordination.
- **Event sourcing systems** avoid the problem entirely by making the event the state, which is the more radical version of the same insight.
- **Saga implementations** rely on the outbox for every step, since a saga whose step commits without publishing its event stalls permanently.

**Common mistakes**

- **Publishing after commit** and assuming a retry covers the crash case — the process that would retry is the one that died.
- **Adopting the outbox without consumer idempotency.**
- **Deduplicating on the broker message id**, which changes on relay retry.
- **Never deleting published rows**, growing an unmaintained table.
- **Thin events in the outbox**, forcing consumers to call back.
- **No alerting on relay lag**, so a stopped relay silently freezes the event stream.
- **Polling relays at very high volume**, where the update-per-row is the ceiling.

**The staff-level view**

The dual-write problem is the single most common correctness defect in event-driven systems, and it is invisible until it has already caused downstream damage.

- **Make the outbox the default publishing mechanism in the service template.** Teams will otherwise dual-write, because dual-writing looks correct and works in every test.
- **Insist that consumer idempotency keys use the outbox event id.** Adopting the outbox and keying on the broker's message id leaves half the problem unsolved.
- **Alert on oldest-unpublished-row age**, since a stopped relay is a silent outage of your entire event stream.
- **Prefer an outbox over CDC on business tables** for domain events, so your internal schema does not become a public contract with unknown consumers.
- **Include outbox row cleanup in the design**, because an unpruned outbox becomes a large, heavily churned table that degrades the transactional database it lives in.

**Go deeper**

Committing a state change and publishing an event are two operations on two systems with no transaction between them, so a crash in the middle either loses the event or announces a change that never happened — silently, and undetectably from either side. The outbox makes the event part of the state: insert it as a row in the same transaction, so either both exist or neither does.

A separate relay then reads unpublished rows and delivers them, retrying until it succeeds. That converts an impossible dual write into an atomic write plus at-least-once delivery. Because the relay can publish and crash before marking the row, duplicates occur, so consumers must deduplicate — keyed on the outbox event id, which is stable across retries, not the broker's message id, which is not.

Polling relays are simple and work anywhere, but the update-per-row limits throughput and the poll interval bounds latency; a change-data-capture relay reading the replication log removes both. Prefer an outbox to capturing business tables directly, because raw row changes make your internal schema a public contract. And alert on the age of the oldest unpublished row, since a stopped relay is a silent outage of your whole event stream.

The transactional outbox exists to solve the single most common correctness defect in event-driven systems: the dual write, which is silent, unreproducible in testing, and discovered downstream long after the fact.

**Why the problem is unavoidable without it.** A database commit and a broker publish are separate operations on separate systems, and no ordinary transaction spans them. Committing first and publishing second loses the event if the process dies between; publishing first and committing second announces a change that never happened. Retrying is not a fix, because the process that would perform the retry is precisely the one that failed. Distributed transactions across a database and a broker are rarely supported and bring their own coordinator failure modes.

**The reframing.** Insert the event as a row in the same transaction as the business write, and either both are durable or neither is. A relay process then delivers it. This does not remove the two-system problem — the event still has to reach the broker — but it makes the failure recoverable, because the event is durably recorded and delivery can be retried indefinitely. An impossible atomicity requirement has become an ordinary at-least-once delivery problem.

**At-least-once, and where the dedupe key comes from.** The relay may publish successfully and crash before marking the row, so the event is published again with a new broker message id. Consumers therefore need idempotency keyed on the outbox row's event id, which is carried in the payload and stable across retries. This detail matters because adopting the outbox while deduplicating on the broker's message id leaves the system exactly as broken as before, in a way that looks solved.

**Relay implementation.** A polling relay selects unpublished rows with `FOR UPDATE SKIP LOCKED` so multiple instances can run safely, publishes, and marks them. It works with any database and needs no extra infrastructure, but the update-per-row causes index churn and becomes the throughput ceiling, while the poll interval bounds latency. A log-based relay reads the database's replication log and publishes each outbox insert it observes, eliminating polling load, the update entirely, and most of the latency. At meaningful volume this is the better implementation, at the cost of operating a change-data-capture pipeline.

**Why not capture business tables directly.** It is tempting to skip the outbox and publish row changes straight from the replication log. That works well for replication, search indexing and analytics, where consumers genuinely want row state. It is the wrong boundary for a public event contract, because your internal schema becomes the thing unknown consumers depend on — a column rename or a normalisation change becomes a breaking change you cannot coordinate. The outbox preserves the distinction between storage decisions and published domain events.

**Operational requirements.** Published rows must be deleted or partitioned and dropped, or the outbox becomes a large, heavily churned table degrading the transactional database it lives in. Payloads should carry enough state that consumers need no callback, since a thin event reintroduces exactly the coupling the architecture was avoiding. And the age of the oldest unpublished row should be a first-class alert: a stopped relay is a silent outage of the entire event stream, and no downstream service will report it, because from their perspective nothing happened at all.

**Prove it — interview questions**

1. **[Basic] What is the dual-write problem?**

   <details><summary>Model answer</summary>

   Writing to your database and publishing an event are two operations on two systems with no transaction spanning them. If the process crashes between them, you either commit a change nobody downstream learns about, or — publishing first — announce a change that never committed. Both are silent: the service sees success, the broker sees nothing, and the inconsistency surfaces downstream much later. Retrying does not help, because the process that would retry is the one that died.

   </details>

2. **[Basic] How does the outbox solve it?**

   <details><summary>Model answer</summary>

   By making the event part of the same transaction as the state change. The event is inserted as a row in an outbox table alongside the business write, so either both exist or neither does. A separate relay process then reads unpublished rows and delivers them to the broker, retrying until it succeeds. You have not eliminated the two-system problem, you have made it recoverable — the event is durably recorded, so delivery can be retried indefinitely.

   </details>

3. **[Senior] Why do consumers still need idempotency with an outbox?**

   <details><summary>Model answer</summary>

   Because the relay can publish an event and then crash before marking the row as published, so the same event is delivered again on the next pass — with a different broker message id. The outbox guarantees at-least-once delivery, not exactly-once. Crucially, the deduplication key must be the outbox row's event id, which is stable across relay retries, rather than the broker's message id, which is not. Teams that adopt the outbox and key on the broker id have fixed publishing and left consumption broken.

   </details>

4. **[Senior] Compare a polling relay with a log-based one.**

   <details><summary>Model answer</summary>

   A polling relay queries for unpublished rows, publishes them, and marks them — simple, works with any database, but it adds query load proportional to the poll interval, bounds latency by that interval, and the update-per-row causes index churn that becomes the throughput ceiling. A log-based relay reads the database's replication log and publishes each outbox insert it observes, which removes polling entirely, gives sub-second latency, and needs no update at all — rows can be deleted immediately. The cost is a change-data-capture pipeline to operate and its ordering and offset semantics to understand. At meaningful volume, log-based is the better choice.

   </details>

5. **[Staff] Why use an outbox rather than capturing changes directly from your business tables?**

   <details><summary>Model answer</summary>

   Because change data capture on business tables publishes *row changes*, which makes your internal schema the public contract. A column rename, a normalisation change, or splitting a table becomes a breaking change for consumers you cannot enumerate — and those consumers are now coupled to storage decisions that should be free to evolve. An outbox lets you publish deliberate domain events with their own versioned schema, while still getting atomicity from the same transaction. Direct CDC is excellent for replication, analytics and search indexing, where the consumer genuinely wants row state; it is the wrong boundary for a public event contract.

   </details>

6. **[Principal] How would you eliminate dual writes across an organisation?**

   <details><summary>Model answer</summary>

   By making the correct path the default one, because dual-writing looks correct, passes every test, and fails only during crashes — so it will never be caught by review or by testing discipline alone. Concretely, the service template should provide an outbox-backed publish API so that the natural way to emit an event is already atomic, with the direct-publish path either absent or requiring explicit justification. The consumer framework should handle deduplication keyed on the outbox event id automatically, since fixing publishing without fixing consumption leaves the system half-correct. Relay lag needs to be a standard alert, because a stopped relay is a silent outage of the entire event stream that no downstream service will report. And I would treat the migration as mechanical rather than architectural: any service currently dual-writing can adopt the outbox incrementally — add the outbox write, run the relay alongside the existing publish, verify consumers tolerate duplicates, then remove the direct publish — which makes it a sequence of safe deploys rather than a project.

   </details>

---

### Schema evolution for events

*Producers and consumers deploy independently and read old data, so every change must be compatible in a direction you chose deliberately.*

**Flow:** `Writer schema` → `Encoded event` → `Reader schema` → `Compatibility check` → `Consumer`

> **The 30-second version**  
> Events are read by consumers you cannot name, running old code, replaying old data. Add optional fields with true defaults, never change meaning, and enforce compatibility in CI.

**The problem**

An event written today will be read tomorrow by a consumer running last month's code, and read again next year by a consumer that does not exist yet, replaying from retained history. The producer cannot coordinate a simultaneous deploy with consumers it cannot enumerate, and it cannot rewrite events already written.

This is fundamentally different from an API, where request and response exist only for the duration of a call and both sides are live. Events are persisted and replayed, so the schema is a contract not just with current consumers but with every past encoding and every future reader.

> **Three independent axes of incompatibility**  
> **New consumer, old data**: replaying history written before a field existed. **Old consumer, new data**: a consumer not yet redeployed receiving events with added fields. **Deployment order**: producer and consumer cannot be updated atomically, so one of them will be ahead of the other for some window.

**Mental model**

Two schemas are involved in every read: the **writer's schema**, which produced the bytes, and the **reader's schema**, which the consumer expects. Evolution is about which differences between them the encoding can reconcile.

1. **Backward compatible** — A new reader can read old data. Required for replay and for consumers that deploy after producers.
2. **Forward compatible** — An old reader can read new data. Required when producers deploy before consumers, which is usually.
3. **Full compatibility** — Both. Achieved by adding only optional fields with defaults and never removing or renaming required ones.
4. **Breaking change** — Anything that cannot be reconciled: removing a required field, changing a type, changing semantics of an existing field.
5. **Schema registry** — A service holding registered schemas and enforcing a compatibility rule at publish time, so incompatible changes cannot be deployed.

> **The worst breaking change is the one that still parses**  
> Renaming a field or changing a type produces an error, which is bad but obvious. Changing the *meaning* of an existing field — `amount` from pounds to pence, `status` gaining a new value consumers do not handle, `timestamp` from event time to ingestion time — passes every compatibility check and silently corrupts downstream data. Semantic changes require a new field, not a redefinition.

**How it works**

**What is safe and what is not**

```text
SAFE (full compatibility)
  add an OPTIONAL field with a default
    old reader: ignores it
    new reader on old data: uses the default
  add a new event TYPE
    old consumers filter it out
  widen a numeric type where the encoding permits
  add a value to an enum ONLY IF consumers have a
    documented default branch

UNSAFE
  remove a field           -> old readers expecting it break
  rename a field           -> equivalent to remove + add
  change a type            -> cannot be reconciled
  make an optional field required -> old data has no value
  change units or meaning  -> parses fine, corrupts silently
  add an enum value        -> if consumers switch exhaustively

THE TWO-STEP REMOVAL
  v1: field required
  v2: field optional, producers still populate it
  v3: producers stop populating (consumers must not require it)
  v4: field removed from the schema
  -> each step is independently deployable
```

1. **Choose a compatibility mode and enforce it in CI** — Backward, forward or full — decided per topic based on deployment order — and checked automatically, so an incompatible schema cannot be published.
2. **Add, never modify** — New optional fields with defaults are free. Changing an existing field is a new field plus a migration, however tempting it looks.
3. **Never repurpose a field's meaning** — Units, semantics and enum sets are part of the contract even though no schema language checks them. A change here is breaking and undetectable.
4. **Handle unknown enum values defensively** — Consumers should have an explicit default branch for values they do not recognise, or adding a legitimate new status becomes a breaking change.
5. **Remove fields in stages** — Optional, then unpopulated, then removed — with enough time between steps that all consumers have redeployed and all retained events have aged out.
6. **Version the event type for genuinely breaking changes** — `OrderPlaced.v2` as a new type or topic, with the producer publishing both during a migration window, is the only safe way to make an incompatible change.

**Choosing the compatibility mode from deployment order**

```text
PRODUCER DEPLOYS FIRST (the usual case)
  new events reach old consumers
  -> need FORWARD compatibility
  -> old readers must tolerate new data

CONSUMER DEPLOYS FIRST
  new consumers read old events
  -> need BACKWARD compatibility
  -> new readers must tolerate old data

REPLAY FROM RETAINED HISTORY
  new consumers read very old events
  -> need BACKWARD compatibility across the full
     retention window, which may be months

UNKNOWN CONSUMERS AND UNCONTROLLED DEPLOY ORDER
  -> need FULL compatibility
  -> this is the realistic default for a public topic

PRACTICAL RULE
  if you cannot name every consumer and control their
  deploy order, use FULL compatibility.
```

> **Compatibility checks do not check semantics**  
> A registry verifies that bytes written under one schema can be decoded under another. It cannot tell you that `price` changed from gross to net, that `region` now uses different codes, or that `status: PENDING` now means something else. These are the changes that cause silent downstream corruption, and the only defence is review discipline plus treating semantic change as requiring a new field.

**Worked example**

Adding multi-currency support to an order event that has assumed pounds for three years.

**The tempting change and the safe one**

```text
CURRENT
  { orderId, totalPence: 4999, ... }
  every consumer assumes GBP

TEMPTING (and wrong)
  add a currency field, and reuse totalPence for
  whatever currency it is
  { orderId, totalPence: 4999, currency: "EUR" }
  -> passes every compatibility check
  -> old consumers still read totalPence as pounds
  -> revenue reporting is now silently wrong for EUR orders
  -> three years of retained events have no currency field,
     so a new consumer cannot tell which are GBP

SAFE
  v1  add currency as OPTIONAL with default "GBP"
      producers populate it on every new event
      -> old consumers ignore it, still correct for GBP
      -> new consumers read it; retained events default
         correctly to GBP, which is historically true
  v2  add totalMinorUnits as a NEW field, explicitly
      paired with currency
      producers populate both old and new fields for
      GBP orders; only the new pair for other currencies
  v3  consumers migrate to the new pair; verify none
      read totalPence
  v4  stop populating totalPence; later remove it

WHY THE DEFAULT WORKS HERE
  "GBP" is not a placeholder - it is the historically
  correct value for every event ever written.
```

| Metric | Value | Note |
|---|---|---|
| Naive change | passes checks | **silently wrong** |
| Currency default | GBP | historically true |
| New field | explicit pairing | unambiguous |
| Removal | staged | after migration |

> **A default is only safe if it is historically correct**  
> Defaulting `currency` to GBP works because every past event genuinely was in pounds — the default reconstructs a true fact. Defaulting a new `discountApplied` field to `false` when historical orders may have had discounts would be fabricating data. Before adding a default, ask whether it is true of the data already written, and if it is not, the field cannot be safely defaulted and consumers must handle its absence explicitly.

**When to use it**

- **Every event schema, from the first version**, because retrofitting compatibility discipline after consumers exist is far harder.
- **Wherever producers and consumers deploy independently**, which is the entire point of event-driven architecture.
- **With retained topics supporting replay**, where old encodings must remain readable for the full retention period.
- **In schema registries with CI enforcement**, so an incompatible change is caught before deploy rather than in production.
- **When planning a breaking change**, where versioned types and dual publishing are the only safe route.

**When to avoid it**

- **Do not remove or rename fields in place** — stage the removal over several releases.
- **Do not change the meaning, units or semantics of an existing field**; add a new one.
- **Do not add enum values** unless consumers have a documented default branch for unknown values.
- **Do not rely on coordinated deploys** across producer and consumers; you cannot enumerate them all.
- **Do not default a new field to a value that is not historically true** of already-written events.

**Advantages**

- **Independent deployment** of producers and consumers, which is the core benefit of event-driven architecture.
- **Replay of old events remains possible**, which is what makes onboarding new consumers cheap.
- **Registry enforcement catches incompatible changes at build time**, before they can cause an incident.
- **Additive evolution is genuinely free**, so schemas can grow indefinitely without breaking anyone.
- **Explicit versioning gives a safe route for breaking changes**, rather than an impossible coordination problem.

**Disadvantages**

- **Fields accumulate**, because removal is slow and multi-stage; schemas grow and rarely shrink.
- **Compatibility checks miss semantic changes**, which are the most damaging kind.
- **Breaking changes are expensive** — a new versioned type plus dual publishing plus a migration window.
- **Defaults can fabricate history** if chosen carelessly.
- **Registry infrastructure to operate**, and discipline to enforce.

**Trade-offs**

**Compatibility modes**

| Mode | Guarantees | Allows | Use when |
|---|---|---|---|
| Backward | New reader reads old data | Delete fields; add optional | Consumers deploy first; replay matters |
| Forward | Old reader reads new data | Add fields; delete optional | Producers deploy first |
| Full | Both directions | Add optional with defaults only | Unknown consumers — the realistic default |
| None | Nothing | Anything | Never, for a shared topic |

> **Framing the decision**  
> “We can't enumerate our consumers, and we retain ninety days for replay, so the topic is configured for full compatibility — only optional fields with defaults. Adding currency is fine because GBP is historically true. Changing what `totalPence` means is not, so that becomes a new field and a staged migration.”

**How it fails**

**Schema evolution failures**

| Failure | Cause | Fix |
|---|---|---|
| Consumers crash after a producer deploy | Field removed or type changed | Full compatibility mode; staged removal |
| Silent downstream data corruption | Semantic or unit change to an existing field | New field instead; review discipline |
| Replay fails on old events | Backward compatibility not maintained | Enforce backward compatibility across the retention window |
| Consumer breaks on a new enum value | Exhaustive switch with no default branch | Require a default branch; treat enums as open sets |
| Historical data mislabelled | Default value not true of past events | Choose defaults that reconstruct fact, or leave absent |
| Cannot remove a deprecated field | No visibility into which consumers read it | Consumer registration; field-level usage telemetry |
| Incompatible schema reaches production | No registry enforcement in CI | Fail the build on compatibility violation |

**Limits**

> **Practical guidance**
>
> - **Full compatibility** is the right default for any topic whose consumers you cannot enumerate.
> - **Backward compatibility must hold across the full retention window**, which may be months.
> - **Field removal** takes three or four releases: optional, unpopulated, unused, removed.
> - **Dual publishing during a versioned migration** typically runs for weeks, bounded by the slowest consumer.
> - **Enums should be treated as open sets** by consumers, with an explicit unknown branch.

**Alternatives**

| Approach | Handles change by | Trade |
|---|---|---|
| Additive evolution + registry | Only ever adding optional fields | Schemas grow; removal is slow |
| Versioned event types | Publishing a new type alongside the old | Dual publishing; migration coordination |
| Versioned topics | A new topic per major version | Consumers must migrate; two streams |
| Schemaless (JSON, no registry) | Hoping consumers are defensive | Breakage discovered in production |
| Upcasting on read | Transforming old events to the current schema at read time | Transformation code accumulates forever |
| Rewriting history | Not possible for published events | — |

Upcasting — a transformation layer that converts any historical version into the current schema on read — is common in event-sourced systems and worth understanding: it keeps consumers simple at the cost of an ever-growing library of version transformers that can never be deleted while old events are retained.

**In real systems**

- **Avro with a schema registry** was designed around writer and reader schemas precisely to make this reconciliation explicit and checkable.
- **Protocol Buffers' field numbers** make additive evolution safe by construction — fields are identified by number, so renaming is harmless and removal reserves the number.
- **Confluent Schema Registry compatibility modes** enforce backward, forward or full compatibility at publish time, which is what makes CI enforcement possible.
- **Public API versioning practices** solve a related but easier problem, because both sides are live and a deprecation window can be coordinated.
- **Event-sourced systems** rely on upcasting, since events from the earliest days of the system must remain readable forever.

**Common mistakes**

- **Removing or renaming a field in place.**
- **Changing units or meaning** of an existing field, which passes every check and corrupts silently.
- **Adding an enum value** when consumers switch exhaustively.
- **Defaults that are not historically true**, fabricating data for past events.
- **Assuming a coordinated deploy** across producer and all consumers.
- **No registry enforcement**, so incompatibility is discovered in production.
- **Ignoring the retention window**, so replay of old events fails after a schema change.

**The staff-level view**

Event schemas are public contracts with consumers you cannot name, which makes evolution discipline a structural requirement rather than good practice.

- **Enforce compatibility in CI, not in review.** A registry check that fails the build is the only mechanism that reliably prevents an incompatible deploy.
- **Default to full compatibility** for any topic without an enumerable consumer list, which in practice is most of them.
- **Treat semantic change as breaking**, even though no tool detects it. Units, meanings and enum sets are part of the contract, and changing them silently corrupts downstream data.
- **Require a documented unknown branch for every enum**, so adding a legitimate new value does not become a breaking change.
- **Invest in consumer visibility** — registration, or field-level usage telemetry — because the practical blocker on removing a deprecated field is not knowing who still reads it.

**Go deeper**

Every read involves two schemas: the writer's, which produced the bytes, and the reader's, which the consumer expects. Backward compatibility means a new reader can read old data, needed for replay; forward compatibility means an old reader can read new data, needed because producers usually deploy first. Full compatibility — only adding optional fields with defaults — is the right default whenever consumers cannot be enumerated or their deploy order controlled.

Adding optional fields is free. Removing or renaming is a three-or-four-release migration: make it optional, stop populating it, verify nobody reads it, then remove. And the most dangerous change is the one that still parses — changing units, redefining a field's meaning, or adding an enum value consumers switch on exhaustively. These pass every compatibility check and silently corrupt downstream data, so semantic change always requires a new field.

Defaults must be historically true: defaulting `currency` to GBP is safe if every past event really was in pounds, while defaulting `discountApplied` to false fabricates data if some were not. Enforce compatibility in CI via a schema registry so an incompatible change cannot deploy, and invest in consumer visibility — the recurring blocker is not making changes safely but retiring old fields nobody can prove are unused.

Event schemas differ from API contracts in a way that changes everything about evolution: events are persisted and replayed, so the schema is a contract with every past encoding and every future reader, not merely with whoever is live right now.

**Three axes of incompatibility.** A new consumer replaying retained history reads events written before current fields existed, requiring backward compatibility across the whole retention window. A consumer not yet redeployed receives events from an updated producer, requiring forward compatibility. And because producers and consumers cannot deploy atomically, one will always be ahead of the other for some window. When consumers cannot be enumerated — the normal situation for a shared topic — full compatibility is the only defensible mode, which in practice means adding optional fields with defaults and nothing else.

**Removal is a migration, not a change.** Deleting or renaming a field breaks readers expecting it and breaks replay of events that contain it. The safe sequence is: make it optional, continue populating it while consumers migrate, stop populating it once telemetry shows nobody reads it, and remove it from the schema only after the retention window has aged out the last events containing it. That is three or four independently deployable releases, which is why schemas grow and rarely shrink.

**The dangerous changes are the ones that parse.** A registry verifies that bytes written under one schema decode under another. It cannot detect that `amount` changed from pounds to pence, that `timestamp` now means ingestion time rather than event time, or that a new `status` value has appeared that consumers switch on exhaustively. These pass every automated check and corrupt downstream data silently, often discovered in a reconciliation weeks later. The only defence is a stated rule — units, semantics and valid value sets are part of the contract, so changing them requires a new field — plus consumers treating enums as open sets with an explicit unknown branch.

**Defaults must reconstruct fact, not invent it.** Adding a `currency` field defaulted to GBP is correct if every historical event genuinely was in pounds; the default recovers information that was implicit. Adding a `discountApplied` field defaulted to false is wrong if some historical orders had discounts, because consumers will treat fabricated data as real. When no default is historically true, the field cannot be safely defaulted and consumers must handle its absence — which meaningfully constrains what can be added cheaply.

**Breaking changes need versioned types.** When a change genuinely cannot be made compatibly, the only safe route is publishing a new versioned event type or topic alongside the old, letting consumers migrate independently, and retiring the old one once nobody reads it. The practical blocker is almost never the mechanism; it is visibility. Teams carry deprecated fields and versions for years because they cannot prove nobody consumes them. Consumer registration and field-level read telemetry convert that from a guess into a decision, and are worth building before they are needed.

**Governance must be tooling, not discipline.** The failure mode is a producer team making a locally reasonable change that breaks consumers they have never met, and review cannot reliably catch it because the reviewers are on the producing team. A schema registry with a per-topic compatibility mode, enforced in CI so an incompatible schema cannot deploy, handles the structural half mechanically. The semantic half remains cultural, which is why it should be stated as an explicit rule rather than left to judgement.

**Prove it — interview questions**

1. **[Basic] What is the difference between backward and forward compatibility?**

   <details><summary>Model answer</summary>

   Backward compatibility means a new reader can read old data, which you need when consumers deploy before producers and whenever you replay retained history. Forward compatibility means an old reader can read new data, which you need when producers deploy first — the usual case. Full compatibility is both, achieved by only ever adding optional fields with defaults, and it is the right default whenever you cannot enumerate your consumers or control their deploy order.

   </details>

2. **[Basic] Why can't you just rename a field?**

   <details><summary>Model answer</summary>

   Because a rename is a removal plus an addition. Consumers reading the old name find nothing, and events already written contain only the old name, so replay breaks too. The safe path is to add the new field, populate both for a period, migrate consumers, verify nobody reads the old one, stop populating it, and only then remove it — which is three or four independently deployable releases rather than one change.

   </details>

3. **[Senior] What is the most dangerous kind of schema change?**

   <details><summary>Model answer</summary>

   A semantic change that still parses. Changing `amount` from pounds to pence, redefining `timestamp` from event time to ingestion time, or adding a status value that consumers do not handle — all of these pass every compatibility check because the bytes decode fine, and then silently corrupt downstream data. No registry can detect them, because they are changes to meaning rather than to structure. The rule that prevents them is that any change to units, semantics or the set of valid values requires a new field rather than a redefinition of an existing one.

   </details>

4. **[Senior] How do you choose a default value for a new field?**

   <details><summary>Model answer</summary>

   By asking whether it is historically true of events already written. Defaulting a new `currency` field to GBP is safe if every past event genuinely was in pounds, because the default reconstructs a fact. Defaulting a new `discountApplied` field to false is unsafe if some historical orders had discounts, because it fabricates data that consumers will then rely on. If no default is historically true, the field cannot be safely defaulted, and consumers must handle its absence explicitly — which is a real constraint on what can be added cheaply.

   </details>

5. **[Staff] You need to make a genuinely breaking change to a widely consumed event. How?**

   <details><summary>Model answer</summary>

   By publishing a new versioned type or topic alongside the old one, because there is no way to coordinate a simultaneous change with consumers you cannot enumerate. The producer emits both `OrderPlaced` and `OrderPlaced.v2` during a migration window, consumers migrate independently at their own pace, and the old type is retired only once consumer telemetry shows nobody is reading it. The practical blocker is usually visibility rather than mechanism — teams cannot retire the old version because they do not know who still consumes it — which is why consumer registration or field-level usage telemetry is worth having before you need it. The migration window is bounded by the slowest consumer, so it should be communicated with a deadline rather than left open-ended.

   </details>

6. **[Principal] How do you govern event schema evolution across many teams?**

   <details><summary>Model answer</summary>

   By moving enforcement from discipline into tooling, since the failure mode is a producer team making a locally reasonable change that breaks consumers they have never met. A schema registry with a compatibility mode per topic, checked in CI so an incompatible schema physically cannot be deployed, handles the structural half — and full compatibility should be the default for any topic without an enumerable consumer list. The semantic half cannot be automated, so it needs a cultural rule stated plainly: units, meanings and valid value sets are part of the contract, and changing them requires a new field. Beyond that, I would invest in consumer visibility, because the recurring organisational blocker is not making changes safely but retiring old ones — teams carry deprecated fields for years because nobody can prove they are unused. Registration plus field-level read telemetry turns that from a guess into a decision, and it is what makes additive evolution sustainable rather than merely safe.

   </details>

---
