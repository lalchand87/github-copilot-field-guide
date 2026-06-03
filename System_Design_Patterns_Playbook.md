# Staff Engineer System Design Patterns Playbook

> "The handbook taught you the fundamentals. This playbook teaches you to use them."

---

## What This Is

This is a practitioner's playbook of 15 recurring problem shapes. Every system design interview — whether it's WhatsApp, Uber, a payment system, or an IAM platform — is really asking you to recognise a pattern and apply it under pressure, with changing constraints.

The patterns in this playbook:
- Appear in every major Staff Engineer interview loop
- Recur across domains (payments, social, infrastructure, IAM)
- Require you to make tradeoff decisions, not recall facts
- Are evaluated differently at Staff level than at Senior level

## How to Read This

Each pattern follows the same structure:

1. **The Shape** — what kind of problem this is
2. **When you see it** — signals in an interview that this pattern applies
3. **The core tension** — the fundamental tradeoff you must navigate
4. **Building blocks** — the techniques that compose the solution
5. **Canonical problems** — worked designs using the pattern
6. **Deep dives** — the hard sub-problems within the pattern
7. **Interview calibration** — what separates Senior from Staff answers
8. **Failure modes** — what goes wrong when the pattern is misapplied

## The IAM/Payments Thread

Every pattern is grounded in examples from IAM and payments — the domains you work in daily. This is deliberate. The fastest way to internalise a pattern is to map it onto a system you already understand deeply.

---

## The 19 Patterns

| # | Pattern | Core Tension | Canonical Problems |
|---|---|---|---|
| 1 | Real-Time Updates | Push vs Pull; latency vs connection cost | WhatsApp, Slack, live sports, collaborative editing |
| 2 | Dealing With Contention | Correctness vs throughput | Ticket booking, payments, inventory, flash sales |
| 3 | Multi-Step Processes | Atomicity vs availability | Payment flows, IAM enrollment, order fulfilment |
| 4 | Scaling Reads | Freshness vs latency | News feed, product catalog, IAM permission checks |
| 5 | Scaling Writes | Consistency vs throughput | Analytics, logging, social media, telemetry |
| 6 | Handling Large Blobs | Reliability vs throughput | Dropbox, YouTube upload, Instagram, Google Drive |
| 7 | Long-Running Tasks | Reliability vs simplicity | Video processing, ML training, report generation |
| 8 | Search Systems | Freshness vs relevance | Product search, log search, document search |
| 9 | Notification Systems | Throughput vs ordering | Social feeds, security alerts, marketing campaigns |
| 10 | Rate Limiting | Fairness vs precision | API gateway, auth brute-force, payment throttling |
| 11 | Scheduling Systems | Reliability vs simplicity | Subscription billing, email campaigns, cert rotation |
| 12 | Distributed Transactions | Atomicity vs availability | Cross-bank payments, order + inventory + payment |
| 13 | Multi-Region Systems | Latency vs consistency | Global payments, IAM, data residency compliance |
| 14 | Event-Driven Systems | Decoupling vs complexity | Analytics pipelines, audit systems, data sync |
| 15 | API Gateway Pattern | Security vs latency | IAM platforms, microservices entry point |
| 16 | Search, Ranking, and Recommendation | Keyword vs semantic; relevance vs personalisation | Netflix recommendations, Amazon product search |
| 17 | Analytics and Aggregation | Latency vs accuracy; streaming vs batch | Fraud analytics, metrics platforms, dashboards |
| 18 | Data Synchronisation | Consistency vs availability across systems | OLTP→OLAP, DB→search index, CDC pipelines |
| 19 | Platform and Control Plane Design | Self-service vs control; stability vs velocity | IAM platform, feature flags, developer portal |

---
# Chapter A: Architecture Evolution

> "Every Staff engineer was once a senior engineer who watched their system break at scale and had to figure out what to replace it with."

---

## What This Chapter Is

Each pattern in this playbook contains an inline Evolution Path section. This chapter serves as a cross-pattern reference — the Universal Evolution Law, the 10x trigger table, and the architectural progression principles that apply across all patterns.

For each major pattern, you will find:
- The stages of evolution
- What breaks at each stage (the forcing function that drives the next stage)
- What replaces it
- The cost and risk of the transition
- The question the interviewer is actually asking when they say "what at 10x?"

---

## The Universal Evolution Law

```text
Every system evolves through the same four phases:

Phase 1: Make it work       (correct before fast)
Phase 2: Make it reliable   (fast before scalable)
Phase 3: Make it scalable   (scalable before cheap)
Phase 4: Make it cheap      (optimise after it works)

Engineers who skip phases pay the debt later, at higher cost.
Staff engineers know which phase they are in.
```

---

## Evolution Path 1: Scaling Reads

```text
Stage 1: Single Database (1-10K req/sec)

  Client → API → PostgreSQL
  
  Everything fits. No complexity. Ship features.
  
  Breaking point: query latency p99 > 200ms
  Why it breaks: hot rows, missing indexes, growing table size
  Signal: DB CPU > 60% sustained, or slow query log filling

Stage 2: Add Indexes (free, do this first)

  Client → API → PostgreSQL (with proper indexes)
  
  Cost: ~0. Impact: often 100-1000× query speedup.
  
  Breaking point: indexes exist, queries are fast, but DB CPU is high
  Why it breaks: too many queries, not that each query is slow
  Signal: query count (not duration) is the dominant CPU consumer

Stage 3: Read Replicas (10K-100K req/sec)

  Client → API → PostgreSQL Primary (writes)
                → PostgreSQL Replica 1 (reads)
                → PostgreSQL Replica 2 (reads)
  
  Cost: 2× database instances. Operational: replica lag monitoring, read-your-writes routing.
  
  Breaking point: replica lag growing; or DB still saturated despite replicas
  Why it breaks: read volume exceeds what even 3 replicas can serve;
                  OR data is too large to cache effectively on each replica
  Signal: replica lag > 5s; all replicas CPU > 70%

Stage 4: Add Cache (100K-1M req/sec)

  Client → API → Redis (cache, TTL-based)
                  → PostgreSQL Replica (on cache miss)
  
  Cost: Redis cluster ($200-1000/month). Operational: cache invalidation strategy.
  
  Breaking point: cache hit ratio < 70%; or cache invalidation too complex to reason about
  Why it breaks: data has too many dimensions (user+product+context) to cache efficiently;
                  OR write rate is so high that cache is constantly invalidated
  Signal: cache miss rate rising; cache invalidation errors in logs

Stage 5: CQRS / Read Model (1M-10M req/sec)

  Writes → PostgreSQL (write model, normalised)
                → CDC (Debezium) → Kafka → Read Model Builder
  Reads  → Read Model (denormalised, optimised for specific queries) → Response
  
  Cost: Kafka + CDC infrastructure ($800+/month). Operational: schema evolution complexity.
  
  Breaking point: read model still can't serve all query patterns at required latency
  Why it breaks: geographic distribution required; single-region read model is too far
  Signal: p99 latency from distant regions > SLO

Stage 6: Multi-Region Cache Hierarchy (10M-100M+ req/sec)

  Global: CDN edge (static + semi-static content, worldwide)
  Regional: Redis cluster per region (hot data, 1ms local reads)
  Origin: PostgreSQL per-region (write authority, with cross-region replication)
  
  Cost: 3-5× infrastructure multiplied by regions. Operational: conflict resolution.
  
  Breaking point: this is the ceiling for most systems.
                   Beyond this: domain sharding (different products to different clusters)
```

**The interviewer's "10x" question**: They are asking which stage transition you are about to hit. The correct answer names the current stage, identifies what the breaking point metric is, and describes the next stage transition — including the migration cost and risk.

---

## Evolution Path 2: Scaling Writes

```text
Stage 1: Direct Database Writes (0-5K writes/sec)

  Client → API → PostgreSQL INSERT
  
  Breaking point: PostgreSQL write throughput saturated (~10-20K simple inserts/sec)
  Signal: write latency p99 > 50ms; DB I/O wait > 30%

Stage 2: Connection Pooling + Batching (5K-50K writes/sec)

  API → PgBouncer → PostgreSQL
  Application: batch multiple inserts into single statement
  
  Cost: ~free (PgBouncer is open source). Impact: 5-10× effective throughput.
  
  Breaking point: single PostgreSQL node is the bottleneck regardless of pooling
  Signal: PostgreSQL CPU > 80%; WAL write rate at disk I/O limit

Stage 3: Write Queue (50K-500K writes/sec)

  Client → API → Kafka (durable write buffer)
                  → Consumer (batch writes to PostgreSQL)
  
  Client sees write acknowledged immediately (Kafka commit).
  Database sees batch writes (e.g., 1000 rows per INSERT) at controlled rate.
  
  Cost: Kafka cluster ($800/month). Latency: data available in DB within 1-5 seconds.
  
  Breaking point: single PostgreSQL still can't keep up with consumer write rate
  Signal: Kafka consumer lag growing despite batching

Stage 4: Partitioning (500K+ writes/sec, or >1TB data)

  Kafka → Consumer → PostgreSQL (partitioned by created_at, range)
  Old partitions: archived to S3 (cold storage)
  Hot partitions: fast, small, indexable
  
  Breaking point: table partitioning helps, but a single PostgreSQL node is still a limit
  Signal: cross-partition queries slow; VACUUM taking too long on large partitions

Stage 5: Sharding (1M+ writes/sec, or >10TB hot data)

  Kafka → Router → PostgreSQL Shard 1 (users 0-25%)
                → PostgreSQL Shard 2 (users 25-50%)
                → PostgreSQL Shard 3 (users 50-75%)
                → PostgreSQL Shard 4 (users 75-100%)
  
  Cost: 4× database instances + router complexity. Operational: cross-shard queries, resharding.
  
  Breaking point: this handles most systems. Above: purpose-built distributed DB.

Stage 6: Distributed Write-Optimised DB (10M+ writes/sec)

  Cassandra, ScyllaDB, or DynamoDB
  
  Cassandra: 100K-1M writes/sec per node, auto-sharding, no single point of failure
  Trade-off: eventual consistency, no joins, schema must match access patterns exactly
```

---

## Evolution Path 3: Real-Time Updates

```text
Stage 1: Polling (1-1K clients)

  Client polls GET /updates every 5 seconds.
  Simple. Works. 99% of responses are empty.
  
  Breaking point: poll QPS exceeds what the server can handle
  Signal: server CPU dominated by "return empty" responses

Stage 2: Long Polling (1K-10K clients)

  Client connects, server holds until event or timeout.
  Near-real-time. One connection per client.
  
  Breaking point: thread-per-connection model runs out of threads
  Signal: thread count > max, request queuing

Stage 3: SSE / WebSockets, Single Server (10K-100K clients)

  Async server (Netty, WebFlux). One event loop thread handles thousands of connections.
  
  Breaking point: single server's RAM for connections
  Signal: connection count per server > 50K; memory > 80%

Stage 4: Multiple WebSocket Servers + Pub/Sub (100K-1M clients)

  Connection Tier (multiple servers) + Redis Pub/Sub for cross-server fan-out.
  
  Breaking point: Redis Pub/Sub memory and CPU
  Signal: Redis CPU > 70%; pub/sub channel count > 100K

Stage 5: Kafka Fan-Out + Connection Gateway (1M-100M clients)

  Separate connection management tier (stateful) from message processing tier (stateless).
  Kafka handles durable, ordered fan-out. Connection gateway handles WebSocket lifecycle.
  
  Breaking point: connection memory at planetary scale
  Signal: infrastructure cost exceeds revenue generated per connection

Stage 6: Tiered Connectivity (100M+ clients)

  Most clients: push notifications (mobile) + SSE (web) for async updates
  Active clients: WebSocket for real-time (only when app is in foreground)
  Connection tier: geographically distributed, nearest-region routing
  
  WhatsApp, WeChat architecture: most messages go via push notification, not WebSocket.
  WebSocket only for active sessions.
```

---

## Evolution Path 4: Multi-Step Processes

```text
Stage 1: Synchronous, In-Transaction (1-100 ops/sec)

  API call executes all steps in a single database transaction.
  Works when all steps are on the same database.
  
  Breaking point: steps span multiple services; or single step takes >5 seconds
  Signal: transaction timeouts; long-running transaction locks

Stage 2: Async with Database Queue (100-1K ops/sec)

  Step 1: save initial state to DB.
  Worker process: polls for incomplete workflows, executes next step.
  
  Cost: dedicated worker process. Operational: worker crash recovery.
  Breaking point: DB polling is expensive; worker is a bottleneck
  Signal: polling queries consuming >20% of DB CPU

Stage 3: Message Queue + State Machine (1K-10K ops/sec)

  Each step completion publishes event to queue.
  Worker consumes event, executes next step, publishes next event.
  State machine stored in DB.
  
  Breaking point: choreography becomes hard to reason about; debugging is painful
  Signal: engineers can't answer "why is workflow 123 stuck?" without extensive log digging

Stage 4: Dedicated Orchestrator (10K+ ops/sec)

  Temporal, Conductor, or custom orchestrator.
  Central state machine with retry, timeout, versioning, and visibility built in.
  
  Cost: dedicated orchestrator service + operational expertise.
  Breaking point: rarely. Temporal handles millions of workflows/day.

Stage 5: Domain-Specific Workflow Platform

  At Google/Amazon scale: build or license a platform.
  Choreography for event-driven decoupled flows.
  Orchestration for complex business processes.
  Run both, with clear guidelines for which to use when.
```

---

## The 10x Trigger Table

When an interviewer asks "what happens at 10x?", they want to know which stage transition you're approaching and what the forcing function is.

| Pattern | Current Stage | 10x Trigger | Next Stage |
|---|---|---|---|
| Scaling Reads | Single DB | Query latency p99 → SLO violation | Indexes → Replicas |
| Scaling Reads | Read Replicas | Replica lag > 5s | Add Cache |
| Scaling Reads | Cache | Hit ratio < 70% | CQRS / Read Model |
| Scaling Writes | Direct DB | Write latency > 50ms | Connection pooling + batching |
| Scaling Writes | Batching | Consumer lag growing | Partitioning |
| Scaling Writes | Partitioned | VACUUM too slow | Sharding |
| Real-Time | Polling | Server CPU dominated by empty responses | Long polling |
| Real-Time | WebSocket single server | Memory > 80% | Multiple servers + pub/sub |
| Real-Time | Redis Pub/Sub | Redis CPU > 70% | Kafka fan-out |
| Contention | DB locks | Lock wait time > 50ms | Redis pre-allocation |
| Multi-Step | In-transaction | Cross-service needed | Message queue + state machine |
| Multi-Step | Choreography | Can't debug stuck workflow | Orchestrator |

---

## Architecture Review Questions for Evolution

When presenting any design, anticipate and answer these:

1. **"What stage is this?"** — Name it explicitly. "This is a Stage 3 read scaling architecture: replicas + cache, appropriate for our current 200K req/sec load."

2. **"What breaks at 10x?"** — "At 2M req/sec, the cache hit ratio drops because the long tail of user-specific data exceeds our Redis capacity. The next stage is CQRS with a purpose-built read model."

3. **"Why not jump straight to the mature architecture?"** — "Each stage transition carries migration risk and operational overhead. Moving to CQRS today adds 3 months of engineering time for a problem we won't have for 18 months."

4. **"How do you know when you've hit the breaking point?"** — Name the specific metric and alert threshold that triggers the transition.

---

*End of Chapter A: Architecture Evolution*

---
---

# Chapter B: Migration Playbook

> "The gap between 'here is the ideal architecture' and 'here is how we get there without taking down production' is where Staff engineers earn their title."

---

## Why Migration Strategy Is a Staff Skill

A junior engineer designs the target state. A senior engineer identifies the gaps between current and target. A staff engineer designs the path between them — with rollback plans, success metrics, and a realistic timeline.

Every interview question that begins "how would you redesign X" is implicitly asking: "how would you migrate from the existing X to the new X without destroying the business?"

---

## The Migration Framework

Every migration has six phases:

```text
Phase 1: CURRENT STATE
  Document what exists.
  Identify the specific pain being solved.
  Measure the baseline (this becomes your success metric).

Phase 2: TARGET STATE
  Design the ideal architecture (what the pattern chapters describe).
  Identify the non-negotiable requirements (downtime tolerance, consistency guarantees).

Phase 3: INCREMENTAL STEPS
  Break the migration into steps that can each be independently deployed and validated.
  Each step must leave the system in a consistent, deployable state.
  "Big bang" migrations fail. Incremental migrations succeed.

Phase 4: DUAL RUNNING
  Run old and new systems in parallel during transition.
  Validate the new system against the old system's behavior.
  Shift traffic incrementally (canary → 1% → 10% → 50% → 100%).

Phase 5: ROLLBACK PLAN
  For every step: what does rollback look like?
  How long does rollback take?
  What data is lost or inconsistent if you roll back?

Phase 6: SUCCESS METRICS
  What specific metric proves the migration succeeded?
  What is the validation window before you consider the old system decommissioned?
```

---

## Canonical Migration 1: Synchronous to Event-Driven Payment Processing

### Current State

```text
POST /payments
  → Fraud check (synchronous, 200ms)
  → Bank authorisation (synchronous, 300ms)
  → DB write (synchronous, 10ms)
  → Email notification (synchronous, 150ms)
  → Response to client

Total latency: ~660ms
Failure mode: if email service is down, payments fail
Coupling: payment service calls fraud, bank, email directly
```

### Target State

```text
POST /payments
  → Validate input
  → Write PENDING to DB (10ms)
  → Publish PaymentInitiated to Kafka (5ms)
  → Return 202 Accepted immediately (15ms total)

Async workers:
  FraudWorker: consumes PaymentInitiated → check → publish FraudChecked
  BankWorker: consumes FraudChecked (if PASS) → authorise → publish BankAuthorised
  SettlementWorker: consumes BankAuthorised → settle → publish PaymentSettled
  EmailWorker: consumes PaymentSettled → send email (failure: retry, doesn't affect payment)

Benefits:
  - Client response: 15ms vs 660ms
  - Email failure doesn't fail the payment
  - Each worker scales independently
  - Full audit trail in Kafka
```

### Incremental Migration Steps

```text
Step 1: Add Kafka (no behaviour change, 0 risk)
  Deploy Kafka cluster.
  Add Kafka publishing to existing payment code (fire-and-forget, no consumers yet).
  Validate: Kafka receives events, nothing breaks.
  Rollback: remove Kafka publish calls.

Step 2: Migrate email to async (low risk)
  Deploy EmailWorker consuming PaymentSettled events.
  Change payment code: remove synchronous email call, publish event instead.
  EmailWorker sends email; failures are retried, don't affect payment response.
  Dual-run: keep old email code for 1 week, compare email delivery rates.
  Success metric: email delivery rate ≥ previous (99.5%+).
  Rollback: re-enable synchronous email call, disable EmailWorker.

Step 3: Migrate bank call to async (medium risk)
  Deploy BankWorker.
  Change response semantics: POST /payments now returns 202 Accepted (was 200 OK).
  Add client polling endpoint: GET /payments/{id}/status.
  Canary: 1% of traffic uses async bank call. Monitor failure rate.
  Success metric: payment success rate ≥ synchronous (no regressions).
  Rollback: re-enable synchronous bank call for all traffic.

Step 4: Migrate fraud check to async (medium risk)
  Deploy FraudWorker.
  Add timeout handling: if fraud check doesn't complete in 5s, apply default rules.
  Canary: 10% of traffic. Monitor fraud rate, false positive rate.
  Rollback: re-enable synchronous fraud check.

Step 5: Remove synchronous fallbacks (completion)
  After 30 days of stable async operation: remove old synchronous code paths.
  Decommission the direct service-to-service calls.
  
  Success metrics:
    - Payment latency p99 < 50ms (was 660ms)
    - Payment success rate ≥ previous
    - Email delivery rate ≥ previous
    - Fraud detection rate unchanged
```

### Rollback Plan

```text
Each step is independently rollbackable because:
  - Dual-running: both paths exist until the old one is explicitly removed
  - Feature flags control which path is active
  - Validation windows catch regressions before old code is deleted

Final rollback (after all steps): re-enable synchronous calls for 100% of traffic.
Takes: 1 deployment cycle (minutes to hours).
Data consistency: Kafka events may have been published but not acted on; idempotent workers handle replay correctly.
```

---

## Canonical Migration 2: Adding a Cache Layer to an Existing System

### Current State

```text
Every API call: SELECT * FROM products WHERE id = ?
DB query latency: 5-20ms
DB load: 8,000 reads/sec at peak (80% of DB capacity)
```

### Target State

```text
API → Redis cache (TTL=5min)
        → miss → PostgreSQL
        
Cache hit ratio target: 80%
DB load after: 1,600 reads/sec (20% of current)
```

### Incremental Migration Steps

```text
Step 1: Instrument first (no behaviour change)
  Add cache-miss and cache-hit counters (even with no cache yet, to establish baseline).
  Add query timing to identify the top 10 most frequently queried hot keys.
  
  Run for 1 week. Output: "product:123 is queried 50,000 times/day; product:456: 40,000/day"

Step 2: Shadow cache (read-only, no serving)
  On every DB read: also write result to Redis with 5-min TTL.
  Do not serve from cache yet.
  Monitor: is Redis populating correctly? Are values correct?
  Duration: 48 hours.

Step 3: Cache reads for read-only entities first (low risk)
  Enable cache reads for product data (never changes in real-time).
  Cache reads for 100% of product lookups.
  Monitor: DB load drops. Validate: product data served from cache matches DB.
  
  Success metric: DB reads for products drop 70%+.

Step 4: Cache reads for mutable entities with TTL (medium risk)
  Enable cache reads for user profiles (changes occasionally).
  TTL = 5 minutes (max staleness tolerance for profiles).
  
  Edge case: user updates profile, sees stale profile for up to 5 minutes.
  Mitigation: on write, also update/invalidate cache.
  
  Success metric: cache hit ratio > 70%.

Step 5: Add active invalidation on writes (completion)
  When any entity is written: immediately delete the corresponding cache key.
  The next read re-populates the cache with fresh data.
  TTL becomes a safety net, not the primary invalidation mechanism.
  
  Final success metric:
    - Cache hit ratio: 80%+ sustained
    - DB load: reduced by 60%+
    - No increased error rate
    - No customer complaints about stale data
```

### Rollback Plan

```text
At any step: disable cache reads (feature flag).
All reads fall back to DB.
DB is always the source of truth; cache is always advisory.
Rollback is instantaneous (flip the feature flag).
```

---

## Canonical Migration 3: Strangler Fig — Monolith to Microservices

### Current State

```text
IAM Monolith:
  All IAM functionality in one deployable.
  Single database.
  All logic coupled.
  
  Pain: auth team can't deploy without coordinating with admin team.
       DB schema changes require full regression testing.
       Can't scale auth tier independently of admin tier.
```

### Target State

```text
Auth Service:      owns login, token issuance
Token Service:     owns token validation, introspection
Admin Service:     owns user management, role assignment
Audit Service:     owns audit logging

Each service: own database, own deployment pipeline, own scaling.
```

### Incremental Migration Steps (Strangler Fig)

```text
Step 1: Add routing proxy in front of monolith (0 risk)
  Deploy nginx/Kong in front of monolith.
  All traffic: proxy → monolith (no change yet).
  This is the "fig vine" attachment point.

Step 2: Extract highest-value, lowest-risk service first
  Identify: token introspection is the highest-traffic, most isolated function.
  Build Token Service alongside monolith.
  
  Dark launch: token introspection calls go to both monolith and Token Service.
  Compare responses. Fix discrepancies.
  When responses match: route 1% of introspection traffic to Token Service.

Step 3: Gradual traffic shift (canary)
  1% → 10% → 50% → 100% of token introspection to Token Service.
  Monolith's introspection code: kept but unused.
  
  Success metric: error rate ≤ monolith (at each traffic stage).

Step 4: Remove introspection from monolith
  After 30 days of 100% traffic on Token Service: delete monolith introspection code.
  DB: migrate token data to Token Service database.
  
  Data migration: run dual writes (monolith DB + Token Service DB) during migration.
  Validate data parity. Then: read from Token Service DB only. Then: stop dual write.

Steps 5-N: Repeat for Auth Service, Admin Service, Audit Service
  Each follows same pattern:
  Build alongside → dark launch → canary → 100% → remove from monolith → migrate data.

Timeline: 18-24 months for complete extraction. This is normal. Do not rush.
```

### The Common Mistake: The Distributed Monolith

```text
Anti-pattern to avoid:
  "Microservices" that call each other synchronously in a chain.
  Token Service calls Auth Service calls User Service calls DB.
  
  This is a monolith that happens to have network hops between components.
  You get: all the operational complexity of microservices + tight coupling of a monolith.
  
  Test: can Token Service be deployed independently without coordinating with Auth Service?
  If no: it's not microservices. It's a distributed monolith.
```

---

## The Migration Evaluation Criteria

A Staff Engineer evaluates any migration proposal against six questions:

```text
1. What is the rollback plan at every step?
   If rollback requires more than one deployment, the step is too large.

2. What is the validation window before removing the old system?
   Minimum: 7 days at production load. For financial systems: 30 days.

3. What data migration is required, and can it be done online?
   Online migration (no downtime): preferred always.
   Offline migration (with downtime): must have SLA-level approval.

4. What is the blast radius if this step fails?
   Acceptable: degraded performance for a subset of users.
   Not acceptable: data loss, security regression, or total outage.

5. How do you validate that the new system matches the old system's behavior?
   Shadow reads: compare responses in parallel.
   Traffic canary: route small percentage, compare error rates.

6. When are you done?
   Define success metrics before starting. 
   "The migration is complete when X metric is Y for Z consecutive days."
```

---

*End of Chapter B: Migration Playbook*

---
---

# Chapter C: Organisational Lens

> "The best technical design fails if nobody owns it, nobody funds it, and nobody gets paged when it breaks."

---

## Why This Chapter Exists

A Distinguished Engineer at Google or Stripe thinks about organisational boundaries as often as technical boundaries. The API gateway is not just a routing component — it is a platform product, with a team that owns it, a roadmap, a service agreement, and an on-call rotation. Ignoring this is why technically excellent systems fail in production.

In Staff and Principal interviews at companies with large engineering organisations, you will be asked: "Who owns this?" and "What's the team model?" These questions have technical implications. Answer them.

---

## The Ownership Models

### Pattern: The Platform Team

```text
What it is:
  A team that builds shared infrastructure for other engineering teams.
  They own the API gateway, the Kafka cluster, the auth platform, the CI/CD platform.
  Other teams are their customers.

Who funds it:
  Typically: central engineering budget, not product teams.
  Risk: platform teams are overhead in the eyes of the business → first to be cut.
  Mitigation: demonstrate measurable value (engineering hours saved, incident reduction).

Who gets paged:
  Platform team. 24/7 on-call for the platform they own.
  SLA to application teams: "our Kafka cluster is available 99.95%."

Who consumes it:
  Application teams. They do not get paged for platform failures.
  They do get paged when their application logic causes platform failure.

Ownership boundary problem:
  Application team makes an API call pattern that causes Kafka to OOM.
  Who owns the fix? Kafka OOM is a platform failure (platform team).
  Bad usage pattern is an application failure (application team).
  Both teams point at each other. This is the most common operational failure in platform organisations.
  
  Fix: define and enforce consumption quotas. Platform team sets limits.
       Application teams must stay within limits. Exceeding limits: application team's problem.
```

### Pattern: The Embedded Team

```text
What it is:
  Shared infrastructure is owned by the team that uses it most heavily.
  The payments team owns the payment database.
  The IAM team owns the auth platform.
  
Who funds it:
  Product team budget. Infrastructure is a line item in the product team's budget.

Who gets paged:
  The owning team. They feel the pain of their own infrastructure choices.

The problem:
  When the IAM platform needs to serve 10 other product teams,
  the IAM team is now a platform team (by function) with a product team's budget and mandate.
  They will deprioritise platform investments in favour of product features.
  
  This is how platforms become technical debt generators.
  
  When to shift: when > 3 teams depend on a shared service, it needs a dedicated platform team.
```

---

## Ownership Map by Pattern

```text
Pattern 1: Real-Time Updates
  Platform team owns:  WebSocket connection tier, fan-out infrastructure, Kafka
  Application teams own: message schemas, business logic, UI components
  SRE owns: on-call for connection tier capacity, Kafka broker health
  
Pattern 2: Contention
  Nobody "owns" contention — it emerges from application code.
  DB platform team owns: PostgreSQL configuration, connection pooling, index advice
  Application team owns: query patterns, lock granularity, retry logic
  Ownership ambiguity: "Postgres is slow" could be either team's problem.
  Resolution: SLO on DB response time. Below SLO → platform team. Queries causing slowness → application team.

Pattern 3: Multi-Step Processes
  Workflow platform team (if exists): Temporal, Conductor, or custom orchestrator
  Application team: saga steps, compensation logic, business rules
  Finance/compliance: approval for which transactions require compensation

Pattern 4: Scaling Reads
  DB platform team: replica management, connection pooling, backup
  Cache platform team (or same team): Redis cluster management
  Application team: cache key design, TTL selection, invalidation logic

Pattern 7: Long-Running Tasks
  Jobs platform team: job queue infrastructure (SQS/Kafka), scheduler
  Application teams: job logic, idempotency, retry logic
  FinOps: spot instance strategy, worker fleet cost optimisation

Pattern 15: API Gateway
  Platform team (Security + Networking): owns the gateway, routing rules, auth plugins
  Security team: approves all auth policies, rate limit policies
  Application teams: register their services, specify their routing requirements
  SRE: owns gateway uptime SLA, capacity planning
```

---

## The On-Call Model

Who gets paged at 3am matters for architecture decisions. If the team building a component doesn't on-call it, they will not prioritise operability.

```text
For each component in your design, answer:
  1. Who is paged when this component is degraded?
  2. What runbook do they follow? (see Chapter D)
  3. What is the escalation path if they can't resolve it?
  4. What is the acceptable Mean Time to Recovery (MTTR)?
```

The answers to these questions often reveal design flaws:

```text
"The payment orchestrator is paged when it's down."
→ "What's the runbook for 'orchestrator is stuck at step 3 for 5000 payments'?"
→ "...we don't have one."
→ Design flaw: the orchestrator's state visibility is insufficient for debugging.
   Fix: add a stuck-saga dashboard, add admin API to force-advance or cancel a saga.
```

---

## The Funding Conversation

Staff engineers are often asked to present architectural proposals to engineering leadership. The proposal must include cost.

```text
Template for architectural investment proposal:

Problem:
  "Our token introspection latency p99 is 180ms. SLO is 10ms.
   This is causing $X in degraded conversion rate for partners."

Proposed solution:
  "Add Redis cache with 30-second TTL for token validation results."

Cost:
  Engineering: 2 weeks (1 engineer)
  Infrastructure: $150/month (Redis r7g.large)
  Ongoing: 0.5 hours/week operational overhead

Expected outcome:
  Token introspection p99 < 5ms (2× better than SLO)
  DB load reduced 95%
  Engineering cost savings: $Y in reduced on-call incidents

Risk:
  Low. Redis failure falls back to DB (graceful degradation).
  Rollback: disable cache reads (feature flag, minutes).

Decision: approve or defer (with reason).
```

---

*End of Chapter C: Organisational Lens*

---
---

# Chapter D: Incident Response Playbooks

> "Every system fails. The question is how fast you recognise it, how fast you contain it, and how fast you fix it permanently."

---

## The Incident Response Framework

```text
Phase 1: DETECT     (automated; should be <5 minutes)
Phase 2: DIAGNOSE   (human; should be <15 minutes for most incidents)
Phase 3: CONTAIN    (stop the bleeding; data loss stops now)
Phase 4: RECOVER    (restore service to normal operation)
Phase 5: POSTMORTEM (prevent recurrence)
```

A Staff engineer designs systems so that Phase 1 is automatic, Phase 2 has runbooks, Phase 3 has pre-planned options, and Phase 4 has rollback scripts ready.

---

## Incident Playbook: Real-Time System (WebSocket/Fan-Out)

### Symptoms
```text
Alert: websocket.connections.active dropped 40% in 5 minutes
Alert: kafka.consumer.lag growing on fan-out topic
User complaints: "messages aren't delivering"
```

### Diagnose

```text
Step 1: Check connection tier health
  Dashboard: connections by server
  kubectl get pods -n realtime | grep CrashLoopBackOff
  → If one server is down: connections on that server dropped
  
Step 2: Check message bus
  kafka-consumer-groups.sh --describe --group fanout-workers
  → If lag growing: fan-out workers can't keep up
  
Step 3: Check downstream (if fan-out is slow)
  redis.connected_clients → if at limit: Redis connection exhaustion
  websocket.server.write.latency → if high: client connections are slow to write to

Step 4: Determine scope
  All users affected? → Connection tier or message bus
  Subset of users? → Specific server or partition
  Specific message type? → Application logic issue
```

### Likely Causes and Mitigations

```text
Cause A: WebSocket server crashed
  Mitigation: Kubernetes restarts it automatically.
              Users reconnect within 30 seconds.
              Missed messages: delivered on reconnect via offline delivery.
  
Cause B: Kafka consumer group rebalancing storm
  Symptoms: fan-out lag spike, then recovers. Repeating.
  Mitigation: increase session.timeout.ms for consumer group.
              Check for consumer crash loop causing repeated rebalance.
  
Cause C: Redis Pub/Sub memory exhaustion (if using Redis fan-out)
  Symptoms: Redis OOM errors in logs, all fan-out stops
  Mitigation: immediately: restart Redis (data loss: in-flight messages lost)
              Permanent: switch to Kafka for fan-out, Redis for connection registry only
  
Cause D: Fan-out workers overwhelmed by celebrity event
  Symptoms: lag spike correlating with a specific user's activity
  Mitigation: circuit break the celebrity user's fan-out (temporarily stop fan-out for them)
              Route celebrity fan-out to dedicated high-capacity workers
```

### Recovery Steps

```text
For Cause A (server crash): automatic. Monitor reconnect success rate.
For Cause B (rebalance storm): 
  kubectl rollout restart deployment/fanout-workers
  Watch: consumer group lag drops as new workers establish stable assignment.
For Cause C (Redis OOM):
  redis-cli FLUSHDB  ← warning: loses all in-flight messages
  Restart Redis
  All WebSocket connections: force reconnect (ping/pong timeout → client reconnects)
  Users receive missed messages via offline delivery on reconnect.
```

### Permanent Fix Criteria

```text
This incident is considered resolved when:
  - websocket.connections.active returns to baseline (±5%) for 30 minutes
  - kafka.consumer.lag returns to <1000 messages for 30 minutes
  - No new user complaints in the last 15 minutes

Permanent fix:
  - Root cause addressed in postmortem
  - Alert thresholds adjusted if too sensitive/not sensitive enough
  - Runbook updated with new cause/mitigation
```

---

## Incident Playbook: Cache Layer Failure

### Symptoms
```text
Alert: redis.commands.rejected_calls > 0
Alert: api.latency.p99 increased 5× (from 20ms to 100ms)
Alert: db.query.rate increased 5× (cache not absorbing reads)
```

### Diagnose

```text
Step 1: Check Redis health
  redis-cli PING → if no response: Redis is down
  redis-cli INFO memory → check used_memory vs maxmemory
  redis-cli INFO stats | grep rejected_connections
  
Step 2: Check application cache circuit breaker
  Is the app configured to fall back to DB on cache failure?
  Check: application cache error rate vs miss rate (miss=normal; error=Redis down)

Step 3: Scope
  All cache keys affected? → Redis instance/cluster issue
  One key type? → Key TTL storm (all keys of one type expired simultaneously)
  Increasing miss rate, not errors? → Cache stampede in progress
```

### Likely Causes and Mitigations

```text
Cause A: Redis node failure (cluster)
  Redis Cluster promotes replica automatically (30-60 seconds).
  During failover: all requests hit DB.
  DB load: 5-10× normal. May trigger DB alerts.
  Mitigation: DB connection pool must have headroom for this scenario.
              Add in-process cache (Caffeine) as L1 buffer during Redis failover.

Cause B: Redis OOM — maxmemory reached
  Redis evicts keys per eviction policy (allkeys-lru recommended).
  Symptom: cache miss rate spikes, DB load increases.
  Immediate mitigation: increase maxmemory if possible, or clear low-value key prefixes.
  Permanent: shard Redis cluster, add nodes, review TTLs.

Cause C: Cache stampede
  Many keys expired simultaneously → mass DB queries → DB overload.
  Immediate: DB may be overwhelmed. Add circuit breaker to cache layer.
  Mitigation: serve stale values during stampede (stale-while-revalidate).
              Add TTL jitter for all cache writes going forward.
```

---

## Incident Playbook: Multi-Step Process (Saga) Failure

### Symptoms
```text
Alert: saga.in_flight_count growing (sagas not completing)
Alert: saga.stuck_count > 0 (sagas in same state for >2× expected duration)
Alert: payment.success.rate dropped
Customer complaints: "my payment is processing forever"
```

### Diagnose

```text
Step 1: Find stuck sagas
  SELECT id, current_step, created_at, updated_at
  FROM sagas
  WHERE status = 'IN_PROGRESS'
    AND updated_at < NOW() - INTERVAL '10 minutes'
  ORDER BY created_at;

Step 2: Identify the stuck step
  SELECT saga_id, step_name, error_message, attempt_count
  FROM saga_step_executions
  WHERE saga_id = '<stuck_saga_id>'
  ORDER BY created_at;

Step 3: Check downstream service health
  If step 2 is "fraud_check" and it's failing: check fraud service health.
  If step 4 is "bank_authorise" and it's timing out: check bank API latency.

Step 4: Determine scope
  All sagas stuck? → Orchestrator DB failure or orchestrator crash
  Sagas stuck at specific step? → That step's downstream service is failing
  Random sagas stuck? → Idempotency bug (some retries not working)
```

### Likely Causes and Mitigations

```text
Cause A: Downstream service unavailable (e.g., fraud service down)
  Mitigation: apply fallback rule (default allow / default deny based on risk policy).
              Resume stuck sagas once fraud service recovers.
              Sagas waiting: will retry on their next scheduled attempt.

Cause B: Orchestrator crashed mid-saga
  Mitigation: automatic — orchestrator restarts and reads state from DB.
              Verify: orchestrator pod is Running (not CrashLoopBackOff).
              Manual trigger: if not auto-recovering, manually trigger saga resume.

Cause C: Idempotency key collision (rare)
  Symptoms: saga step appears to succeed but saga stays in same state.
  Cause: duplicate saga created with same idempotency key.
  Mitigation: find and cancel the duplicate; let the original proceed.

Cause D: Compensating transaction failing
  Symptoms: saga in 'COMPENSATING' state for >30 minutes.
  Cause: the downstream service for the compensating transaction is down.
  Mitigation: this requires manual intervention. Add to manual_review_queue.
  Process: SRE team contacts downstream service team; manually applies compensation.
```

### Recovery Checklist

```text
□ Identify all stuck sagas
□ Classify by stuck step
□ For each downstream failure: confirm downstream service recovery
□ Trigger saga retries (via admin API or manually update saga state to 'RETRY')
□ Monitor: saga completion rate returns to normal
□ For permanently stuck sagas: manually compensate or escalate to finance team
□ Postmortem: add alert for "saga stuck at step X for >Y minutes"
```

---

## The On-Call Runbook Structure

Every system you design should have a runbook in this format:

```text
RUNBOOK: [System Name] — [Failure Mode]

Severity: P0/P1/P2/P3
Team: [owning team]
Escalation: [next team if unresolved in 30 minutes]

SYMPTOMS:
  - [alert name] with [threshold]
  - [user-visible behavior]

DIAGNOSIS STEPS:
  1. [specific command or dashboard check]
  2. [specific command or dashboard check]
  ...
  
MITIGATION OPTIONS:
  Option A: [name] — [what it does] — [risk: none/low/medium/high]
  Option B: [name] — [what it does] — [risk: none/low/medium/high]

RECOVERY VALIDATION:
  - [metric]: should return to [value] within [timeframe]
  - [metric]: should return to [value] within [timeframe]

ESCALATION CRITERIA:
  Escalate to [team] if not resolved within [timeframe].
  Escalate to [team] if [specific condition].

POSTMORTEM REQUIRED: Yes/No
```

---

*End of Chapter D: Incident Response Playbooks*

---
---

# Chapter E: Architecture Design Review Checklists

> "The questions that prevent the next incident are the ones you ask before it ships."

---

## How to Use These Checklists

These are the questions a Principal Engineer asks when reviewing a design. Use them in two ways:

1. **Before a system design interview**: as a self-test. If you can't answer all questions for your design, find the answer.
2. **As an interviewer signal**: if you proactively address items on this list without being prompted, that is a Staff-level signal.

---

## Pattern 1: Real-Time Updates — Design Review Checklist

```text
Connectivity
□ What protocol? WebSocket, SSE, Long Poll? Justified?
□ What happens when a client's connection drops? Auto-reconnect within how long?
□ What is the heartbeat interval? Is it shorter than proxy idle timeout?
□ How are connections authenticated? JWT in handshake header? What happens on token expiry?

Fan-Out
□ What is the fan-out ratio? (messages per sender per second × recipients)
□ Is there a celebrity/hot-user case? How is it handled?
□ What is the maximum fan-out latency guarantee? (e.g., p99 < 500ms)
□ What happens if the fan-out bus (Redis/Kafka) is down?

Offline Delivery
□ What is the offline message retention policy?
□ How are messages replayed on reconnect?
□ Is the replay idempotent? (safe to deliver twice)

Capacity
□ What is the peak concurrent connection count?
□ What is the memory cost per connection? Total memory for peak connections?
□ What is the peak outbound bandwidth? Does it exceed server NIC capacity?
□ How many servers are needed at peak?

Observability
□ Alert: connection drop > 10% in 5 minutes?
□ Alert: fan-out lag growing?
□ Alert: p99 delivery latency > SLO?
```

## Pattern 2: Contention — Design Review Checklist

```text
Conflict Analysis
□ What is the expected concurrent write rate on the contested resource?
□ What is the expected collision rate? Does it justify pessimistic or optimistic locking?
□ Is there a hot row? What is the maximum write rate to that row?
□ Could the hot row be eliminated by redesigning the data model?

Lock Strategy
□ Is SELECT FOR UPDATE held across any external calls? (This is almost always wrong)
□ What is the maximum lock hold time?
□ Are locks acquired in consistent order to prevent deadlocks?
□ What is the retry strategy on lock failure?

Idempotency
□ Is every write operation idempotent?
□ Is the idempotency key generated by the client (not server)?
□ What is the idempotency key expiry?
□ What happens if the same idempotency key is submitted after expiry?

Capacity
□ At what write rate does the chosen locking strategy break down?
□ What is the plan when that rate is reached?
□ Has the pre-allocation pattern been considered for the hot resource?
```

## Pattern 3: Multi-Step Processes — Design Review Checklist

```text
State Machine
□ Are all valid states explicitly enumerated?
□ Are all valid transitions defined?
□ Are invalid transitions explicitly rejected?
□ Is the state machine persisted durably before each step is executed?

Compensation
□ Does every reversible step have a compensating action?
□ Are compensating actions idempotent?
□ What happens if a compensating action fails permanently?
□ Is there a manual review queue for permanently stuck compensations?

Idempotency
□ Is every step call idempotent?
□ Is the idempotency key derived deterministically from the saga ID?
□ What happens if the orchestrator retries a step that already succeeded?

Operations
□ Is there a dashboard showing in-flight sagas by step?
□ Is there an alert for sagas stuck in the same state for > 2× expected duration?
□ Is there an admin API to force-cancel or force-advance a saga?
□ What is the maximum saga age before it's considered abandoned?
```

## Pattern 4: Scaling Reads — Design Review Checklist

```text
Cache Design
□ What is the cache hit ratio target?
□ What is the expected cache size for this hit ratio? Is it within budget?
□ What is the cache invalidation strategy? TTL only? Active invalidation? CDC?
□ What is the maximum acceptable staleness?
□ Has the cache stampede case been handled? (XFetch, jitter, or mutex)

Consistency
□ After a write, will the user see their own update? (read-your-writes)
□ Is it possible for a user to see data go backwards? (monotonic reads)
□ If so, is this acceptable for this use case?

Failure Modes
□ What happens if Redis fails? Does the system degrade gracefully?
□ What happens if the read replica falls >5 seconds behind?
□ Is there a circuit breaker to stop writing to a lagging replica?

Capacity
□ What is the DB read rate with no cache (cold start scenario)?
□ Can the DB handle that load if cache fails?
□ If not: what is the fallback? (serve stale from in-process cache)
```

## Pattern 5: Scaling Writes — Design Review Checklist

```text
Write Path
□ What is the peak write rate?
□ What is the maximum acceptable write latency?
□ At what write rate does the current storage system saturate?
□ What is the plan for when that rate is reached?

Durability
□ What is the acceptable data loss window? (RPO)
□ Does the chosen write buffering strategy meet this RPO?
□ If writes go to Kafka, what happens if Kafka is unavailable?

Ordering
□ Are there ordering requirements? (e.g., events for a user must be ordered)
□ If using Kafka: is the partition key designed to maintain required ordering?
□ If order is not required: is that explicitly stated and agreed?

Hot Shards
□ Is the shard key uniformly distributed?
□ Has the hot shard scenario been considered?
□ What is the plan for resharding?
```

## Pattern 7: Long-Running Tasks — Design Review Checklist

```text
Worker Design
□ Are workers idempotent? (safe to process the same job twice)
□ What happens if a worker is killed mid-job?
□ How long before the job becomes visible again for retry?
□ Is the visibility timeout longer than the maximum job duration?

Job State Tracking
□ Is job progress visible externally (to API callers)?
□ Is there a stuck job detector (job in RUNNING for > 2× expected duration)?
□ What is the maximum job retry count before DLQ?
□ Is the DLQ monitored with an alert?

Capacity
□ How many workers are needed at peak job submission rate?
□ What is the autoscaling trigger? (queue depth? job age?)
□ What is the maximum acceptable job queue latency?
□ Are GPU/specialized workers used efficiently (spot instances)?
```

## Patterns 8–15: Quick Checklists

```text
Pattern 8: Search
□ Is Elasticsearch derived from the DB (via CDC)? Never the primary store.
□ What is the maximum acceptable index lag?
□ Is the cluster sized correctly? (shard size 10-65GB per ES 8.x guidelines)
□ Is there tenant isolation in the index? (user A cannot see user B's documents)
□ How is the index rebuilt after a cluster failure?

Pattern 9: Notifications
□ Is there a per-user, per-channel daily rate limit?
□ Is there notification collapsing? (5 likes → "N people liked your post")
□ What is the fallback channel if push fails?
□ Is there a DLQ for undeliverable notifications?
□ Is there an unsubscribe mechanism for every notification type?

Pattern 10: Rate Limiting
□ Is the rate limiter idempotent? (a request is counted once, even if checked multiple times)
□ What happens when Redis is unavailable? (fail open or closed?)
□ Are Retry-After headers returned on 429 responses?
□ Is there adaptive rate limiting? (tighten limits under backend stress)
□ Is rate limiting enforced at the edge (CDN) for DDoS scenarios?

Pattern 11: Scheduling
□ What happens if the job runs twice? (is it idempotent?)
□ What happens if the job misses its window? (catch-up or skip?)
□ Is there overlap prevention? (INSERT ON CONFLICT DO NOTHING pattern)
□ Are long-running jobs protected with heartbeat/lock extension?

Pattern 12: Distributed Transactions
□ Is the compensation action idempotent?
□ What happens if compensation fails permanently?
□ Is there a manual review queue for permanently stuck sagas?
□ Is saga state durably persisted before each step is called?
□ Can the system handle a replica of the orchestrator's DB falling behind?

Pattern 13: Multi-Region
□ Is data residency requirement documented per entity type?
□ Is there a single "home region" per entity to prevent write conflicts?
□ What is the failover time if a region goes down?
□ Is the user experience during cross-region fallback acceptable?
□ Is cross-region data transfer cost included in the cost model?

Pattern 14: Event-Driven
□ Is the schema registry enforcing backward compatibility?
□ Are all consumers idempotent?
□ Is consumer lag the primary health metric?
□ What is the plan when a consumer is down for 24 hours? (Kafka retention)
□ Has event replay been tested from a new consumer group?

Pattern 15: API Gateway
□ Is the gateway horizontally scalable?
□ Is there a circuit breaker per upstream service?
□ What happens when the auth service is down? (JWKS cache TTL)
□ Is JWT validation cached? At what TTL?
□ Is there an emergency bypass for the gateway? (for disaster recovery)
```

---

*End of Chapter E: Architecture Design Review Checklists*

---
---

# Chapter F: Threat Models

> "Security is not a feature. It is the absence of vulnerabilities. The only way to achieve it is to enumerate what can go wrong."

---

## The Threat Modeling Structure

For every pattern, threats are evaluated on two axes:
- **What is being attacked?** (confidentiality, integrity, availability)
- **Who is the attacker?** (external attacker, malicious insider, compromised service, accidental misconfiguration)

The STRIDE framework:
- **S**poofing: impersonating another identity
- **T**ampering: modifying data or code
- **R**epudiation: denying an action occurred
- **I**nformation Disclosure: exposing data to unauthorised parties
- **D**enial of Service: making the system unavailable
- **E**levation of Privilege: gaining access beyond what was authorised

---

## Pattern 1: Real-Time Updates — Threat Model

```text
THREAT 1: Connection Hijacking (Spoofing)
  Attack: attacker intercepts WebSocket upgrade request, substitutes their own token
  Impact: attacker receives messages intended for the victim
  Mitigation: 
    - TLS on all connections (prevents interception)
    - JWT in Upgrade request Authorization header (not URL — URL is logged)
    - Short-lived access tokens (15-minute expiry); refresh tokens via separate endpoint
    - Validate JWT on every WebSocket message for high-security channels

THREAT 2: Message Injection (Tampering)
  Attack: authenticated user sends messages purporting to be from another user
  Impact: false messages appear in conversation
  Mitigation:
    - Server sets sender_id from JWT claims, never from message body
    - Client cannot forge the sender_id field
    - "The server signs the message" is the design principle

THREAT 3: Connection Flood (DoS)
  Attack: attacker opens 100,000 WebSocket connections consuming server memory
  Impact: legitimate users cannot connect
  Mitigation:
    - Rate limit: max N connections per IP per minute
    - Rate limit: max M connections per authenticated user
    - CAPTCHA/proof-of-work for unauthenticated connection requests
    - CDN/load balancer DDoS protection (Cloudflare, AWS Shield)

THREAT 4: Presence Disclosure (Information Disclosure)
  Attack: user queries presence API to determine when specific people are online
  Impact: stalking, surveillance
  Mitigation:
    - Presence visible only to mutual connections (not all users)
    - "Last seen" shows approximate time, not exact
    - Users can disable presence entirely
    - Presence queries: rate-limited to prevent enumeration

THREAT 5: Replay Attack (Tampering)
  Attack: attacker captures a valid message and replays it later
  Impact: duplicate messages, repeated actions
  Mitigation:
    - Each message has a unique message_id (UUID)
    - Server deduplicates by message_id within a time window
    - Message timestamp: server rejects messages older than 5 minutes
```

---

## Pattern 2: Contention — Threat Model

```text
THREAT 1: Race Condition Exploitation (Elevation of Privilege)
  Attack: attacker exploits the window between balance check and debit
  Example: TOCTOU (Time-of-Check to Time-of-Use) in payment systems
  Impact: negative balance, double claim of a resource
  Mitigation:
    - Atomic compare-and-swap: the guard condition IS the balance check
    - Never: check, then act. Always: act with guard condition atomically.

THREAT 2: Lock Starvation (DoS)
  Attack: attacker holds a distributed lock open indefinitely
  Example: acquire lock, then make infinite slow loop
  Impact: no other process can acquire the lock; system deadlocks
  Mitigation:
    - All distributed locks MUST have a TTL (no indefinite locks)
    - Heartbeat renewal only if holder is still active
    - Alert: lock held for > 2× expected duration

THREAT 3: Idempotency Key Reuse (Tampering)
  Attack: attacker reuses a valid idempotency key from a previous request
  Example: replay a payment request with a used idempotency key
  Impact: payment charged twice (if server doesn't deduplicate correctly)
  Mitigation:
    - Idempotency keys: scoped to (user_id, key) pair, not just key alone
    - An idempotency key from user A cannot be reused by user B
    - Expiry: idempotency keys expire after 24 hours
    - Revocation: if a user's session is compromised, revoke all their pending idempotency keys
```

---

## Pattern 10: Rate Limiting — Threat Model

```text
THREAT 1: Rate Limit Bypass via IP Rotation (DoS)
  Attack: attacker routes requests through many IPs to avoid per-IP limit
  Example: credential stuffing via botnet with thousands of IPs
  Impact: brute force attacks succeed despite "rate limiting"
  Mitigation:
    - Rate limit by user_id (not just IP) for authenticated endpoints
    - Rate limit by device fingerprint (for unauthenticated endpoints)
    - CAPTCHA after N failures from any IP to any account
    - Reputation-based IP scoring (block known botnet IPs via threat intelligence)

THREAT 2: Account Enumeration (Information Disclosure)
  Attack: attacker determines valid vs invalid usernames by response timing or messages
  Example: "user not found" (100ms) vs "wrong password" (200ms) reveals valid accounts
  Impact: attacker builds list of valid accounts for targeted attacks
  Mitigation:
    - Identical response time for "user not found" and "wrong password"
    - Identical response message for both cases
    - Rate limit per IP regardless of whether account exists
    - Add jitter to response times

THREAT 3: Rate Limit Poisoning (DoS)
  Attack: attacker makes many requests pretending to be victim user
  Impact: victim's rate limit is exhausted; victim is locked out
  Mitigation:
    - Rate limit by (user_id, IP pair), not user_id alone
    - Alert: same user_id from many IPs simultaneously (credential sharing or attack)
    - Emergency unlock API for customer support (with strong auth)
    - Distinguish: rate limit exceeded vs account lockout (different actions)

THREAT 4: Timing Attack on Rate Limit Check (Information Disclosure)
  Attack: measure response time to determine whether a Redis key exists
  Impact: reveals information about system state
  Mitigation:
    - Constant-time responses (not a practical concern for most systems; matters for auth systems)
    - Reduce the sensitivity of timing signals by adding jitter to response time
```

---

## Pattern 15: API Gateway — Threat Model

```text
THREAT 1: JWT Replay (Spoofing)
  Attack: attacker captures a valid JWT and reuses it after the user logs out
  Impact: attacker can make API calls as the victim
  Mitigation:
    - Short-lived access tokens (15-minute TTL) minimise replay window
    - Token revocation list in Redis: on logout, add token_id to revocation list
    - Gateway checks revocation list on every request (adds ~1ms from Redis)
    - Alternatively: short TTL + refresh token rotation (logout invalidates refresh token)

THREAT 2: JWT Algorithm Confusion (Spoofing)
  Attack: attacker sends JWT with alg:none or changes from RS256 to HS256
  Impact: forged JWT accepted by gateway
  Mitigation:
    - Gateway MUST explicitly accept only expected algorithms
    - Never allow alg:none
    - Use a JWT library that validates algorithm against expected (not against JWT header)
    - pin the algorithm: "we only accept RS256 from our IAM service"

THREAT 3: SSRF via Gateway Routing (Elevation of Privilege)
  Attack: attacker crafts a request that causes the gateway to forward to an internal service
  Example: POST /api/internal/admin endpoint not meant for external traffic
  Impact: internal endpoints exposed
  Mitigation:
    - Allowlist-based routing: only forward to explicitly configured upstream services
    - Never passthrough the upstream URL from the request
    - Internal services: not reachable from external network (VPC isolation)
    - Gateway: explicitly block paths like /actuator, /admin, /internal

THREAT 4: Tenant Escape (Elevation of Privilege)
  Attack: tenant A's request accesses tenant B's data via the gateway
  Example: API key for tenant A with tenant_id in JWT, but request accesses tenant B's resources
  Impact: cross-tenant data breach
  Mitigation:
    - Gateway extracts tenant_id from JWT and adds X-Tenant-Id header
    - Upstream services enforce: resource tenant_id must match X-Tenant-Id header
    - Gateway does NOT accept X-Tenant-Id header from clients (it's set by gateway only)
    - Regular penetration testing of tenant isolation boundaries

THREAT 5: API Key Leakage (Information Disclosure + Spoofing)
  Attack: API key committed to source code, exposed in logs, or visible in URLs
  Impact: attacker makes unlimited API calls as the key owner
  Mitigation:
    - API keys: never in URLs (visible in logs, browser history, proxies)
    - API keys: in Authorization header or custom header only
    - Automated API key scanning: gitleaks, truffleHog in CI/CD pipeline
    - API key rotation: keys auto-expire after 90 days
    - Immediate revocation API: compromise response time <5 minutes
    - Alert: usage from unexpected geographies (API key used from new country)
```

---

*End of Chapter F: Threat Models*

---
---

# Pattern 1: Real-Time Updates

> "The question isn't just 'how do I push data to clients.' It's 'how do I push data to millions of clients, reliably, in order, without melting the server.'"

---

## The Shape

A client needs to see data change without refreshing. Users expect to see a new message the moment it arrives, not when they next check. The server needs to push state to clients, not wait for them to ask.

This sounds simple. At one user it is simple. At ten million concurrent users, it is one of the hardest infrastructure problems in distributed systems.

---

## When You See This Pattern

An interviewer asks you to design any of these:
- WhatsApp / Slack / iMessage
- Live sports scores / stock ticker / trading platform
- Google Docs / Figma collaborative editing
- Notification bell / activity feed (real-time part)
- Ride-sharing (driver location on map)
- Online presence ("Alice is typing...")
- Live auction / seat reservation (real-time availability)

The signal: **the client must see a change before it asks for it**.

---

## The Core Tension

```text
More connections ←————————————————→ Lower server cost
(better real-time)                   (simpler operations)

Push everything ←————————————————→ Push only changes
(no staleness)                       (lower bandwidth)

Ordered delivery ←————————————————→ Higher throughput
(correct UX)                         (no ordering guarantee)
```

Every real-time design is a negotiation between these three axes. The "right" answer depends entirely on what the product requires.

---

## Building Blocks

### 1. Short Polling

The client asks the server for updates every N seconds. The server answers immediately.

```text
Client:  GET /messages?since=1700000000    (every 5 seconds)
Server:  200 OK { messages: [] }           (99% of responses: empty)
Client:  GET /messages?since=1700000000    (5 seconds later)
Server:  200 OK { messages: [new msg] }    (finally, something)
```

**When to use it**: Update frequency measured in minutes. Simplicity is paramount. Behind restrictive proxies. Quick prototypes.

**The math that kills it**: 1M users polling every 5 seconds = 200,000 req/sec. At 1KB per response, that's 200 MB/sec. 99% of those responses are empty. You're paying the full cost of a distributed system to answer "nothing new."

**The failure mode that surprises people**: Poll interval = 5s. User sends a message. Recipient waits up to 5 seconds to see it. Acceptable for a status dashboard. Not acceptable for a chat application.

---

### 2. Long Polling

The client asks the server for updates. The server **holds the connection open** until something new arrives (or a timeout fires). The client immediately reconnects.

```text
Client: GET /messages?since=T             (connects)
Server: [holds connection — 22 seconds]
        [new message arrives]
Server: 200 OK { messages: [new msg] }    (responds immediately)
Client: GET /messages?since=T+22          (immediately reconnects)

Timeout path:
Client: GET /messages?since=T             (connects)
Server: [holds connection — 30 seconds, nothing arrives]
Server: 200 OK { messages: [] }           (heartbeat timeout)
Client: GET /messages?since=T             (immediately reconnects)
```

**What makes this work**: Response arrives within seconds of the event, not within the poll interval. Near-real-time without a persistent connection.

**The thread problem and its fix**: If the server uses one thread per held connection, 100k users = 100k threads = 50GB RAM just for stacks. Fix: async/non-blocking server (Spring WebFlux, Node.js, Netty). Each held connection is a suspended coroutine, not a thread.

**The proxy timeout problem**: Corporate proxies kill idle connections after 60–90 seconds. Set server-side timeout shorter (25–28 seconds) so the server always fires a heartbeat response before the proxy kills the connection.

**When to use it**: Near-real-time required, can't use WebSockets (restrictive firewalls/proxies), unidirectional server→client updates, moderate concurrent user count.

---

### 3. Server-Sent Events (SSE)

A persistent one-way HTTP connection where the server streams events to the client continuously. The browser's built-in `EventSource` API handles reconnection automatically.

```text
Client: GET /events                       (opens SSE connection)
Server: Content-Type: text/event-stream
        [holds connection open forever]

Server pushes events:
  data: {"type":"message","id":"msg-123","text":"Hello"}\n\n
  data: {"type":"typing","user":"Alice"}\n\n
  id: 1700000045\n
  data: {"type":"message","id":"msg-124","text":"World"}\n\n

Client disconnects (network drop):
  Browser: reconnects automatically after 3s
  Browser: sends Last-Event-ID: 1700000045
  Server: replays events since that ID
```

**Why SSE is underused**: Most engineers reach for WebSockets without considering SSE. For the majority of real-time use cases — where the server pushes and the client only reads — SSE is simpler, works over standard HTTP/2, and has built-in reconnect with event replay.

**The HTTP/2 multiplexing advantage**: Over HTTP/1.1, browsers allow 6 connections per domain. Multiple SSE streams exhaust this quickly. Over HTTP/2, all SSE streams share one TCP connection. No limit.

**The heartbeat requirement**: Proxies kill idle connections. Send a heartbeat event every 25 seconds:
```
event: heartbeat
data: {}
```

**When to use it**: Server pushes to browser, client doesn't send data back. Notifications, live feeds, progress bars, security event streams, dashboard metrics.

---

### 4. WebSockets

A full-duplex, persistent TCP connection. Both sides can send data at any time. The connection starts as HTTP and upgrades.

```text
Client → Server: HTTP GET /ws
                 Upgrade: websocket
                 Connection: Upgrade
                 
Server → Client: HTTP 101 Switching Protocols

Now bidirectional:
Client → Server: {"type":"message","text":"Hello","room":"general"}
Server → Client: {"type":"message","from":"Alice","text":"Hello","ts":1700000000}
Server → Client: {"type":"presence","user":"Bob","status":"online"}
Client → Server: {"type":"typing","room":"general"}
```

**The architecture problem at scale**: A WebSocket connection is stateful — it lives on one server. Load balancers must route future requests for that connection to the same server (sticky sessions). This is fine until:
- That server dies (all its connections drop)
- You need to broadcast a message to users spread across 50 servers

**The fan-out architecture**: Decouple message delivery from connection management.

```text
Message arrives at Server A:
  Server A → publishes to Pub/Sub (Redis, Kafka) → topic: room-general

All servers subscribe to room-general:
  Server A: delivers to its local WebSocket connections for room-general
  Server B: delivers to its local WebSocket connections for room-general
  Server C: delivers to its local WebSocket connections for room-general

Result: message reaches all users in room-general regardless of which server holds their connection.
```

**The connection count problem**: At 1M concurrent WebSocket connections, each connection uses ~10KB of kernel memory (TCP socket) + ~20KB application state. 1M × 30KB = 30GB RAM just for connections. You need a connection tier separate from your business logic tier.

**When to use it**: Bidirectional real-time (chat, collaborative editing, multiplayer games, live trading), high-frequency client→server data, where SSE isn't sufficient.

---

### 5. Push Notifications (Mobile)

For mobile clients that aren't foreground-active, WebSockets and SSE don't apply. Push notifications go through platform-specific infrastructure (APNs for iOS, FCM for Android).

```text
Your Server → FCM/APNs → Device (even if app is closed)

Your server never connects directly to the device.
FCM/APNs maintains persistent connections to all enrolled devices.
Your server simply tells FCM/APNs what to deliver.
```

**The delivery guarantee problem**: FCM/APNs is best-effort. Messages can be dropped, delayed, or delivered out of order. Design for this:
- Notifications are hints, not data delivery mechanisms
- When app opens: sync full state from server via REST
- Notification payload: minimal ("you have 3 new messages"), not the actual content

**The token management problem**: Device tokens expire, rotate, and become invalid (app uninstall). Your server must handle `INVALID_REGISTRATION` responses from FCM/APNs by removing the stale token. Accumulating dead tokens causes wasted requests and silently missing deliveries.

---

### 6. Fan-Out

Fan-out is not a protocol — it is the **delivery strategy** for pushing a single event to multiple recipients.

```text
Fan-out on Write (push model):
  User A posts a message to 500 followers
  → Write event to 500 followers' inboxes immediately
  → Read inbox: O(1) — pre-populated
  → Write cost: O(followers) per event
  → Celebrity problem: 10M followers × every tweet = 10M writes

Fan-out on Read (pull model):
  User A posts a message
  → Write once to A's timeline
  → Reader constructs their feed: "fetch latest from all people I follow"
  → Read cost: O(following count) per read
  → Fine for users who follow 50 people; catastrophic for users who follow 5000

Hybrid (Twitter/Instagram approach):
  Regular users (<10K followers): fan-out on write
  Celebrity users (>10K followers): fan-out on read
  Feed construction: merge pre-built timeline + recent celebrity posts
```

**The SCC lens on fan-out**: Fan-out moves work from the read path to the write path. This is a State (where does the inbox data live) + Concentration (celebrity accounts) problem. The hybrid model is the answer to the concentration problem.

---

### 7. Presence Systems

"Alice is online." "Bob is typing." These are among the hardest real-time problems because:
- Presence changes at very high frequency
- State must be globally consistent
- Connections die without notice (mobile, network drop)

```text
Presence Architecture:

Client (heartbeat every 5s):
  → WebSocket server: { type: "heartbeat", user: "alice" }

WebSocket server:
  → Redis SETEX presence:alice "online" 15   (15-second TTL)
  
  If heartbeat stops: TTL expires → alice goes offline automatically
  No explicit "disconnect" needed.

Presence query:
  → Redis: MGET presence:alice presence:bob presence:carol
  → Returns: ["online", nil, "online"]  (nil = offline)
```

**The typing indicator problem**: "Bob is typing" should disappear 3 seconds after Bob stops typing. Broadcast typing start, set a TTL, broadcast typing stop (or let the TTL expire).

**The thundering herd on login**: When a popular user comes online, every one of their followers should be notified. At 1M followers, this is 1M presence fan-out events per login. Mitigations:
- Don't fan out presence to followers; have clients poll periodically for active friends
- Only fan out presence to users who are currently in the same "room" or "conversation"
- Throttle: batch presence updates (send every 10 seconds, not every state change)

---

## Canonical Problem 1: Design WhatsApp (Messaging Layer)

### Requirements
- 2B users; ~100M messages/day
- Message delivery: near-real-time (<200ms in same region)
- Offline delivery: messages queued, delivered when user reconnects
- Read receipts: single check (delivered), double check (read)
- End-to-end encryption

### The Design

```text
Mobile Client
    │
    │ (WebSocket — always connected when app open)
    ▼
Connection Tier (WebSocket servers, stateless logic)
    │
    │ (publishes to message queue)
    ▼
Message Queue (Kafka: partitioned by conversation_id)
    │
    │ (consumed by)
    ▼
Delivery Service
    ├── Is recipient online? → push via WebSocket server
    └── Is recipient offline? → store in Message Store
                                 push notification via APNs/FCM
    │
    ▼
Message Store (Cassandra)
    Partition key: conversation_id
    Clustering key: message_id (time-ordered)
```

**Why Cassandra**: Append-only writes (messages are never updated), high write throughput, time-ordered reads by conversation. Exactly the Cassandra sweet spot.

**Why Kafka between tiers**: Decouples sending from delivery. If the delivery service is slow or down, messages queue up without the sender experiencing failure. The sender gets ack when Kafka receives the message — not when the recipient's phone receives it.

**Offline delivery**:
```text
User comes online:
  1. WebSocket connection established
  2. Client sends: { type: "sync", last_received_id: "msg-456" }
  3. Server queries Message Store: SELECT * WHERE conversation_id=? AND msg_id > msg-456
  4. Delivers pending messages in order
  5. Client sends read receipts
```

**Read receipts**:
```text
Single tick (sent):    message stored in Message Store
Double tick (delivered): recipient's device acknowledges receipt
Blue ticks (read):     recipient's app sends explicit read event
```

Each receipt event flows back through the same pipeline in reverse.

### Staff-Level Consideration: The 2B Users Problem

At 2B registered users with ~100M daily active:
- WebSocket connections at peak: ~50M concurrent
- Connection tier: ~50K connections per server (tuned kernel) → ~1,000 servers
- Message Store: Cassandra cluster, sharded by conversation_id
- Key insight: most connections are idle most of the time — connection servers are I/O-bound, not CPU-bound

The real bottleneck is **connection management**, not message processing. Separate your connection tier (high concurrency, low CPU) from your processing tier (lower concurrency, higher CPU).

---

## Canonical Problem 2: Design a Live Sports Score System

### Requirements
- 50M concurrent viewers during major match
- Score updates: within 1 second of real-world event
- One-way: server → clients only
- No user-generated data

### The Design

```text
Score Update Source
  (stadium system, official feed)
        │
        ▼
Ingestion Service
  (validates, deduplicates, stores to DB)
        │
        ▼
Kafka Topic: match-events
  (partitioned by match_id)
        │
        ▼
Fan-out Workers
  (one per match; maintains in-memory state)
        │
        ├── CDN Edge Cache (invalidate on score change)
        │
        └── WebSocket/SSE servers
              │
              ▼
          50M connected clients
```

**Why SSE over WebSockets here**: Clients only receive; they never send. SSE is sufficient and simpler to load-balance (stateless HTTP). WebSockets would add bidirectional complexity without benefit.

**Why CDN as a layer**: Not all 50M clients need a persistent connection. For score displays embedded in websites, a CDN-cached endpoint with `Cache-Control: max-age=1` reduces origin load by 95%. Only dedicated score-watching apps need real push.

**The fan-out math**: 50M clients watching 1 match. Score changes every ~5 minutes on average. 50M × 1 message/5min = 167K messages/sec. This is not a high-throughput problem per match — it is a **connection management** problem. The bottleneck is maintaining 50M persistent connections, not the message rate.

### Staff-Level Insight

The interviewer is testing whether you separate the two distinct problems:
1. **High fan-out** (one event → 50M clients): solved by the CDN layer + SSE servers
2. **Low latency** (event in database within 1 second): solved by the Kafka pipeline

They are different problems with different solutions. Junior engineers conflate them.

---

## Canonical Problem 3: Design Collaborative Editing (Google Docs)

### Requirements
- Multiple users editing the same document simultaneously
- Changes appear in near-real-time for all editors
- No conflicts / coherent final state

### The Hard Problem: Conflict Resolution

```text
Document state: "Hello World"

User A (at position 5): inserts "Beautiful " → "Hello Beautiful World"
User B (at position 5, same time): inserts "Cruel " → "Hello Cruel World"

Server receives both. What should the document say?
```

This is the **Operational Transformation (OT)** or **CRDT** problem. The answer determines everything about the architecture.

**Operational Transformation (OT)**:
Each operation is transformed against concurrent operations before being applied. The server is the authority: it sequences all operations and sends transformed versions to all clients.

```text
Server receives:
  Op A: insert("Beautiful ", pos=5)
  Op B: insert("Cruel ", pos=5)  [concurrent]

Server orders them (e.g., A first, then B):
  Apply A: "Hello Beautiful World"
  Transform B against A: insert("Cruel ", pos=15)  [position shifted]
  Apply B: "Hello Beautiful Cruel World"

All clients converge to same state.
Server is the ordering authority.
```

**CRDTs (Conflict-Free Replicated Data Types)**:
Data structures that merge automatically regardless of operation order. No central authority needed. Used by Figma, Notion.

**The architecture consequence**: OT requires a server to sequence operations. CRDTs can work peer-to-peer. For most enterprise products: OT with a central server is simpler and correct.

```text
WebSocket Architecture for Collaborative Editing:

Client A ─── WebSocket ──→ Document Server
                              │
Client B ─── WebSocket ──→   │ (same server; sticky session)
                              │
                         Operation Log (append-only, Kafka)
                              │
                         Document State (Redis: current snapshot)
                              │
                         Storage (Postgres/S3: periodic checkpoints)
```

**Sticky sessions are required**: OT requires the server to have the current document state to transform operations. Either:
- Route all clients for a document to the same server (sticky)
- Or replicate document state across all servers (more complex)

For most real-world systems: sticky sessions at the document level, with failover (new server loads from checkpoint + replays operation log).

---

## Deep Dive 1: Millions of WebSocket Connections

### The Problem

A naive WebSocket server on a typical 8-core machine handles ~10,000 concurrent connections before CPU becomes the bottleneck. At 1M connections, you need 100 servers just for connection management. At 10M connections, 1,000 servers.

The problem is not the connections themselves — the OS can handle ~1M sockets per machine with tuning. The problem is **what happens when you need to send a message to a user whose connection is on server #457 out of 1,000**.

### The Solution: Connection Gateway + Message Bus

```text
Architecture at scale:

Users → Connection Gateway Tier (stateless within tier, stateful per connection)
              │
              │ (each server registers: "user X is connected here")
              ▼
         Service Registry (Redis: user_id → server_id)
              │
              ▼
         Message Bus (Kafka / Redis Pub/Sub)
              ▲
              │ (services publish to user's channel)
Application Services

Delivery path:
  1. Application service wants to send to user X
  2. Looks up: service registry says user X is on server #457
  3. Publishes to Redis channel: channel:server-457
  4. Server #457 has subscribed to its channel
  5. Server #457 looks up its local connection table: user X is socket #12345
  6. Sends message on socket #12345
```

**Kernel tuning for high connection count**:
```bash
# Max open file descriptors (each socket = 1 fd)
ulimit -n 1000000
# Also in /etc/security/limits.conf

# TCP settings
net.ipv4.tcp_tw_reuse = 1
net.core.somaxconn = 65535
net.ipv4.ip_local_port_range = 1024 65535
```

**The memory budget**:
```text
Per WebSocket connection:
  Kernel TCP socket:    ~3KB
  TLS state:            ~8KB
  Application (user_id, room_id, last_ping): ~2KB
  Total: ~13KB

1M connections × 13KB = 13GB RAM
At $0.05/GB/hr: ~$15,000/month just for connection memory at 1M connections
```

---

## Deep Dive 2: Ordering Guarantees

### Why Ordering Is Hard

```text
User A sends: "Hello" (T=0)
User A sends: "World" (T=1)

Both messages arrive at different servers (load-balanced).
Server 2 processes "World" faster.
Bob receives: "World", then "Hello".
```

### Solutions

**Ordering by producer**: Client assigns sequence numbers. Server rejects out-of-order messages (or buffers them).

**Ordering by partition key**: In Kafka, all messages from the same sender go to the same partition (partition key = sender_id). Partitions are strictly ordered. Consumers see messages in order.

**Ordering by conversation**: All messages in a conversation get a monotonically increasing sequence number assigned by a single sequencer service. Recipients buffer messages and reorder before displaying.

```text
Sequencer Service:
  INCR conversation:conv-123:seq  → returns 42
  Message stored with seq=42

Recipient:
  Receives messages: [seq=44, seq=42, seq=43]  (out of order, network reordering)
  Buffers until seq=42 arrives
  Displays: [42, 43, 44]  (in order)
```

**What to tell the interviewer**: Strict ordering across all conversations is expensive. Ordering within a single conversation is achievable. Define the ordering guarantee scope upfront — this is a staff-level question because junior engineers assume global ordering is required when per-conversation ordering is sufficient.

---

## Deep Dive 3: Offline Delivery

### The Problem

A message is sent to a user who is offline. When they come online, they must receive it reliably. In order. Once (not twice).

### The Architecture

```text
Message Store (Cassandra):
  Partition key: recipient_user_id
  Clustering key: message_id (snowflake: time-ordered)
  
  Every message to an offline user is written here.
  
On reconnect:
  Client: { last_received_id: "msg-ABC" }
  Server: SELECT * FROM messages WHERE recipient = ? AND msg_id > ? ORDER BY msg_id
  Server: delivers in order
  Client: acks each message
  Server: marks as delivered (but never deletes — delivery is idempotent)
```

**The idempotency requirement**: The client may reconnect mid-delivery. Messages might be delivered twice. The client must deduplicate by message_id. The server must not charge for delivery failures.

**Retention policy**: How long do you keep undelivered messages? WhatsApp: 30 days. After 30 days, the message is dropped and the sender is notified. This is a product decision, not a technical one — but the technical implication is a TTL on the offline message store.

---

## Interview Calibration

### What a Senior Engineer Answers

- Picks WebSockets for everything
- Describes the connection upgrade handshake
- Mentions pub/sub for fan-out
- Handles the "what if a server dies" question with "restart it"

### What a Staff Engineer Answers

- Starts with: "What are the delivery requirements? Bidirectional or server-to-client only? What latency? What ordering guarantee? What's the scale?"
- Distinguishes SSE from WebSockets and chooses deliberately
- Separates the connection management problem from the message delivery problem
- Addresses offline delivery, reconnection, and deduplication as first-class concerns
- Quantifies the connection cost and identifies the binding constraint
- Discusses the fan-out strategy and celebrity/hot user problem
- Identifies where ordering is required and scopes it minimally

### The Tradeoff They Expect You to Navigate

> "50M users. Score updates every 30 seconds. Should I use WebSockets or SSE?"

**Wrong answer**: "WebSockets, they're more powerful."

**Right answer**: "SSE. Clients only receive — no bidirectional need. SSE is simpler to load-balance (stateless HTTP vs sticky WebSocket), works over HTTP/2, has built-in reconnect with event replay, and is sufficient for this update frequency. WebSockets would add complexity without benefit here."

The ability to **choose the simpler tool deliberately** is a Staff-level signal.

---

## Failure Modes

### 1. Missing heartbeats → silent connection death

**What happens**: Mobile device loses signal briefly. TCP connection doesn't close (OS buffers the FIN). Server thinks client is still connected. Client thinks server is still reachable. Messages sent to the server are never delivered to the client.

**Detection**: Track `last_ping_at` per connection. Alert if a connection has no heartbeat for >60 seconds.

**Fix**: Application-level heartbeat every 20 seconds. If client misses 3 heartbeats: server closes the connection and removes from registry.

### 2. Fan-out overwhelming downstream

**What happens**: A celebrity sends a tweet. Your fan-out worker tries to write to 30M inboxes synchronously. The inbox write service is overwhelmed. Other users' writes queue behind the celebrity.

**Fix**: Async fan-out via queue (Kafka). Fan-out workers scale independently. Apply backpressure at the queue. For celebrities: hybrid model (fan-out on read).

### 3. Message ordering violation on reconnect

**What happens**: User receives messages [1, 2, 4, 5]. Message 3 is delayed. Client displays gap. 30 seconds later, message 3 arrives — now displayed out of order.

**Fix**: Client buffers and holds messages until the expected next sequence number arrives (with a timeout to give up after 5 seconds and display a "message may be missing" indicator).

### 4. Redis pub/sub memory explosion

**What happens**: You use Redis Pub/Sub to fan out messages to WebSocket servers. A chatty channel has 50K subscribers. Each message is replicated to all 50K subscribers in memory before delivery. Redis memory spikes.

**Fix**: Kafka for high-fan-out scenarios. Redis Pub/Sub is appropriate for low-subscriber-count channels. For 50K subscribers: Kafka with consumer groups is more appropriate.

### 5. Sticky session imbalance

**What happens**: You route WebSocket connections to servers by user_id hash. One server gets popular users who are highly active. That server is CPU-saturated while others idle.

**Fix**: Route by connection_id (random assignment), not user_id. Use the service registry to route messages to the right server regardless. Connection distribution = random. Message routing = lookup.

---

## Staff-Level Thinking

The real-time updates pattern looks like a technology choice (WebSockets vs SSE) but is actually an **architecture** choice.

The questions a staff engineer asks before choosing:

**On the data flow**: "Is this bidirectional or server-to-client? What's the update frequency? What's the payload size?"

**On the scale**: "How many concurrent connections? What's the fan-out ratio? Are there celebrity/hot users?"

**On the reliability**: "What's the offline delivery requirement? What's the ordering guarantee? What happens when a server dies?"

**On the operational cost**: "What does 1M persistent connections cost? Is the connection tier separate from the compute tier?"

**The key insight**: At small scale, real-time = WebSockets + Redis Pub/Sub. At large scale, real-time = a **tiered architecture** where connection management, fan-out, delivery, and storage are separate layers that scale independently.

The interviewer is not testing whether you know what a WebSocket is. They are testing whether you can reason about a system where each of those layers fails independently and must be designed to degrade gracefully.

---

---

## Evolution Path

```text
Stage 1: Short Polling (0–10K users)
  Every client polls every 5 seconds. Simple. Wasteful.
  Breaking point: server CPU dominated by "return empty" responses.
  Signal: empty-response ratio > 95%; server CPU > 50% from polling alone.

Stage 2: Long Polling (10K–100K users)
  Server holds connection until event fires. Near-real-time.
  Breaking point: thread-per-connection model exhausted.
  Signal: thread count at max; request queue depth growing.

Stage 3: SSE or WebSocket, Single Server (100K–500K users)
  Async server (Netty/WebFlux). One event loop handles thousands of connections.
  Breaking point: single server RAM for connections saturated.
  Signal: connection count > 50K/server; memory > 80%.

Stage 4: Multiple Servers + Redis Pub/Sub (500K–5M users)
  Connection tier scales horizontally. Redis routes cross-server messages.
  Breaking point: Redis CPU/memory under high fan-out load.
  Signal: Redis CPU > 70%; pub/sub channel memory growing.

Stage 5: Kafka Fan-Out + Dedicated Connection Gateway (5M–50M users)
  Separate connection management (stateful) from message processing (stateless).
  Kafka provides durable, ordered, replayable fan-out.
  Breaking point: infrastructure cost per connection becomes dominant.
  Signal: monthly cost per active connection > $0.001.

Stage 6: Tiered Connectivity (50M+ users — WhatsApp scale)
  Inactive mobile users: push notifications (APNs/FCM), not WebSocket.
  Active sessions only: WebSocket for foreground app.
  Geographic distribution: connection servers per region, nearest-region routing.
  Breaking point: this is the ceiling. Beyond it: domain partitioning.
```

**The 10x question answer**: "At 10x users we hit [current stage]'s breaking point — [specific signal metric]. The next stage is [next stage], which requires [specific architectural change] at a cost of [rough estimate]. We should begin planning that transition when [metric] reaches [threshold]."

---

## Migration Strategy

**From polling to WebSocket (zero-downtime)**

```text
Step 1: Deploy WebSocket endpoint alongside existing poll endpoint.
        Both serve the same data. No clients use WebSocket yet.

Step 2: Client-side feature flag: 1% of clients switch to WebSocket.
        Monitor: connection stability, message delivery rate vs polling baseline.

Step 3: Ramp: 1% → 10% → 50% → 100% over 2 weeks.
        Rollback trigger: WebSocket error rate > polling error rate.

Step 4: Deprecate polling endpoint (6-month notice).
        During deprecation: polling still works, WebSocket is default.

Step 5: Remove polling endpoint.

Rollback at any step: disable WebSocket flag; all clients revert to polling.
Rollback time: seconds (feature flag flip).
Data risk: none (stateless read path).
```

---

## Organisational Ownership

```text
Platform / Infrastructure team owns:
  - WebSocket connection tier (capacity planning, server fleet)
  - Kafka / Redis fan-out infrastructure (uptime SLA to product teams)
  - Connection registry (service discovery for cross-server routing)
  On-call: Platform SRE. Paged for: connection drop > 10%, fan-out lag > SLO.

Product teams own:
  - Message schemas and business logic
  - Which events trigger fan-out to which users
  - Client-side reconnect and replay logic
  On-call: product on-call. Paged for: their specific feature's message delivery rate.

Ownership boundary conflict (most common):
  "Messages not delivering" — is it the connection tier (platform) or the event publisher (product)?
  Resolution: connection tier exposes per-producer delivery metrics.
  If delivery confirmed at connection tier → product team's client issue.
  If not delivered at connection tier → platform team's issue.
```

---

## Incident Response

**Symptom**: "Messages not delivering" / connection drop alert

```text
Diagnose (first 5 minutes):
  1. Dashboard: websocket.connections.active — dropped? (connection tier issue)
  2. Dashboard: kafka.consumer.lag[fan-out-group] — growing? (fan-out bottleneck)
  3. Dashboard: redis.connected_clients — at limit? (Redis exhaustion)
  4. kubectl get pods -n realtime — any CrashLoopBackOff?

Likely cause → mitigation:

  A. WebSocket server pod crashed
     → Kubernetes auto-restarts. Users reconnect in ~30s.
     → Missed messages: delivered on reconnect via offline delivery.
     → Action: verify pod restarted; check for OOM kill (increase memory limit).

  B. Kafka consumer lag growing
     → Fan-out workers overwhelmed.
     → Immediate: kubectl scale deployment/fanout-workers --replicas=+4
     → Watch: lag should start decreasing within 2 minutes.

  C. Redis Pub/Sub OOM
     → Immediate: redis-cli CONFIG SET maxmemory-policy allkeys-lru
     → If already evicting: restart Redis (lose in-flight messages;
       users get missed messages on reconnect via offline delivery).
     → Permanent: migrate to Kafka fan-out; Redis only for connection registry.

  D. Celebrity/viral event fan-out spike
     → Identify: which user/event caused the spike?
     → Immediate: circuit-break that user's fan-out (stop enqueuing their events).
     → Let queue drain. Re-enable with rate limiting.

Recovery validation:
  websocket.connections.active returns to baseline (±5%) for 10 minutes.
  kafka.consumer.lag returns to < 1000 for 10 minutes.
```

---

## Design Review Checklist

```text
□ Protocol justified? WebSocket chosen over SSE because client also sends? Documented?
□ Heartbeat interval < proxy idle timeout (typically 60–90s)? Set to 25s?
□ Auto-reconnect implemented? With exponential backoff and jitter?
□ JWT validated on WebSocket handshake? What happens when token expires mid-session?
□ Fan-out ratio calculated? Celebrity/hot-user case handled?
□ Offline delivery implemented? Messages queued and replayed on reconnect?
□ Replay is idempotent? (safe to deliver same message twice)
□ Connection memory budget calculated? (connections × 13KB per connection)
□ Outbound bandwidth calculated at peak fan-out? Fits within NIC capacity?
□ Alert: connection drop > 10% in 5 min?
□ Alert: fan-out lag growing for > 2 min?
□ Alert: p99 delivery latency > SLO?
□ Connection tier team identified? On-call rotation documented?
```

---

## Threat Model

```text
T1: Connection Hijacking (Spoofing)
  Attack: intercept WebSocket upgrade; substitute attacker's token.
  Mitigation: TLS mandatory; JWT in Authorization header (not URL);
              15-minute access token TTL; validate on every message for high-security channels.

T2: Message Injection (Tampering)
  Attack: authenticated user sends message as a different user.
  Mitigation: server sets sender_id from JWT claims; never from message body.
              Client cannot forge the sender_id field — server is the authority.

T3: Connection Flood (DoS)
  Attack: 100K connections from a botnet exhausting server memory.
  Mitigation: max N connections per IP per minute (rate limit at LB);
              max M connections per authenticated user_id;
              CDN/WAF DDoS protection (Cloudflare, AWS Shield) upstream.

T4: Presence Stalking (Information Disclosure)
  Attack: query presence API to track when specific individuals are online.
  Mitigation: presence visible only to mutual connections;
              "last seen" shows bucketed time (not exact);
              presence queries rate-limited; users can disable presence.

T5: Replay Attack (Tampering)
  Attack: capture a valid signed message; replay it later.
  Mitigation: each message has UUID (message_id); server deduplicates within 5-minute window;
              server rejects messages with timestamp older than 5 minutes.
```

---

*End of Pattern 1: Real-Time Updates*
# Pattern 2: Dealing With Contention

> "Two threads walk into a bar and try to order the last beer simultaneously. This is your entire distributed systems career."

---

## The Shape

Multiple concurrent actors want to read and modify the same piece of state. Only one outcome is correct. Most of the time the correct outcome is obvious. Under load, with network failures, with retries, and with distributed systems, getting that correct outcome consistently is the central engineering challenge.

---

## When You See This Pattern

- **Ticket/seat booking**: Two users buy the last ticket simultaneously
- **Inventory management**: Flash sale; concurrent deductions from stock count
- **Payment processing**: Concurrent debits from the same account
- **Rate limiting**: Concurrent requests incrementing the same counter
- **Distributed locks**: Exactly one worker should run a job at a time
- **Leader election**: Exactly one node should be primary
- **Coupon redemption**: A coupon can only be used once

The signal: **there is a finite resource and multiple actors want to claim or modify it simultaneously, and allowing both to proceed would produce an incorrect result**.

---

## The Core Tension

```text
More concurrency ←———————————————→ Stronger correctness guarantee
(higher throughput)                  (no double-bookings, no oversell)

Fewer locks ←————————————————————→ More locks
(faster, simpler)                    (safer, slower)

Optimistic ←—————————————————————→ Pessimistic
(assume no conflict;                 (assume conflict;
 retry on collision;                  block until safe;
 great when conflicts are rare)       great when conflicts are frequent)
```

---

## Building Blocks

### Single-Node: Atomic Operations

Before introducing distributed complexity, check if a single-node atomic operation solves the problem.

```text
"Decrement ticket count and fail if it hits zero"

Redis: DECR tickets:event-123
  Returns -1? → somebody already got the last one. Fail.
  Returns 0?  → you got the last one. Continue.
  Returns N>0? → more remain. Continue.

Atomicity guaranteed by Redis's single-threaded execution model.
No lock needed. No transaction needed.
```

This pattern handles: rate limiting counters, stock decrement for small inventories, like counts, view counts.

**The limit**: Redis is in-memory. Not durable by default. For financial operations (debit an account), you need database-level guarantees.

---

### Single-Node: Database Locks

**Pessimistic locking** (`SELECT FOR UPDATE`): Lock the row before reading it. No other transaction can read-for-update or write until you commit.

```sql
BEGIN;
  SELECT balance FROM accounts WHERE id = 123 FOR UPDATE;
  -- Row is now locked. Other transactions trying to SELECT FOR UPDATE on this row will WAIT.
  UPDATE accounts SET balance = balance - 100 WHERE id = 123 AND balance >= 100;
  INSERT INTO transactions (account_id, amount, type) VALUES (123, -100, 'DEBIT');
COMMIT;
-- Lock released.
```

**When it works well**: Conflicts are frequent (e.g., a popular seller's inventory). Lock time is short (milliseconds). Users expect to wait briefly.

**When it fails**: Lock held across a slow operation (HTTP call to fraud service). Other transactions pile up. Performance collapses. This is the most common production contention disaster.

```text
NEVER do this:

BEGIN;
  SELECT ... FOR UPDATE;           ← lock acquired
  result = httpClient.call(fraud_service);  ← SLOW EXTERNAL CALL WHILE LOCKED
  UPDATE ...;
COMMIT;                            ← lock released
```

---

**Optimistic locking** (version number): Read the row without locking. Apply your change only if the row hasn't changed since you read it (check version number). If it has changed: retry.

```sql
-- Read:
SELECT balance, version FROM accounts WHERE id = 123;
-- Returns: balance=1000, version=5

-- Update (only if version still matches):
UPDATE accounts 
SET balance = 900, version = 6 
WHERE id = 123 AND version = 5;  -- ← the guard

-- If 0 rows updated: somebody else changed the row. Retry.
-- If 1 row updated: success.
```

**When it works well**: Conflicts are rare (each user's account is typically only accessed by one active request at a time). Reading is cheap. Retry cost is acceptable.

**When it fails**: Under high contention on the same row (flash sale: 1000 concurrent requests on inventory:item-X), most transactions will conflict and retry. With 1000 retries all firing simultaneously, you get a **contention storm**: each retry arrives, collides, retries again. Throughput collapses.

---

### Isolation Levels

The isolation level of a database transaction determines what anomalies are possible. This is a contention problem at its core.

```text
READ UNCOMMITTED: Can read another transaction's uncommitted changes.
  Risk: Dirty reads. Rarely correct for financial data.

READ COMMITTED (PostgreSQL default): Only see committed data.
  Risk: Non-repeatable reads. Read same row twice, see different values (another txn committed between reads).

REPEATABLE READ: Rows read in a transaction won't change.
  Risk: Phantom reads. A range query (SELECT WHERE status='PENDING') might return different rows on re-execution.

SERIALIZABLE: Transactions behave as if executed one at a time.
  Risk: Serialization failures → retry overhead. Performance cost.
```

**The production rule**: Use READ COMMITTED for most operations. Use REPEATABLE READ or SERIALIZABLE when you must prevent read skew or write skew anomalies. Always document why you chose a higher isolation level — the performance cost must be justified.

**Write skew (the subtle one)**:
```text
Two doctors on call. Policy: at least one must always be on call.

Doctor A: reads "both doctors on call" → decides to go off call
Doctor B: reads "both doctors on call" → decides to go off call
Both commit.
Result: nobody on call. Policy violated.

Neither modified the other's row. No conflict detected by READ COMMITTED.
Requires SERIALIZABLE isolation (or explicit row-level locking) to prevent.
```

---

### Distributed Locks

When the resource spans multiple nodes (or you need a lock across processes), database row-level locks don't work. You need a distributed lock.

**Redis-based distributed lock**:
```text
Acquire:
  SET lock:resource-123 <unique-token> NX PX 5000
  (NX = only if not exists; PX 5000 = expire in 5 seconds)
  
  Returns OK → lock acquired
  Returns nil → lock held by someone else → retry/fail

Release (safe):
  Lua script (atomic):
  if GET lock:resource-123 == <my-unique-token>:
    DEL lock:resource-123
  
  Must check token before deleting. Otherwise you might delete another holder's lock
  if your lock expired but theirs is now held.
```

**The fencing token problem**:
```text
Process A acquires lock (token=5)
Process A pauses for 10 seconds (GC, network stall)
Lock TTL expires after 5 seconds
Process B acquires lock (token=6)
Process B modifies the resource
Process A wakes up — still thinks it has the lock
Process A modifies the resource — CORRUPTING IT

Fix: the storage layer (database) must know the current lock token.
     Only accept writes with the current token.
     Process A's write (token=5) is rejected because token=6 is current.
```

**Fencing token in practice**:
```sql
-- When acquiring the lock, record the token in the resource row:
UPDATE jobs SET status='PROCESSING', lock_token=5, worker='server-A' 
WHERE job_id=123 AND (lock_token IS NULL OR lock_token < 5);

-- When writing back:
UPDATE jobs SET status='COMPLETE', result='...'
WHERE job_id=123 AND lock_token=5;  -- ← guard: only if I still hold the lock
-- 0 rows updated? My token expired. Another worker took over. Abort.
```

---

### Consensus-Based Coordination

For the hardest contention problems — leader election, distributed lock at high reliability — you need consensus protocols (Raft, Paxos) via etcd, ZooKeeper, or Consul.

These are covered in depth in the handbook (Chapter 47). The key point for this pattern:

```text
Use Redis/database locks when:
  - A brief window of double-lock is acceptable
  - The protected operation is idempotent (double-execution is harmless)
  - Performance is critical

Use etcd/ZooKeeper consensus locks when:
  - Exactly-once execution is required
  - Split-brain cannot be tolerated
  - Database primary election, Kubernetes leader election
```

---

## Canonical Problem 1: Ticket Booking (Last Seat)

### Requirements
- 100K concurrent users trying to book during a popular concert sale
- A seat can only be booked once
- Timeout: respond within 500ms

### Why This Is Hard

At the moment of sale, 10,000 users simultaneously try to book seat 14C. Your naïve implementation:

```python
# WRONG
seat = db.query("SELECT * FROM seats WHERE id='14C'")
if seat.status == 'AVAILABLE':
    db.execute("UPDATE seats SET status='BOOKED' WHERE id='14C'")
    return "Booked!"
```

This is a **check-then-act** race condition. Between the check and the act, 10,000 other requests also check, all see 'AVAILABLE', and all proceed to book. 10,000 bookings for one seat.

### Solution 1: Pessimistic Lock (Correct, Slower)

```sql
BEGIN;
  SELECT * FROM seats WHERE id = '14C' FOR UPDATE;
  -- All other transactions wait here
  
  IF status = 'AVAILABLE':
    UPDATE seats SET status = 'BOOKED', user_id = :user_id, booked_at = NOW()
    WHERE id = '14C';
    INSERT INTO bookings (seat_id, user_id, event_id) VALUES ('14C', :user_id, :event_id);
  COMMIT; -- returns 'BOOKED' or 'UNAVAILABLE'

  ELSE:
    ROLLBACK; -- returns 'UNAVAILABLE'
```

Under 10,000 concurrent requests, these queue up and execute serially. Throughput: however fast the database can process commits. For seat booking where concurrency is expected to be <100 for any single seat, this is fine.

### Solution 2: Atomic Compare-and-Swap (Faster)

```sql
-- No lock needed; atomicity comes from the WHERE clause
UPDATE seats 
SET status = 'BOOKED', user_id = :user_id, booked_at = NOW()
WHERE id = '14C' AND status = 'AVAILABLE';

-- If 1 row updated: you got it
-- If 0 rows updated: somebody else got it first
```

No `SELECT FOR UPDATE`. No serialised queue. The database processes each UPDATE atomically. At most one UPDATE wins (only one can match the `AND status = 'AVAILABLE'` condition).

This is faster and simpler for seat booking because:
- The window of conflict is tiny (microseconds)
- The atomic update is handled by the database's row-level locking internally
- No explicit transaction needed for the check-and-update

### Solution 3: Redis for High-Throughput Pre-Check

At 100K concurrent users, even atomic database updates become a bottleneck (PostgreSQL handles ~10K-50K writes/sec). Add a Redis pre-check:

```text
Architecture:

1. Pre-reservation (Redis, fast):
   SET seat:14C:holder user-456 NX PX 120000  (2-minute hold)
   → Returns OK: user-456 has a 2-minute window to complete payment
   → Returns nil: seat already held, return "seat unavailable"

2. Payment (your payment flow, up to 2 minutes)

3. Confirm (database, durable):
   BEGIN;
     UPDATE seats SET status='BOOKED', user_id=user-456 WHERE id='14C' AND status='AVAILABLE';
     INSERT INTO bookings ...;
   COMMIT;
   DEL seat:14C:holder  -- release Redis hold
   
4. If payment fails or times out:
   DEL seat:14C:holder  -- release hold; seat becomes available again
   TTL on Redis key also releases it automatically after 2 minutes
```

**Why this works**: Redis handles the high-concurrency "race to claim" (step 1). The database only sees confirmed, serialised bookings (step 3). Redis throughput: 100K+ ops/sec. Database: only sees ~1K committed bookings/sec.

---

## Canonical Problem 2: Payment Account Debit

### Requirements
- Debit $100 from account A, credit $100 to account B
- Account A must not go below $0 (overdraft prevention)
- Exactly once (no duplicate debits from retries)

### The Double Debit Problem

```text
Client sends: POST /transfer { from: A, to: B, amount: 100 }
Server processes: begins debit of A
Network timeout: client doesn't get response
Client retries: POST /transfer { from: A, to: B, amount: 100 }
Server processes: ANOTHER debit of A

A is now debited $200. B is credited $200. Disaster.
```

### Solution: Idempotency Key + Optimistic Lock

```sql
-- Client generates idempotency_key = UUID once, before the first request
-- Sends same key on retry

-- Step 1: Check if already processed (idempotency)
SELECT * FROM transfers WHERE idempotency_key = :key;
-- If found: return the original result (no reprocessing)

-- Step 2: Atomic debit with balance guard (optimistic)
BEGIN;
  INSERT INTO transfers (idempotency_key, from_account, to_account, amount, status)
  VALUES (:key, :from, :to, :amount, 'PENDING')
  ON CONFLICT (idempotency_key) DO NOTHING;  -- second request: skip
  
  UPDATE accounts 
  SET balance = balance - :amount, version = version + 1
  WHERE id = :from_account 
    AND balance >= :amount;        -- ← balance guard: prevents overdraft
  -- 0 rows: insufficient funds → ROLLBACK
  
  UPDATE accounts SET balance = balance + :amount WHERE id = :to_account;
  UPDATE transfers SET status = 'COMPLETE' WHERE idempotency_key = :key;
COMMIT;
```

**The lock ordering for transfers**: Always lock accounts in the same order (by account ID ascending) to prevent deadlocks when two transfers involve the same two accounts.

```sql
-- Transfer A→B and concurrent Transfer B→A

-- WRONG (deadlock possible):
-- T1: LOCK A, then LOCK B
-- T2: LOCK B, then LOCK A

-- RIGHT (always lock by account ID order):
accounts_to_lock = sorted([from_account, to_account])
for account in accounts_to_lock:
    SELECT ... FROM accounts WHERE id = account FOR UPDATE;
```

---

## Canonical Problem 3: Inventory Management (Flash Sale)

### Requirements
- 1,000 items available
- 100,000 concurrent requests during sale launch
- No oversell
- Respond within 200ms

### Why Database Locks Fail Here

At 100K concurrent requests, `SELECT FOR UPDATE` on the inventory row means 100K transactions queueing behind a single lock. Each holds the lock for ~5ms. Throughput: 200 orders/sec. 100K users wait an average of 250 seconds. Unusable.

### Solution: Pre-Allocation Pattern

```text
Step 1 (before sale opens): Pre-populate Redis with 1,000 slots.
  for i in range(1000):
      RPUSH inventory:item-X "slot-{i}"
  
  List has 1,000 items. Each item = one unit of inventory.

Step 2 (during sale): Atomic pop.
  slot = LPOP inventory:item-X
  → Returns "slot-42" → you have a unit reserved. Proceed to payment.
  → Returns nil → inventory exhausted. Return "sold out."

  LPOP is atomic. Only one caller gets each slot.
  No database lock needed. Redis handles 100K+ ops/sec.

Step 3 (payment): Deduct from database (durable record).
  BEGIN;
    INSERT INTO orders (user_id, item_id, slot_id, status) VALUES (..., 'PENDING');
    UPDATE inventory SET remaining = remaining - 1 WHERE item_id = 'X' AND remaining > 0;
  COMMIT;

Step 4 (payment failed / timeout): Return slot to Redis.
  RPUSH inventory:item-X "slot-42"
  UPDATE orders SET status = 'CANCELLED' WHERE slot_id = 'slot-42';
```

**Why this works**: The contention bottleneck (claiming a slot) is moved from the database to Redis. Redis's single-threaded atomic operations handle 100K+ ops/sec without locking. The database only sees committed, serialised inventory changes.

**The failure mode**: Redis restarts and loses the pre-allocated list before the sale is over. Fix: persist the slot allocation to DB when order is confirmed, not when slot is claimed. The Redis list is a performance cache, not the source of truth.

---

## Deep Dive 1: Deadlocks

### What They Are

```text
Transaction T1:              Transaction T2:
  LOCK account-A               LOCK account-B
  LOCK account-B    ←WAIT→    LOCK account-A

T1 waits for T2 to release B.
T2 waits for T1 to release A.
Neither can proceed. Forever.
```

PostgreSQL detects deadlocks and kills one transaction (with a `deadlock detected` error). The application must retry.

### Prevention

**Lock ordering**: Always acquire locks in canonical order (e.g., by ID ascending). Both T1 and T2 would try to lock A first, then B. T2 blocks on A until T1 completes. No circular wait.

**Lock scope minimisation**: Hold locks for the absolute minimum time. Don't make external calls, file I/O, or HTTP requests while holding a database lock.

**Timeout**: Set a lock timeout (PostgreSQL: `SET lock_timeout = '500ms'`). If you can't acquire the lock in 500ms, fail fast and retry rather than waiting indefinitely.

### Detection in Production

```sql
-- Find waiting transactions
SELECT pid, query, state, wait_event_type, wait_event, query_start
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- Check PostgreSQL logs for:
-- ERROR: deadlock detected
-- DETAIL: Process 1234 waits for ShareLock on transaction 5678
```

---

## Deep Dive 2: The ABA Problem

### What It Is

Optimistic locking with a version number can be defeated if the version cycles:

```text
Thread A reads: value=10, version=5
Thread B writes: value=20, version=6
Thread C writes: value=10, version=5  (cycles back to same value AND version!)
Thread A checks: value=10, version=5 → matches! Proceeds. But state is not what A saw.
```

In practice: version numbers are monotonically increasing (never decrease), so the ABA problem doesn't occur with standard optimistic locking. The ABA problem primarily affects lock-free CAS (compare-and-swap) operations at the hardware level or when using timestamps (which can repeat) instead of sequence numbers.

**The production lesson**: Use monotonically increasing version integers, not timestamps. Timestamps can repeat due to clock adjustment; version integers cannot.

---

## Deep Dive 3: Hot Rows

### What They Are

A single database row receives so many concurrent writes that all transactions queue behind each other. The row becomes the throughput bottleneck.

**Classic example**: An inventory counter row during a flash sale. All decrements serialise on one row.

### Solutions

**Partition the counter**: Instead of one counter, maintain N counters. Randomly pick one to decrement. Total inventory = sum of all N counters.

```sql
-- 10 counter shards for inventory
counters table: (item_id, shard_id, count)

-- Decrement:
UPDATE counters 
SET count = count - 1 
WHERE item_id = 'X' AND shard_id = FLOOR(RANDOM() * 10) AND count > 0;

-- Total inventory:
SELECT SUM(count) FROM counters WHERE item_id = 'X';
```

This distributes write contention across 10 rows instead of 1. 10× throughput improvement.

**Redis for high-throughput counters**: Move the counter to Redis (DECR), use the database only for durable committed orders. Covered in the Canonical Problem 3 above.

**Async aggregation**: Don't update a single counter on every operation. Write individual events to an event log. Periodically aggregate. Accept eventual consistency of the count.

---

## Interview Calibration

### What a Senior Engineer Answers

- "Use a transaction with SELECT FOR UPDATE"
- Describes ACID properties
- Mentions optimistic vs pessimistic locking
- Handles the basic race condition

### What a Staff Engineer Answers

- Starts with: "What's the conflict rate? How often will two requests actually collide?"
- Chooses optimistic vs pessimistic based on conflict rate, not habit
- Recognises that the real problem at scale is throughput, not just correctness
- Proposes the Redis pre-allocation pattern for the flash sale
- Handles the retry/idempotency interaction: "every retry must be idempotent"
- Identifies the hot row problem and proposes counter sharding
- Recognises that distributed locks require fencing tokens to be safe

### The Question That Separates Levels

> "How do you prevent an account from being overdrafted when 10,000 concurrent requests are trying to debit it?"

**Senior**: "SELECT FOR UPDATE on the account row, check balance before debit."

**Staff**: "The row lock is correct for correctness. But at 10,000 concurrent requests, we have a performance problem — all serialise on one row. I'd use optimistic locking (balance >= amount guard in the UPDATE WHERE clause) rather than SELECT FOR UPDATE. This avoids the lock acquisition overhead while still preventing overdraft. The UPDATE itself is atomic. If 0 rows updated, funds are insufficient. For retries, the idempotency key ensures we don't double-debit on client retry."

---

## Failure Modes

### 1. Lock held across slow operations

```text
BEGIN;
  SELECT ... FOR UPDATE;           ← row locked
  response = fraud_service.check(); ← 2-second HTTP call
  UPDATE ...;
COMMIT;                             ← row unlocked after 2 seconds

All other transactions wait 2 seconds for this row.
At 100 concurrent users: 200 seconds of cumulative wait per second of fraud latency.
```

**Fix**: Never make external calls while holding a database lock. Do the fraud check before the transaction; only enter the transaction for the atomic update.

### 2. Retry amplification on contention

```text
10,000 concurrent requests. All collide. All retry.
Retry adds exponential backoff, but all started at the same time.
After 100ms backoff: all 10,000 retry simultaneously.
Second collision. All retry again.
Thundering herd.
```

**Fix**: Retry with full jitter (`sleep(random(0, base_backoff * 2^attempt))`). Retries spread out over time. Contention disperses.

### 3. Redis lock without fencing token

```text
Process A acquires Redis lock (TTL=5s). Token=uuid-A.
Process A pauses (GC) for 7 seconds.
Lock expires. Process B acquires lock. Token=uuid-B.
Process B modifies resource.
Process A wakes up. Checks Redis: key is absent (expired). Thinks lock is... gone?
No check: proceeds to modify resource. Double write.
```

**Fix**: Always check the fencing token against the resource, not just whether the Redis key exists.

### 4. Oversell due to pre-check/act gap

```text
Available: 1 unit
Request 1: checks → 1 available → delays (network/processing)
Request 2: checks → 1 available → commits → 0 units
Request 1: acts → commits → -1 units (oversold)
```

**Fix**: Use atomic compare-and-swap in the database. The guard `WHERE count > 0` in the UPDATE prevents the negative write.

---

## Staff-Level Thinking

Contention is a **throughput problem masquerading as a correctness problem**. The correctness part is easy — lock the row, check before acting. The throughput part is where staff-level thinking is required.

The key questions:

**"What is the actual collision rate?"** If you're designing a flash sale inventory system but the expected concurrent requests per item is 100K, you need Redis-based pre-allocation. If it's 100, database locks are fine.

**"What is the cost of a conflict?"** For a financial debit: correctness is paramount, retry is acceptable. For a like count: approximate is fine, lock-free is correct choice.

**"Can I make this idempotent?"** Idempotent operations can be retried freely. Non-idempotent operations (charging a credit card) require idempotency keys at the application layer.

**"Where is the hot row?"** Any time you have a global counter, a shared queue, or a single-resource claim, you have a potential hot row. Identify it early and pre-plan the sharding or pre-allocation strategy.

The staff engineer's insight: **the right answer to a contention problem is usually to eliminate the contention** — by restructuring the data model so that concurrent operations don't touch the same row. This is harder than adding locks, but it is the answer that scales.

---

---

## Evolution Path

```text
Stage 1: Optimistic Locking, Single DB (0–1K writes/sec on contested rows)
  Atomic WHERE clause guard. Fast. Simple. No infrastructure.
  Breaking point: collision rate > 20% (high-contention rows).
  Signal: retry rate > 20%; write latency p99 > 200ms.

Stage 2: Pessimistic Locking (1K–5K writes/sec, high collision rate)
  SELECT FOR UPDATE. Serialises access. Correct under high contention.
  Breaking point: lock held across slow operations; lock wait time dominates latency.
  Signal: pg_stat_activity shows many Lock-waiting queries; p99 latency > 500ms.

Stage 3: Redis Pre-Allocation (5K–100K claims/sec)
  LPOP from a pre-filled list. Atomic. 100K ops/sec. No DB lock.
  Breaking point: Redis is a single point of failure for the claim step.
  Signal: Redis unavailability causes 100% claim failure (no fallback).

Stage 4: Counter Sharding (100K+ writes/sec to one counter)
  Distribute one counter across N shards. Write to random shard.
  Read = SUM across shards (eventual, but fast).
  Breaking point: cross-shard aggregation latency for reads.
  Signal: SUM query latency > read SLO.

Stage 5: Distributed Consensus Lock (leader election, critical coordination)
  etcd or ZooKeeper with fencing tokens. Exactly-once guarantee.
  Use only when the cost of double-execution is catastrophic.
  Breaking point: this handles essentially any scale. Cost is write latency (~5ms/op).
```

---

## Migration Strategy

**From SELECT FOR UPDATE to Redis Pre-Allocation (flash sale pattern)**

```text
Step 1: Parallel deploy Redis pre-allocation alongside existing DB lock logic.
        Feature flag: 0% traffic uses Redis path.

Step 2: Shadow mode — Redis pre-allocation runs on all requests but result is ignored.
        Validate: Redis list stays consistent with DB inventory count.

Step 3: Enable for 1% of traffic. Compare: DB path success rate vs Redis path success rate.

Step 4: Ramp to 100% over 3 days.

Step 5: Remove DB lock path after 7-day stable operation.

Rollback: flip feature flag. DB lock path resumes immediately.
Data risk: Redis list and DB count can diverge during dual-run.
          Reconciliation job runs every 5 minutes: if diverged, repopulate Redis from DB.
```

---

## Organisational Ownership

```text
Platform / DB team owns:
  - PostgreSQL configuration, connection pooling (PgBouncer), index advice.
  - Redis cluster for pre-allocation and distributed locks.
  On-call: paged for DB lock contention alerts, Redis failure.

Application team owns:
  - Lock strategy choice (optimistic vs pessimistic vs Redis).
  - Idempotency key generation and handling.
  - Retry logic with jitter.
  On-call: paged for high retry rate, collision storm, payment failure rate.

Ownership boundary conflict:
  "High DB lock wait time" — DB config (platform) or query pattern (application)?
  Resolution: platform team provides lock wait time per query fingerprint.
  If one query is 90% of lock waits → application team's query.
  If distributed across queries → DB configuration (platform team).
```

---

## Incident Response

**Symptom**: Payment failure rate spike / "inventory shows available but purchase fails"

```text
Diagnose (first 5 minutes):
  1. SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock';
     → If > 50: lock contention incident.
  2. SELECT pid, query, wait_event FROM pg_stat_activity WHERE state='active'
     ORDER BY query_start LIMIT 20;
     → Identify which query holds the lock.
  3. redis-cli LLEN inventory:{item_id}
     → If 0 or negative: Redis list depleted (oversell risk).

Likely cause → mitigation:

  A. Lock contention storm (many transactions waiting for one row)
     → Immediate: identify and kill the blocking query:
       SELECT pg_terminate_backend(pid) FROM pg_stat_activity
       WHERE wait_event_type = 'Lock' AND query_start < NOW() - INTERVAL '30s';
     → Check: is any transaction holding lock across an external call (fraud API)?
     → Permanent: remove external calls from within transactions.

  B. Redis pre-allocation list empty (flash sale oversold)
     → Immediate: return "sold out" to all new requests (fail closed).
     → Reconcile: SELECT remaining FROM inventory WHERE item_id=?
       → RPUSH inventory:{item_id} slot-{i} for i in range(remaining)
     → Investigate: was list populated before sale started?

  C. Deadlock storm
     → grep "deadlock detected" /var/log/postgresql/postgresql*.log | wc -l
     → If > 10/minute: lock ordering violation.
     → Immediate: identify the two queries forming the cycle; add explicit ordering.

Recovery: payment success rate returns to baseline for 5 minutes.
```

---

## Design Review Checklist

```text
□ Collision rate estimated? Justifies optimistic vs pessimistic choice?
□ No external calls (HTTP, file I/O) inside a database transaction holding a lock?
□ Locks acquired in consistent canonical order (by ID ascending) to prevent deadlocks?
□ Retry logic uses exponential backoff with full jitter?
□ Idempotency key generated by client (not server)?
□ Idempotency key scoped to (user_id, key) pair — not just key alone?
□ What is maximum lock hold time? Is it documented and enforced?
□ For flash sales: Redis pre-allocation tested at expected peak QPS?
□ Fencing token implemented for distributed locks?
□ Alert: lock_wait_time p99 > 50ms?
□ Alert: collision/retry rate > 10%?
□ Alert: deadlock rate > 1/minute?
□ Hot row identified? Mitigation plan if it becomes a bottleneck?
```

---

## Threat Model

```text
T1: TOCTOU Race (Tampering / Elevation of Privilege)
  Attack: exploit the gap between balance check and debit.
  Mitigation: atomic compare-and-swap (WHERE balance >= amount in UPDATE);
              never: SELECT then UPDATE as two separate statements.

T2: Lock Starvation (DoS)
  Attack: acquire distributed lock; perform infinite loop; block all others.
  Mitigation: all distributed locks MUST have TTL; no indefinite locks permitted;
              alert if lock held > 2× expected duration.

T3: Idempotency Key Reuse (Tampering)
  Attack: replay a previous request using a stolen idempotency key.
  Mitigation: scope keys to (user_id, key); key from user A cannot be reused by user B;
              keys expire after 24 hours; revoke on session compromise.

T4: Negative Balance Exploit (Elevation of Privilege)
  Attack: concurrent debits exceed balance; account goes negative.
  Mitigation: WHERE balance >= :amount guard in UPDATE (atomic);
              DB-level CHECK constraint: balance >= 0 (defence in depth);
              test with concurrent load at 100× expected peak.
```

---

*End of Pattern 2: Dealing With Contention*
# Pattern 3: Multi-Step Processes

> "A payment that is half-processed is worse than a payment that never started."

---

## The Shape

A business operation requires multiple steps, each of which can fail independently. All steps must either complete together (consistency) or be cleanly undoable (compensation). The steps may span multiple services, multiple databases, or take minutes to hours to complete.

---

## When You See This Pattern

- Payment processing (debit → fraud check → reserve → settle → notify)
- Order fulfilment (place → inventory reserve → payment → ship → notify)
- IAM enrollment (create identity → set credentials → assign roles → provision access → send welcome)
- Loan origination (application → credit check → underwriting → approval → disburse → notify)
- Video processing (upload → transcode → thumbnail → index → publish)

The signal: **the operation has named steps that must all succeed or all be compensated, and some steps cannot be trivially rolled back**.

---

## The Core Tension

```text
Atomicity ←————————————————————————→ Availability
(all steps succeed or none persist)   (each step available independently)

Long transactions ←—————————————————→ Short transactions
(simple to reason about)               (scales horizontally)

Orchestration ←—————————————————————→ Choreography
(central coordinator; easier debug)    (decoupled services; harder debug)
```

---

## Building Blocks

### State Machines

Every multi-step process is a state machine. Making the state machine explicit forces you to answer the questions that production will ask:

```text
Payment State Machine:

CREATED → FRAUD_CHECK → AUTHORISED → CAPTURED → SETTLED
    ↓           ↓            ↓           ↓
CANCELLED   DECLINED     CANCELLED   DISPUTED → REFUNDED

Every arrow is a transition. Every transition can fail.
What state does the payment land in if CAPTURED → SETTLED fails?
Answer that before you write a line of code.
```

```sql
CREATE TYPE payment_status AS ENUM (
  'CREATED', 'FRAUD_CHECK', 'AUTHORISED', 
  'CAPTURED', 'SETTLED', 'DECLINED', 'CANCELLED', 'REFUNDED', 'DISPUTED'
);

ALTER TABLE payments ADD COLUMN status payment_status NOT NULL DEFAULT 'CREATED';

-- Valid transitions only (enforced in application code or DB trigger):
VALID_TRANSITIONS = {
  'CREATED':      ['FRAUD_CHECK', 'CANCELLED'],
  'FRAUD_CHECK':  ['AUTHORISED', 'DECLINED'],
  'AUTHORISED':   ['CAPTURED', 'CANCELLED'],
  'CAPTURED':     ['SETTLED', 'DISPUTED'],
  'SETTLED':      ['REFUNDED', 'DISPUTED'],
}
```

---

### The Saga Pattern

A Saga is a sequence of local transactions. Each transaction completes and emits an event (or sends a command). If a step fails, compensating transactions undo the preceding steps.

```text
Order Saga (Orchestration):

SAGA: PlaceOrder
  Step 1: OrderService.createOrder()          → compensate: OrderService.cancelOrder()
  Step 2: InventoryService.reserveItems()     → compensate: InventoryService.releaseItems()
  Step 3: PaymentService.chargePayment()      → compensate: PaymentService.refundPayment()
  Step 4: ShippingService.createShipment()    → compensate: ShippingService.cancelShipment()
  Step 5: NotificationService.sendConfirmation() → no compensation (emails can't be unsent)

Success: all steps complete. Order confirmed.
Failure at Step 3 (payment declined):
  Execute: InventoryService.releaseItems()  (compensate Step 2)
  Execute: OrderService.cancelOrder()       (compensate Step 1)
  Result: order cancelled, inventory returned, no charge.
```

**The irreversible step rule**: Place irreversible steps last (sending emails, external bank calls). Once you send an email, you can't unsend it. If that step is last and everything before it succeeded, you've already committed — the email is a consequence, not a risk.

---

### Orchestration vs Choreography

**Orchestration**: A central Saga Orchestrator sends commands to services and tracks state.

```text
                    ┌──────────────────────┐
                    │  Saga Orchestrator    │
                    │  (state: step 2 of 5)│
                    └──────┬───────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  OrderService      InventoryService    PaymentService
  "createOrder"     "reserveItems"      "chargePayment"
```

Pros: Easy to see the full flow. Easy to add retry/timeout logic. Easy to debug (one place has the state).
Cons: Orchestrator becomes a bottleneck. Orchestrator must be highly available.

**Choreography**: Services react to events. No central coordinator.

```text
OrderService: emits → OrderCreated
                            ↓
InventoryService: consumes OrderCreated → reserves items → emits InventoryReserved
                                                                    ↓
PaymentService: consumes InventoryReserved → charges → emits PaymentCharged
                                                                    ↓
ShippingService: consumes PaymentCharged → creates shipment → emits ShipmentCreated
```

Pros: Fully decoupled. Each service owns its own logic. Scales independently.
Cons: Hard to see the full flow. Hard to detect when a saga is stuck. Business logic scattered across services.

**When to choose which**:
- Complex flows with conditional logic, retries, timeouts → Orchestration
- Simple linear flows with independent services → Choreography
- Need to track "where is this order right now?" → Orchestration (single source of state)

---

### The Outbox Pattern (Reliable Event Publishing)

The hardest problem in a multi-step process: how do you write to a database AND publish an event atomically?

```text
WRONG:
  BEGIN;
    UPDATE orders SET status = 'PAID';
  COMMIT;
  publish(OrderPaid event);  ← if this fails, event never sent
                                → inventory never reserved
                                → order paid but never fulfilled

OUTBOX PATTERN:
  BEGIN;
    UPDATE orders SET status = 'PAID';
    INSERT INTO outbox (event_type, payload) VALUES ('OrderPaid', {...});
  COMMIT;
  ← both changes atomic in same transaction

  Background relay:
    SELECT * FROM outbox WHERE published = FALSE;
    → publish to Kafka
    → UPDATE outbox SET published = TRUE;
```

The event is guaranteed to exist if the database commit succeeds. The relay may publish it twice (at-least-once) but never zero times. Consumers must be idempotent.

---

### Exactly-Once Execution

True exactly-once is impossible in a distributed system. The practical target is **effectively exactly-once** = at-least-once delivery + idempotent processing.

```text
Step 2: InventoryService.reserveItems()

Called once: reserve items. reserved=TRUE, reservation_id=res-123.
Called twice (retry after network timeout):
  Check: SELECT * FROM reservations WHERE order_id = :order_id AND item_id = :item_id
  If exists: return the existing reservation (don't create a second one)
  If not: create new reservation

The inventory service is idempotent: calling it twice has the same effect as calling it once.
The reservation is deduplicated by (order_id, item_id).
```

---

## Canonical Problem 1: Design a Payment Processing Flow

### Requirements
- User initiates a payment (₹5,000 to a merchant)
- Must debit user's account exactly once
- Must credit merchant's account exactly once
- Must handle: fraud check failure, bank rejection, timeout, retry

### The State Machine

```text
INITIATED
    │
    ▼
FRAUD_CHECKED ──(declined)──→ DECLINED (terminal)
    │
    ▼
BANK_AUTHORISED ──(declined)──→ AUTH_FAILED (terminal)
    │
    ▼
DEBITED ──(error)──→ DEBIT_FAILED → [compensate: if partially applied]
    │
    ▼
MERCHANT_CREDITED
    │
    ▼
SETTLED (terminal)
```

### The Architecture

```text
API Layer:
  POST /payments → creates payment record (status=INITIATED), returns payment_id

Saga Orchestrator (separate service):
  Watches for INITIATED payments
  
  Step 1: FraudService.check(payment)
    → PASS: advance to FRAUD_CHECKED
    → FAIL: advance to DECLINED, stop

  Step 2: BankService.authorise(payment)
    → PASS: advance to BANK_AUTHORISED
    → FAIL: advance to AUTH_FAILED, stop

  Step 3: LedgerService.debitAccount(user, amount, idempotency_key=payment_id)
    → PASS: advance to DEBITED
    → FAIL: advance to DEBIT_FAILED, stop (account not debited; no compensation needed)

  Step 4: LedgerService.creditMerchant(merchant, amount, idempotency_key=payment_id+"-credit")
    → PASS: advance to MERCHANT_CREDITED
    → FAIL: 
        Retry up to 5 times (merchant credit can always be retried — idempotent)
        If still fails: advance to CREDIT_FAILED
                        COMPENSATE: LedgerService.refundDebit(user, amount, ...)
                        advance to REFUNDED

  Step 5: NotificationService.notifyBothParties(payment)
    → Fire and forget (no compensation; notification failure doesn't affect payment)
    advance to SETTLED
```

**The idempotency key design**: `payment_id` is used as the idempotency key for the debit. `payment_id + "-credit"` for the credit. These are derived deterministically — no matter how many times the orchestrator retries, the ledger service will not create duplicate entries.

### The Client Polling Problem

Payment flows are asynchronous. The client sends a POST, gets a `payment_id`, and then needs to know when it completes.

```text
Option A: Polling
  Client: GET /payments/:id every 2 seconds
  Server: returns current status
  Simple. Works. Wastes requests.

Option B: Webhook
  Merchant registers: POST /webhooks { url: "https://merchant.com/payment-result" }
  When payment completes: POST https://merchant.com/payment-result { payment_id, status }
  Reliable: retry webhook delivery until merchant acks.
  
Option C: SSE
  Client keeps SSE connection open
  Server pushes payment status changes
  Best UX; requires persistent connection management
```

For B2B payments: webhook. For consumer-facing apps: SSE or polling. For mobile: push notification.

---

## Canonical Problem 2: IAM Enrollment (Multi-Step with Compensation)

### Requirements
- Provision a new user: create identity → set credentials → assign roles → provision external systems → send welcome
- If any step fails before "provision external systems": clean rollback
- "Provision external systems" is partially irreversible (some systems don't support delete)

### The Saga Design

```text
SAGA: ProvisionUser

Step 1: IdentityService.createIdentity(user_data)
  Compensation: IdentityService.deleteIdentity(identity_id)
  Idempotency: ON CONFLICT (external_id) DO NOTHING; return existing

Step 2: CredentialService.setInitialPassword(identity_id, temp_password)
  Compensation: CredentialService.deleteCredentials(identity_id)
  Idempotency: check if credential already set for this identity

Step 3: AuthorisationService.assignDefaultRoles(identity_id, role_ids)
  Compensation: AuthorisationService.revokeAllRoles(identity_id)
  Idempotency: INSERT ... ON CONFLICT DO NOTHING

Step 4: ProvisioningService.provisionExternalSystems(identity_id)
  Compensation: ProvisioningService.attemptDeprovision(identity_id)
  Note: some external systems don't support delete → manual review queue if deprovisioning fails
  This is the "point of no return" — if successful, fully reversing is not guaranteed.

Step 5: NotificationService.sendWelcomeEmail(identity_id)
  No compensation (can't unsend email)
  Fire and forget; failure doesn't block enrollment

Failure handling:
  Steps 1–3 fail: clean compensation available
  Step 4 fails:   retry up to 3 times; if still fails: compensation + alert for manual review
  Step 5 fails:   log and retry later; don't roll back enrollment
```

---

## Deep Dive: Workflow Versioning

### The Problem

Your payment saga has been running for 6 months. A payment is in progress (step 3 of 5) when you deploy a new version of the orchestrator that changes the steps. The in-flight saga now has state that doesn't match the new code.

```text
Old saga (v1):  Step 1 → Step 2 → Step 3 → Step 4
New saga (v2):  Step 1 → Step 2 → Step 2b → Step 3 → Step 4

Payment P-123: currently at step 3 (v1 numbering)
New orchestrator: tries to resume at "step 3" — which is now different.
```

### Solutions

**Version tagging**: Each saga instance stores its schema version. The orchestrator handles each version separately.

```python
class SagaOrchestrator:
    def resume(self, saga_state):
        if saga_state.schema_version == 1:
            return SagaV1Handler.resume(saga_state)
        elif saga_state.schema_version == 2:
            return SagaV2Handler.resume(saga_state)
```

**Forward compatibility**: New code handles both old and new saga state. Old in-flight sagas complete on the old code path. New sagas use the new path.

**Drain before deploy**: For critical flows, drain all in-flight sagas to completion before deploying new orchestrator code. Only feasible for short-lived sagas.

---

## Interview Calibration

### What a Senior Engineer Answers

- "Use a transaction to wrap all the steps"
- Describes 2PC (two-phase commit) for distributed transactions
- Knows what a saga is

### What a Staff Engineer Answers

- "A single transaction doesn't work across services. I need a saga."
- Chooses orchestration vs choreography based on the flow complexity and debugging requirements
- Identifies which steps are irreversible and places them last
- Designs idempotency keys derived from the saga ID for all steps
- Addresses compensation logic explicitly: "What's the compensating action for each step?"
- Addresses the "point of no return": "Step 4 is irreversible. If it fails after partial completion, I need a manual review queue, not just retry."
- Considers workflow versioning for long-lived sagas

---

*End of Pattern 3: Multi-Step Processes*

---

## Evolution Path

```text
Stage 1: Single Database Transaction (0–100 ops/sec, same DB)
  All steps in one ACID transaction. Simple. Correct.
  Breaking point: steps span multiple services or take > 5 seconds.
  Signal: transaction timeouts; cross-service calls inside transactions.

Stage 2: Async Steps with DB Queue (100–1K ops/sec)
  Steps stored as rows; worker polls and executes next step.
  Breaking point: DB polling consumes > 20% of DB CPU; worker is a bottleneck.
  Signal: polling queries showing in slow query log; step execution lag growing.

Stage 3: Message Queue + Explicit State Machine (1K–10K ops/sec)
  Each step completion publishes event; worker executes next step.
  State machine stored in DB. Compensation logic defined per step.
  Breaking point: choreography becomes hard to debug; "why is workflow 123 stuck?"
  Signal: mean time to diagnose a stuck workflow > 30 minutes.

Stage 4: Dedicated Orchestrator — Temporal / Conductor (10K+ ops/sec)
  Central state machine with built-in retry, timeout, versioning, visibility.
  Breaking point: rarely hit. Temporal handles millions of workflows/day.
  Signal: orchestrator itself becomes a write bottleneck (rare; indicates DB needs sharding).

Stage 5: Hybrid — Orchestration + Choreography
  Complex business processes: orchestration (full visibility, retry control).
  Simple event reactions: choreography (decoupled, scalable).
  Platform provides both; team guidelines specify which pattern for which use case.
```

---

## Migration Strategy

**From synchronous call chain to Saga orchestration**

```text
Step 1: Define the state machine explicitly.
        Name every state. Document every valid transition.
        Before writing code — agree on this with the business.

Step 2: Add saga_state table to existing DB.
        Start persisting workflow state without changing execution logic.
        This is the audit trail foundation.

Step 3: Extract the slowest/most-fragile step first.
        If fraud check is slow: make it async first.
        Keep all other steps synchronous.
        Dual-run: compare fraud check results (sync vs async) for 1 week.

Step 4: Extract remaining steps one at a time.
        Each extraction: shadow mode → canary → 100% → remove old path.

Step 5: Add compensation logic.
        Compensation is often more complex than the happy path.
        Test every compensation path explicitly (chaos engineering).

Rollback at any step: re-enable synchronous execution via feature flag.
Data risk: in-flight sagas during rollback must complete on the old path.
          Drain in-flight sagas before rollback if possible.
```

---

## Organisational Ownership

```text
Platform team owns (if using Temporal/Conductor):
  - Workflow engine infrastructure and uptime SLA.
  - SDK published to internal package registry.
  - Monitoring: in-flight workflow count, step latency, stuck workflow detection.
  On-call: paged for orchestrator unavailability.

Domain teams own:
  - Workflow definitions (steps, compensation logic, timeout values).
  - Downstream service calls from within workflow steps.
  - Business rules embedded in step logic.
  On-call: paged for their workflow's stuck rate, step failure rate.

Shared responsibility:
  "Workflow X is stuck at step 3" — is step 3's downstream (domain) or the orchestrator (platform)?
  Resolution: orchestrator exposes per-step success/failure metrics with error classification.
  Downstream error → domain team. Orchestrator error → platform team.
```

---

## Incident Response

**Symptom**: "Payment stuck in processing" / saga_stuck_count alert fires

```text
Diagnose (first 5 minutes):
  1. SELECT id, current_step, updated_at FROM sagas
     WHERE status='IN_PROGRESS' AND updated_at < NOW() - INTERVAL '10 minutes'
     ORDER BY created_at LIMIT 20;
     → Identify stuck sagas and which step they're on.

  2. SELECT step_name, error_message, attempt_count FROM saga_step_executions
     WHERE saga_id = '<stuck_id>' ORDER BY created_at;
     → What error is step N producing?

  3. Check downstream health for step N's service:
     → curl http://fraud-service/health
     → Check error rate in Datadog for fraud-service.

Likely cause → mitigation:

  A. Downstream service (fraud, bank) unavailable
     → Sagas queue automatically; will retry when service recovers.
     → Immediate: verify downstream service health; escalate if needed.
     → If SLA breach imminent: apply fallback rule (allow payments under $X with review flag).

  B. Orchestrator crashed (pods down)
     → kubectl get pods -n payments | grep orchestrator
     → If CrashLoopBackOff: kubectl describe pod <pod> for crash reason.
     → Orchestrator reads last persisted state on restart; sagas resume automatically.

  C. Compensating transaction failing permanently
     → Identify: saga in COMPENSATING state for > 30 minutes.
     → These require manual intervention.
     → INSERT INTO manual_review_queue (saga_id, reason) VALUES (?, 'compensation_failed');
     → Page finance/ops team for manual resolution.

Recovery: saga completion rate returns to baseline for 10 minutes.
```

---

## Design Review Checklist

```text
□ All valid states enumerated? All invalid transitions rejected?
□ Saga state persisted to DB before each step is called?
□ Every step call is idempotent? Idempotency key derived from saga_id?
□ Every reversible step has a compensating action defined?
□ Compensating actions are idempotent?
□ What happens if compensation fails permanently? Manual review queue?
□ Irreversible steps placed last in the saga? (email, external bank call)
□ Saga versioning strategy defined? (what happens to in-flight sagas on code deploy?)
□ Alert: stuck sagas (IN_PROGRESS for > 2× expected duration)?
□ Admin API to force-cancel or force-advance a saga?
□ Maximum saga age before considered abandoned? Action taken?
□ Dashboard: in-flight sagas by step? Visible to on-call?
□ Orchestrator team identified? On-call rotation defined?
```

---

## Threat Model

```text
T1: Saga Replay Attack (Tampering)
  Attack: replay a completed saga event to trigger a second execution.
  Mitigation: idempotency key on every step; ON CONFLICT DO NOTHING in DB;
              saga events carry version/nonce; replayed events rejected.

T2: Compensation Bypass (Elevation of Privilege)
  Attack: trigger a saga failure at step N after step N-1 has applied a benefit.
  Example: claim a refund by deliberately failing step 3 after step 2 credited an account.
  Mitigation: compensation requires the same auth as the original action;
              rate limit compensation actions per user per day;
              flag accounts with unusually high compensation rate for review.

T3: Orchestrator Impersonation (Spoofing)
  Attack: send fake "step complete" message to orchestrator to advance a saga.
  Mitigation: step completion messages signed with service's mTLS certificate;
              orchestrator verifies sender identity before advancing state machine.
```

---

*End of Pattern 3: Multi-Step Processes*
# Pattern 4: Scaling Reads

> "Most systems are 90% reads. The database bottleneck is almost always on the read path. Fix the read path and the database breathes again."

---

## The Shape

A service is handling far more reads than writes. Read latency is too high. The database is the bottleneck. You need reads to be faster, cheaper, and more scalable without sacrificing correctness (or with deliberate, bounded staleness).

---

## When You See This Pattern

- Product catalog (10M reads/sec, 100 writes/sec)
- News feed / timeline (100M reads/day)
- User profile service (every API call reads the profile)
- IAM permission checks (500K checks/sec)
- Search autocomplete
- Dashboard metrics (everyone reads the same aggregation)

The signal: **reads >> writes, and the database is either slow or expensive at this read volume**.

---

## The Core Tension

```text
Freshness ←———————————————————————→ Performance
(always current data)                 (fast, cheap reads)

Cache everything ←————————————————→ Cache nothing
(fast but stale)                      (slow but accurate)

One database ←————————————————————→ Many read replicas
(simple but bottleneck)               (scalable but lag risk)
```

---

## Building Blocks

### 1. Indexing (Cheapest Fix First)

Before adding infrastructure, check if the read is slow due to a missing index.

```sql
EXPLAIN ANALYZE
SELECT * FROM payments WHERE user_id = 123 ORDER BY created_at DESC LIMIT 20;

-- Output: Seq Scan on payments (rows=50000000 width=300) Execution time: 8200ms
-- This is a full table scan on 50M rows. Missing index.

CREATE INDEX idx_payments_user_created ON payments(user_id, created_at DESC);

-- After: Index Scan (rows=20 width=300) Execution time: 2ms
-- 4100× speedup. No new infrastructure.
```

Check indexing before any other scaling approach. It is free and often sufficient.

---

### 2. Read Replicas

Add replica databases that receive all writes from the primary via replication. Route read queries to replicas.

```text
Primary (writes + reads)         Read Replicas (reads only)
      │                         ┌──────────────────────────┐
      │ replication stream       │                          │
      ├────────────────────────► Replica 1    Replica 2    │
      │                         │                          │
Application:                    └──────────────────────────┘
  Write: primary                  ↑ all reads
  Read: replica pool (round-robin)
```

**The replication lag problem**: Replicas are typically 10ms–5s behind the primary (asynchronous replication). This causes:

- User updates their password → reads from replica → sees old password → confusion
- User places an order → reads order confirmation from replica → order not visible yet

**Read-your-writes consistency**: After a write, read from the primary for that user for the next 5 seconds. Or use a replication position watermark.

```python
def read_user_profile(user_id, context):
    if context.recent_write_by(user_id, within_seconds=5):
        return primary_db.read(user_id)  # consistency guaranteed
    else:
        return replica_db.read(user_id)  # fast path
```

---

### 3. Caching

Store the result of expensive reads in a fast cache (Redis, Memcached, in-process).

```text
Cache-Aside (most common):
  1. Read: check cache first
  2. Cache hit: return immediately
  3. Cache miss: read from DB, write to cache, return

Write-Through:
  On every write: update both DB and cache simultaneously
  Cache always has the latest data
  Higher write latency; eliminates cache misses on hot data

Read-Through:
  Cache sits in front of DB; cache fetches from DB on miss
  Application only talks to cache

Write-Back:
  Write to cache first; async write to DB
  Risk: data loss if cache fails before DB write
  Never use for financial data
```

**Cache sizing rule**: Cache the "hot 20%." For most systems, 20% of records account for 80% of reads (Pareto). Cache that 20% and achieve ~80% cache hit rate. Caching 100% of records for a marginal improvement in hit rate is rarely worth the cost.

**TTL selection**:
```text
User profile (changes rarely):           TTL = 10 minutes
Product price (changes sometimes):       TTL = 30 seconds
Stock level (changes constantly):        TTL = 5 seconds or no cache
IAM permissions (security-critical):     TTL = 30 seconds + explicit invalidation on change
Session token (must be exact):           No TTL → explicit invalidation only
```

---

### 4. Materialized Views

Pre-compute expensive aggregations and store them as first-class data.

```text
Expensive query (runs on every dashboard load):
  SELECT merchant_id, SUM(amount), COUNT(*), AVG(amount)
  FROM payments
  WHERE created_at > NOW() - INTERVAL '24 hours'
  GROUP BY merchant_id;
  -- 500ms on 50M rows. Runs every 10 seconds per merchant dashboard.

Materialized view (pre-computed):
  CREATE MATERIALIZED VIEW merchant_daily_stats AS
  SELECT merchant_id, SUM(amount) as total, COUNT(*) as count, AVG(amount) as avg
  FROM payments
  WHERE created_at > NOW() - INTERVAL '24 hours'
  GROUP BY merchant_id;

  REFRESH MATERIALIZED VIEW CONCURRENTLY merchant_daily_stats;
  -- Run this every 60 seconds (pg_cron)

  Query: SELECT * FROM merchant_daily_stats WHERE merchant_id = 123;
  -- 2ms. No computation.
```

**CQRS (Command Query Responsibility Segregation)**: Extend this idea across services. Write to one (normalised) model; read from a different (denormalised, pre-computed) model.

---

### 5. CDN

For static or infrequently changing content (images, CSS, product photos, documentation), a CDN serves the content from edge nodes geographically close to the user.

```text
Without CDN: User in Mumbai fetches image from US server. Latency: 200ms.
With CDN:    User in Mumbai fetches from CDN edge in Mumbai. Latency: 5ms.

CDN cache hit ratio = (requests served from edge) / (total requests)
At 95% cache hit ratio: only 5% of requests hit your origin. 20× load reduction.
```

**Cache-Control headers** drive CDN behavior:
```text
Cache-Control: public, max-age=86400     → CDN caches for 24 hours
Cache-Control: private, no-store         → CDN does not cache (per-user content)
Cache-Control: public, max-age=3600, stale-while-revalidate=86400
                                         → serve stale for 24h while revalidating in background
```

---

## Canonical Problem: Design a Product Catalog Read Layer (50M reads/day)

### Requirements
- 10M products, 50M read requests/day (~580 reads/sec average, 5K peak)
- Product data changes: ~1000 updates/day
- Read latency: p99 < 50ms
- Freshness: within 5 minutes of any update

### Scaling the Read Path

```text
Request flow:

Client → CDN (cache: product images, static data)
    │ miss or dynamic data
    ▼
API Layer → L1 In-Process Cache (Caffeine, TTL=60s)
    │ miss
    ▼
Redis Cache (TTL=5min)
    │ miss
    ▼
PostgreSQL Read Replica
    │ (primary for writes)
    ↑
Product Write Service → Primary DB → CDC → Cache Invalidator
                                          → Redis DEL product:{id}
```

**Cache invalidation on update**: When a product is updated, the write service invalidates the Redis key immediately. The next read re-populates from the database with fresh data. The 5-minute TTL is a safety net, not the primary invalidation mechanism.

**Why in-process cache (L1)?**: At 5K req/sec across 10 API nodes, each node handles 500 req/sec. A Redis lookup per request = 500 × 1ms = 500ms of Redis round-trip time per second per node (just for this endpoint). An in-process cache with 60-second TTL absorbs most of that: 500 req/sec × 60s = a key is fetched from Redis once per 60 seconds regardless of request rate.

---

## Deep Dive: Cache Stampede

**What it is**: A popular cache key expires. Hundreds of requests simultaneously find a cache miss and simultaneously query the database.

```text
"top_products" key expires at T=0.
500 concurrent requests: all see cache miss.
500 requests all query DB for top_products.
DB: 500 concurrent identical queries. CPU 100%. Timeouts cascade.
```

**Fix 1: Probabilistic early recomputation (XFetch)**

```python
def cached_get(key, fetch_fn, ttl):
    entry = cache.get_with_ttl(key)  # returns (value, remaining_ttl)
    if entry is None:
        value = fetch_fn()
        cache.set(key, value, ttl)
        return value
    
    value, remaining_ttl = entry
    # Probability of recomputing increases as expiry approaches
    # XFetch: recompute with probability = 1 - (remaining_ttl/ttl)^2
    if random() < (1 - (remaining_ttl / ttl) ** 2):
        # Recompute in background; return stale value now
        executor.submit(lambda: cache.set(key, fetch_fn(), ttl))
    
    return value  # always fast; background refresh happens before expiry
```

**Fix 2: Mutex (single-filler lock)**

```python
def cached_get(key, fetch_fn, ttl):
    value = cache.get(key)
    if value:
        return value
    
    lock_key = f"lock:{key}"
    acquired = cache.set(lock_key, "1", nx=True, px=5000)  # 5s lock
    if acquired:
        try:
            value = fetch_fn()
            cache.set(key, value, ttl)
            return value
        finally:
            cache.delete(lock_key)
    else:
        # Wait for the winner to populate the cache
        time.sleep(0.05)  # 50ms backoff
        return cache.get(key) or fetch_fn()  # fallback to DB if still empty
```

**Fix 3: TTL jitter**
```python
ttl = base_ttl + random.randint(0, base_ttl // 5)  # ±20% jitter
```

---

## Interview Calibration

### What a Staff Engineer Adds Beyond Senior

- "Before adding a cache, let me check if the query has the right index."
- Distinguishes cache-aside from write-through from CQRS — and chooses deliberately.
- Addresses read-your-writes consistency: "After a user updates their profile, they must see the update even if we route reads to replicas."
- Handles cache stampede as a first-class design concern.
- Quantifies the cache hit ratio needed to meet the latency SLO.
- Asks about the write pattern: "How often does this data change? That determines TTL."

---

*End of Pattern 4: Scaling Reads*

---

## Evolution Path

```text
Stage 1: Single DB, Direct Queries (0–5K reads/sec)
  SELECT with basic WHERE. Add indexes first — free and immediate.
  Breaking point: index-covered queries still slow; table too large for indexes alone.
  Signal: EXPLAIN shows seq scan despite indexes; bloat from MVCC vacuum lag.

Stage 2: Read Replicas (5K–50K reads/sec)
  Route read queries to replicas. 2–3 replicas give 3× read throughput.
  Breaking point: replica lag causes stale reads; or all replicas still saturated.
  Signal: replica lag > 1s; replica CPU > 70%.

Stage 3: Application Cache — Redis (50K–500K reads/sec)
  Cache hot records. 80% hit ratio → 80% reduction in DB reads.
  Breaking point: cache invalidation too complex; hit ratio < 60%; or cache memory > budget.
  Signal: cache miss rate rising; DB reads not dropping despite cache.

Stage 4: CQRS / Pre-computed Read Models (500K–5M reads/sec)
  Write model: normalised, correct. Read model: denormalised, fast.
  CDC keeps read model in sync. Reads serve pre-joined, pre-aggregated data.
  Breaking point: single-region read model; distant users pay cross-region latency.
  Signal: p99 latency from distant regions > SLO.

Stage 5: Multi-Region Cache Hierarchy (5M+ reads/sec)
  CDN (static), regional Redis (hot data), regional DB replicas (cold cache miss).
  Each region is self-sufficient for reads.
  Breaking point: this handles essentially all read-scaling requirements.
```

---

## Migration Strategy

**From direct DB reads to Redis cache**

```text
Step 1: Instrument (zero behaviour change).
  Add cache-miss counters. Identify top 10 most-queried hot keys.
  Run for 48 hours. Output: "product:123 queried 80K times/day."

Step 2: Shadow write (populate cache without reading from it).
  On every DB read: write result to Redis with TTL. Do not serve from cache yet.
  Validate: Redis values match DB values. Run for 24 hours.

Step 3: Enable cache reads for read-only data first (zero staleness risk).
  Static config, feature flags, product catalog (rarely changes).
  Monitor: DB reads drop; cache hit ratio climbs.

Step 4: Enable cache reads for mutable data with TTL.
  User profiles (TTL=10min), session data (TTL=30s).
  Add: on-write cache invalidation (delete cache key when DB row updated).
  Monitor: hit ratio, staleness complaints, read-your-writes violations.

Step 5: Validate and scale.
  Target: hit ratio > 80%; DB read rate reduced > 60%.
  If not met: increase cache size or add explicit pre-warming.

Rollback at any step: disable cache reads (feature flag). Fallback to DB instantly.
Data risk: none. Cache is always advisory; DB is always the source of truth.
```

---

## Organisational Ownership

```text
Platform / DB team owns:
  - PostgreSQL primary + replicas, connection pooling (PgBouncer).
  - Redis cluster: capacity planning, eviction policy, TLS config.
  - CDC pipeline (Debezium + Kafka) for cache invalidation.
  On-call: DB health, replica lag, Redis OOM.

Application team owns:
  - Cache key design (what to cache, at what granularity).
  - TTL selection (how stale is acceptable per data type).
  - Cache invalidation logic (which writes should invalidate which keys).
  - Read-your-writes routing (when to read from primary vs replica).
  On-call: cache hit ratio below threshold, staleness complaints.

Ownership boundary conflict:
  "Cache is stale" — was the invalidation call missed (application) or did CDC lag (platform)?
  Resolution: platform team monitors CDC pipeline lag; application team monitors explicit invalidation success.
```

---

## Incident Response

**Symptom**: API latency p99 spike 5× / DB CPU spike / user sees stale data

```text
Diagnose (first 5 minutes):
  1. redis-cli PING → no response? Redis is down. All reads hitting DB.
  2. redis-cli INFO stats | grep keyspace_hits keyspace_misses
     → hit ratio = hits / (hits + misses). If < 50%: stampede or Redis eviction.
  3. DB: SELECT count(*) FROM pg_stat_activity WHERE state='active';
     → If 5× normal: cache failure is hammering DB.
  4. Check replica lag:
     SELECT now() - pg_last_xact_replay_timestamp() FROM pg_stat_replication;

Likely cause → mitigation:

  A. Redis cluster node failure
     → Cluster promotes replica automatically (30–60s).
     → During failover: 5× DB load. Ensure DB connection pool has headroom.
     → If DB overloading: enable circuit breaker; serve stale from in-process cache (Caffeine).
     → Return to normal after Redis failover completes.

  B. Cache stampede (mass TTL expiry)
     → Symptom: periodic DB CPU spikes on a schedule (every TTL seconds).
     → Immediate: temporarily set all TTLs to staggered values via:
       redis-cli OBJECT IDLETIME key → add jitter based on idle time.
     → Permanent: add random jitter to all cache writes.

  C. Cache key design mismatch (low hit ratio)
     → redis-cli --hotkeys: which keys are most accessed?
     → If top keys have 0 hits: key format in application ≠ key stored.
     → Check: is the application generating cache keys consistently?

Recovery: DB read rate returns to cached baseline for 5 minutes.
```

---

## Design Review Checklist

```text
□ Indexes verified first? (EXPLAIN ANALYZE before adding any cache infrastructure)
□ Cache hit ratio target defined? Memory budget calculated for that target?
□ TTL justified per data type? Freshness requirement documented?
□ Cache invalidation strategy? (TTL only, active delete, CDC-driven)
□ Read-your-writes: after a write, does the user see their update?
□ Cache stampede mitigated? (jitter on TTL, XFetch, or mutex lock)
□ What happens if Redis fails? Does the system degrade or break?
□ In-process (L1) cache for highest-traffic keys to absorb Redis load?
□ Replica lag monitoring with alert threshold (e.g., > 1s = alert)?
□ Circuit breaker: stop routing to lagging replica when lag > N seconds?
□ CDC pipeline lag monitored if using CDC for invalidation?
□ Cache hit ratio alert: if drops below 70%, investigate?
□ Platform team on-call defined for Redis and replica incidents?
```

---

## Threat Model

```text
T1: Cache Poisoning (Tampering)
  Attack: attacker writes malicious data to cache (if cache is accessible).
  Mitigation: Redis protected by VPC isolation + auth (requirepass / ACL);
              cache keys include a hash of the source data version;
              never expose Redis to public network.

T2: Timing Attack via Cache (Information Disclosure)
  Attack: measure response time to determine if a record exists in cache.
  Cache hit (fast response) reveals: "user 123 exists."
  Mitigation: add constant-time jitter to API responses for sensitive lookups;
              use opaque IDs (UUID) so attacker can't enumerate valid IDs.

T3: Replica Lag Exploitation (Information Disclosure)
  Attack: user resets password; attacker reads old password hash from lagging replica.
  Mitigation: password/credential lookups always go to primary (never replica);
              security-sensitive reads never served from cache.
```

---

*End of Pattern 4: Scaling Reads*
# Pattern 5: Scaling Writes

> "Reads scale with hardware. Writes scale with architecture."

---

## The Shape

The system generates writes faster than a single database instance can absorb. The write path is the bottleneck. You need to distribute, defer, or batch writes without losing data.

---

## When You See This Pattern

- Analytics / telemetry (millions of events/second)
- Social media (likes, views, comments at scale)
- IoT sensor data
- Logging and audit trails
- Financial transaction ledgers
- Ride-sharing GPS updates

The signal: **write volume exceeds what a single database can absorb, or write latency is unacceptable at the current volume**.

---

## The Core Tension

```text
Durability ←—————————————————————→ Throughput
(every write persisted immediately)   (batch/defer for speed)

Consistency ←————————————————————→ Partition tolerance
(global single view of writes)        (distribute writes, accept lag)

Simple schema ←——————————————————→ Write-optimised schema
(natural domain model)                (append-only, event log, wide column)
```

---

## Building Blocks

### 1. Write Queues / Write Buffering

Don't write to the database immediately. Buffer writes in a fast queue and flush in batches.

```text
Without buffering:
  1M events/sec → 1M individual DB inserts/sec → PostgreSQL saturated

With Kafka buffering:
  1M events/sec → Kafka (1M/sec capacity per partition) → consumer batch-inserts
  Consumer: INSERT INTO events VALUES (1000 rows) per batch → 1000 events/insert
  DB: 1000 batch inserts/sec (instead of 1M individual inserts)
  
  Write amplification reduction: 1000×
  Latency added: up to 100ms (batch interval)
  Acceptable for analytics/logging use cases.
```

---

### 2. Partitioning / Sharding

Distribute writes across multiple database instances.

```text
Hash sharding by user_id:
  user_id % 4:
    Shard 0: users 0, 4, 8, 12, ...
    Shard 1: users 1, 5, 9, 13, ...
    Shard 2: users 2, 6, 10, 14, ...
    Shard 3: users 3, 7, 11, 15, ...

4 shards → 4× write throughput
Query routing: hash(user_id) → shard number → direct query to that shard
```

**The hot shard problem**: If your shard key creates uneven distribution (e.g., shard by geography: 50% of users are in one city → one shard handles 50% of writes), one shard becomes the bottleneck.

**Reshard without downtime** (the hard problem): Growing from 4 shards to 8 shards requires moving data. The transition period requires dual writes (write to old and new shard), then validation, then cutover. Plan this from day 1.

---

### 3. Append-Only / Event Log

Instead of updating rows in place (which requires finding the row, locking it, updating it), write only new rows. Current state is derived from the log.

```text
Traditional (update-in-place):
  UPDATE balances SET amount = 500 WHERE account_id = 123
  
  Requires: find row (index lookup), lock row, update row, update index
  Write amplification: 1 logical write → multiple physical writes

Append-only (event log):
  INSERT INTO ledger (account_id, delta, type, ts) VALUES (123, -100, 'DEBIT', NOW())
  INSERT INTO ledger (account_id, delta, type, ts) VALUES (123, +600, 'CREDIT', NOW())
  
  No locking. No index update (on sequential append). Sequential I/O.
  Throughput: 10-100× higher than update-in-place.
  Current balance: SELECT SUM(delta) FROM ledger WHERE account_id = 123
  
  Materialise balance for fast reads: maintain a separate balances table as a cache.
```

**Why append-only scales**: Sequential disk writes are 100-1000× faster than random writes. An append-only log requires no index maintenance on write (only an append to the time-ordered index). Cassandra, Kafka, and PostgreSQL WAL all use this principle.

---

### 4. Batching

Group multiple writes into a single database operation.

```sql
-- Instead of:
INSERT INTO events (user_id, type, ts) VALUES (1, 'click', NOW());  -- ×1000
INSERT INTO events (user_id, type, ts) VALUES (2, 'click', NOW());
-- ...

-- Use:
INSERT INTO events (user_id, type, ts)
VALUES (1, 'click', T1), (2, 'click', T2), ..., (1000, 'click', T1000);
-- One network round-trip. One transaction overhead. 1000× fewer DB round-trips.
```

**In Java (Spring JDBC)**:
```java
jdbcTemplate.batchUpdate(
    "INSERT INTO events (user_id, type, ts) VALUES (?, ?, ?)",
    events.stream()
        .map(e -> new Object[]{e.userId(), e.type(), e.timestamp()})
        .collect(toList())
);
```

**Batch size tuning**:
- Too small: too many round-trips, defeats the purpose
- Too large: transaction holds longer, more lock contention, single failure loses more data
- Rule of thumb: 100–1000 rows per batch for most use cases

---

### 5. Write Aggregation / Pre-Aggregation

For metrics and counters, don't write individual events to the database. Aggregate in memory first.

```text
Example: video view counts

Naive: each view → INSERT INTO view_events (video_id, user_id, ts)
  At 10M views/day per popular video: 115 inserts/sec on one video → DB bottleneck

Better: aggregate in memory (per worker) → periodic flush
  Worker receives 10,000 views for video-123 in 60 seconds
  Worker: UPDATE videos SET view_count = view_count + 10000 WHERE id = 123
  DB: 1 update/minute per video. Not 10,000 inserts/minute.
  
  Tradeoff: if worker crashes before flush, up to 60 seconds of counts are lost.
  Acceptable for view counts. Not acceptable for payment records.
```

---

## Canonical Problem: Design a Write-Heavy Analytics Pipeline (100M events/day)

### Requirements
- 100M events/day (~1200 events/sec average, 10K peak)
- Event schema: user_id, event_type, properties (JSON), timestamp
- Query pattern: aggregate by event_type, time range, user cohort
- Latency requirement: events available for query within 5 minutes

### Architecture

```text
Event Producers
    │
    ▼ (publish, fire-and-forget)
Kafka (10 partitions, partitioned by user_id)
    │
    ▼
Consumer Group (5 consumers)
    │
    │ (batch: every 1000 events or 5 seconds, whichever first)
    ▼
ClickHouse (columnar analytics DB)
    │
    ▼
Query API (aggregate queries, dashboards)
```

**Why Kafka, not direct writes?**: Producers are isolated from consumer speed. If ClickHouse is slow or down for maintenance, events accumulate in Kafka (durable). Producers don't slow down or fail.

**Why ClickHouse, not PostgreSQL?**: ClickHouse is a columnar analytics database. Aggregation queries (COUNT, SUM, GROUP BY) over billions of rows: PostgreSQL ~30s, ClickHouse ~100ms. For OLAP workloads, columnar storage provides 100-1000× query speedup.

**Why partition by user_id?**: Events from the same user land on the same partition → events are ordered per user → queries like "all events for user X in order" can be answered from one partition.

---

## Deep Dive: Hot Shards

### What Causes Them

Shard key creates uneven distribution. One shard gets disproportionate traffic.

```text
Shard by user_id % 4:
  user_id=1: shard 1
  user_id=2: shard 2
  ...
  
Looks even. But user 1 is a celebrity with 10M followers.
Every follow/unfollow/post action for user 1's social graph lands on shard 1.
Shard 1: 80% of writes. Shards 2-4: 20% combined.
```

### Mitigation

**Virtual shards**: `hash(user_id + "_" + (timestamp % 10)) % num_shards`. One user's data is spread across 10 virtual shards. Queries for that user: scatter-gather across 10 shards (acceptable for analytics; avoid for real-time lookups).

**Separate hot entities**: Detect entities above a threshold. Route them to a dedicated "hot" shard with more resources.

**Time-based bucketing**: For time-series data, append a time bucket to the shard key:
```text
shard key: user_id + "_" + YYYYMM
user 1, January:  shard for (user1, Jan)
user 1, February: shard for (user1, Feb)
```

Hot users are sharded across time automatically as months progress.

---

## Interview Calibration

### What a Staff Engineer Adds

- Identifies whether the bottleneck is write throughput or write latency (different solutions)
- Proposes Kafka as a write buffer before jumping to sharding
- Asks about durability requirements: "Can we lose up to 60 seconds of view counts? Or is every write critical?"
- Addresses the resharding problem: "How do we add shards without downtime?"
- Considers write amplification: "Every write to the DB also updates 3 indexes. Can we reduce index count?"
- Asks about the query pattern: "OLAP aggregations → ClickHouse. OLTP lookups → PostgreSQL. Don't use one for both."

---
---

## Evolution Path

```text
Stage 1: Direct DB Writes (0–5K writes/sec)
  INSERT directly to PostgreSQL. Simple. Correct.
  Breaking point: DB write latency > 50ms; DB I/O wait > 30%.
  Signal: PostgreSQL IOPS at disk limit; WAL write rate saturated.

Stage 2: Connection Pooling + Batch Writes (5K–50K writes/sec)
  PgBouncer pools connections. Application batches multiple INSERTs into one statement.
  Cost: free. Impact: 5–10× effective write throughput.
  Breaking point: single PostgreSQL node CPU saturated despite pooling.
  Signal: DB CPU > 80% sustained; connection pool wait time growing.

Stage 3: Write Queue (50K–500K writes/sec)
  Kafka buffers writes. Consumer batch-inserts to DB at controlled rate.
  Decouples producer speed from DB write speed. Adds 1–5s write-to-read latency.
  Breaking point: single DB still can't absorb consumer write rate.
  Signal: Kafka consumer lag growing despite batching at max batch size.

Stage 4: Table Partitioning (data > 100GB or > 100M rows)
  Partition by created_at (monthly). Each partition: small, fast, indexable.
  Old partitions: archive to S3 (parquet). Frees primary DB storage.
  Breaking point: single-node DB still bottleneck for writes even with partitioning.
  Signal: VACUUM taking > 2 hours; write WAL rate at disk limit.

Stage 5: Sharding (> 1M writes/sec or > 1TB hot data)
  Distribute writes across N shards by shard key.
  Breaking point: hot shard forms; or cross-shard queries too slow.
  Signal: one shard CPU >> others; scatter-gather queries > SLO.

Stage 6: Purpose-Built Write-Optimised DB (> 10M writes/sec)
  Cassandra, ScyllaDB: 100K–1M writes/sec per node, auto-sharding, LSM-tree storage.
  ClickHouse: 500K+ rows/sec per node for append-only analytics workloads.
  Breaking point: rarely hit. Scaling is near-linear by adding nodes.
```

---

## Migration Strategy

**From direct PostgreSQL inserts to Kafka-buffered batch writes**

```text
Step 1: Deploy Kafka. No consumers yet. No behaviour change.

Step 2: Dual publish: write to both DB directly (existing) and Kafka (new).
  Consumer reads from Kafka but writes to a shadow table.
  Validate: shadow table matches primary table. Run for 48 hours.

Step 3: Switch consumer to write to primary table (not shadow).
  Maintain direct-write path as fallback.
  Compare: data in Kafka path vs direct path for 24 hours.

Step 4: Canary — route 10% of writes to Kafka-only path.
  Monitor: write success rate, end-to-end latency (write to readable).
  Key change for callers: write acknowledgement now means "Kafka received it"
  not "DB committed it." Consumers of this data must tolerate 1–5s latency.

Step 5: Route 100% of writes to Kafka path.
  Remove direct-write code after 7-day stable operation.

Rollback: re-enable direct-write path via feature flag (seconds).
Data risk: writes in Kafka not yet consumed. On rollback: consumer drains Kafka first,
          then direct writes resume. No data loss (Kafka is durable).
```

---

## Organisational Ownership

```text
Data Platform team owns:
  - Kafka cluster (broker health, partition count, retention policy).
  - Consumer infrastructure (batch consumers, autoscaling).
  - ClickHouse / analytics DB (if present).
  On-call: Kafka broker failure, consumer lag growing, batch write failure rate.

Application team owns:
  - Event schema design (Kafka message format, Avro schema).
  - Partition key choice (determines ordering and hot shard risk).
  - Consumer business logic (what to do with each event).
  On-call: their consumer's lag, their data's correctness.

Ownership conflict:
  "Kafka consumer lag growing" — is the consumer too slow (application) or
  Kafka under-provisioned (platform)?
  Resolution: consumer throughput metric (events/sec/consumer) published by application.
  If consumer at max throughput → add partitions (platform).
  If consumer below capacity → fix consumer code (application).
```

---

## Incident Response

**Symptom**: Write latency spike / Kafka consumer lag alert / data pipeline delay

```text
Diagnose (first 5 minutes):
  1. kafka-consumer-groups.sh --describe --group <writer-group>
     → Check lag per partition. One partition lagging >> others? Hot partition.
     → All partitions lagging equally? Consumer too slow or Kafka broker issue.
  2. Check Kafka broker health:
     → kafka-topics.sh --describe → any under-replicated partitions?
     → If yes: broker failure in progress.
  3. Application consumer CPU/memory:
     → kubectl top pods -n data-pipeline
     → Is consumer CPU at 100%? Scale horizontally.

Likely cause → mitigation:

  A. Kafka broker failure
     → Cluster elects new leader for affected partitions (30–60s).
     → In-flight messages: not lost (replicated to surviving brokers).
     → Consumer lag: will grow during election; catches up after new leader elected.
     → Action: verify under-replicated partition count returns to 0.

  B. Hot partition (one partition >> others)
     → Identify: which partition_key is causing concentration?
     → Immediate: add entropy to partition key: hash(key + bucket).
     → Long-term: increase partition count (requires consumer group rebalance).

  C. Consumer too slow for current throughput
     → Immediate: scale consumer pods (kubectl scale --replicas=+4).
     → Ceiling: max consumers = number of partitions (Kafka limit).
     → If at partition ceiling: add partitions (platform team change).

Recovery: consumer lag returns to < 1000 messages for 10 minutes.
```

---

## Design Review Checklist

```text
□ Partition key chosen for uniform distribution? Hot shard scenario tested?
□ Consumer idempotent? (safe to process same event twice)
□ Batch size tuned? (100–1000 rows per INSERT; benchmarked at production load)
□ Kafka retention long enough for consumer to catch up from failure?
□ Consumer lag the primary health metric? Alert threshold defined?
□ Schema registry enforcing backward compatibility?
□ What is the acceptable write-to-readable latency? Consumers of this data informed?
□ Partition count planned for peak throughput? (throughput / 50K events/sec/partition)
□ Resharding strategy planned? (what happens when we need to add partitions?)
□ Alert: consumer lag growing for > 5 minutes?
□ Alert: under-replicated Kafka partitions > 0?
□ Alert: any shard/partition receiving > 3× average write rate?
```

---

## Threat Model

```text
T1: Event Injection (Tampering)
  Attack: attacker publishes malicious events to Kafka topic.
  Mitigation: Kafka ACLs (SASL/SCRAM or mTLS); producers authenticated;
              schema registry validates event structure before acceptance;
              consumers validate event authenticity before processing.

T2: Partition Key Enumeration (Information Disclosure)
  Attack: guess partition keys to determine data volume per entity.
  Mitigation: hash partition keys (don't use raw entity IDs as partition keys);
              partition key values not exposed in monitoring dashboards.

T3: Consumer Crash Loop as DoS
  Attack: publish malformed event that crashes the consumer (poison message).
  Mitigation: dead letter queue (DLQ) after N retries; consumer catches all exceptions;
              schema validation at publish time prevents most malformed events.
```

---

*End of Pattern 5: Scaling Writes*
# Pattern 6: Handling Large Blobs

> "A database is not a file system. A file system is not a database. Use each for what it is."

---

## The Shape

Users upload large binary content: videos, images, documents, audio files. These can range from kilobytes to gigabytes. Storing them in a relational database is technically possible and practically catastrophic. Delivering them efficiently requires a completely different architecture from your API data.

---

## When You See This Pattern

- File storage (Dropbox, Google Drive)
- Video upload and streaming (YouTube, Netflix upload)
- Image sharing (Instagram, S3 as backend)
- Document management (PDFs, contracts)
- Profile photos, product images

The signal: **the content is large, binary, and accessed differently from your structured data**.

---

## The Core Tension

```text
Durability ←————————————————————→ Cost
(never lose a byte)                 (storage is expensive at petabyte scale)

Upload reliability ←—————————————→ Simplicity
(resume interrupted uploads)         (direct PUT to server)

Global delivery ←————————————————→ Storage location
(low latency everywhere)              (data must be somewhere)
```

---

## Building Blocks

### 1. Object Storage

Never store large files in your database. Use object storage: S3, GCS, Azure Blob Storage.

```text
Database: optimised for structured queries, small row sizes, indexes
Object storage: optimised for large binary objects, high throughput reads, cheap at scale

1TB in PostgreSQL: requires careful DB sizing, backups are slow, query impact
1TB in S3: $23/month, infinite scale, no DB overhead, 99.999999999% durability
```

**The architecture pattern**:
```text
Client → API server → issues presigned URL → S3
                          ↓
Client → uploads directly to S3 (bypasses your server)
                          ↓
S3 → triggers event → your server → records metadata in DB
```

Your server never touches the file bytes. It only handles metadata.

---

### 2. Presigned URLs

A presigned URL is a time-limited URL with embedded credentials that allows a client to directly upload to or download from object storage without going through your API server.

```python
# Generate presigned upload URL (server-side):
url = s3.generate_presigned_url(
    ClientMethod='put_object',
    Params={
        'Bucket': 'uploads',
        'Key': f'videos/{user_id}/{video_id}.mp4',
        'ContentType': 'video/mp4',
        'ContentLength': expected_bytes,
    },
    ExpiresIn=3600  # URL valid for 1 hour
)
# Return URL to client

# Client uploads directly to S3:
PUT {presigned_url}
Content-Type: video/mp4
{bytes}
```

**Why this matters at scale**: Without presigned URLs, every byte of every upload flows through your API servers. At 10K concurrent uploads of 1GB files: your API servers need ~80Gbps of network bandwidth. With presigned URLs: your API servers handle only metadata (kilobytes). S3 handles the bytes.

---

### 3. Multipart Upload

Files larger than ~100MB should be split into parts and uploaded in parallel, with each part independently resumable.

```text
File: 1GB video

Without multipart:
  PUT /upload (1GB body)
  Network drops at 99%: restart from 0.
  User tries again: starts over.

With multipart:
  Step 1: POST /createMultipartUpload → upload_id
  
  Step 2: Upload parts in parallel (each ~5MB):
    PUT /uploadPart?uploadId=XXX&partNumber=1  {bytes 0-5MB}    → ETag1
    PUT /uploadPart?uploadId=XXX&partNumber=2  {bytes 5-10MB}   → ETag2
    ...
    PUT /uploadPart?uploadId=XXX&partNumber=200 {bytes 995MB-1GB} → ETag200
  
  Step 3: Complete:
    POST /completeMultipartUpload { uploadId, [ETag1, ETag2, ...ETag200] }
    → S3 assembles the file

Network drops at part 150? Only re-upload parts 150–200. 75% of work saved.
```

**Client-side tracking**:
```javascript
// Store upload progress in localStorage:
localStorage.setItem(`upload:${fileHash}`, JSON.stringify({
    uploadId: 'XXX',
    completedParts: [1, 2, 3, ...149],
    totalParts: 200
}));
// On resume: continue from part 150
```

---

### 4. CDN for Delivery

Never serve large files from your origin servers for anything beyond trivial scale.

```text
Without CDN:
  User in Mumbai requests 4K video → US S3 origin → 200ms latency, 50Mbps bandwidth used

With CDN:
  First request: CDN edge in Mumbai → S3 origin (cache miss) → cached at Mumbai edge
  All subsequent requests: served from Mumbai edge → 5ms latency, S3 bandwidth: zero

Cache hit rate after warmup: typically 90-99% for popular content.
Origin load reduction: 10-100×.
```

**Cache-Control for CDN**:
```text
Static assets (CSS, JS): Cache-Control: public, max-age=31536000, immutable
User-generated content (videos, photos): Cache-Control: public, max-age=86400
Private content: Cache-Control: private, no-store (CDN doesn't cache)
```

**Signed URLs for private content**: User's documents should not be publicly accessible. Use CDN signed URLs (time-limited access tokens embedded in the URL).

---

## Canonical Problem: Design Dropbox

### Requirements
- Upload files up to 10GB
- Sync across devices
- Share with other users
- Version history (last 30 revisions)
- 500M users, 100M daily active

### Architecture

```text
Upload flow:
  Client app → API: POST /files (metadata: name, size, hash)
  API → issues presigned S3 URL
  Client → S3: multipart upload (resume-able, hash-verified per chunk)
  S3 → event: file uploaded → Lambda → API: record file in DB

DB schema:
  files: (file_id, user_id, name, path, size, hash, s3_key, created_at)
  versions: (version_id, file_id, s3_key, hash, size, created_at)
  shares: (share_id, file_id, shared_with_user_id, permission, expires_at)

Download flow:
  Client → API: GET /files/{id}/download
  API: generate presigned download URL (5 minute expiry)
  Client → CDN/S3: download via presigned URL

Sync flow:
  Client: sends delta (list of file hashes) since last_sync_time
  API: returns list of added/modified/deleted files since that time
  Client: downloads only changed files
```

**Deduplication with content-addressed storage**:
```text
hash = SHA256(file_bytes)
s3_key = f"content/{hash[:2]}/{hash[2:]}"

If two users upload identical files:
  hash is identical → same S3 key
  Only one copy stored in S3
  Both users' metadata records point to the same S3 object

Storage saved: 100% for identical files (common: OS files, shared documents, duplicates)
```

---

## Deep Dive: Resumable Uploads

**The problem**: Mobile network drops mid-upload. How do you resume without starting over?

```text
Protocol: TUS (https://tus.io) or custom implementation

POST /upload/create
  Headers: Upload-Length: 1073741824  (1GB)
  Response: Location: /upload/upload_id_xyz

PATCH /upload/upload_id_xyz
  Headers: Content-Range: bytes 0-5242879/1073741824
           Upload-Offset: 0
  Body: {bytes 0-5MB}
  Response: Upload-Offset: 5242880

[network drop at offset 104857600 (100MB)]

[reconnect]
HEAD /upload/upload_id_xyz
  Response: Upload-Offset: 104857600

PATCH /upload/upload_id_xyz
  Headers: Upload-Offset: 104857600
  Body: {bytes 100MB onwards}
  → continues from where it left off
```

---

## Interview Calibration

**What separates Staff level**: Understanding that presigned URLs are not just a performance optimisation — they are a security boundary. The URL is signed with credentials scoped to one object. The client never gets AWS credentials. The URL expires. The object key is determined server-side (no path traversal attacks). This is the secure design, not just the scalable one.

---

*End of Pattern 6: Handling Large Blobs*

---

## Evolution Path

```text
Stage 1: Blob in Database (0–10GB total, small files only)
  Store file bytes as BYTEA in PostgreSQL. Simple. Terrible at scale.
  Breaking point: > 10MB files cause DB bloat; backup time explodes; query performance degrades.
  Signal: VACUUM taking hours; backup size >> data size; DB storage cost growing fast.

Stage 2: Filesystem on Application Server (10GB–1TB, single server)
  Store files on local disk; serve via application.
  Breaking point: server fails → files gone; can't scale horizontally.
  Signal: disk full; server is SPOF for file delivery.

Stage 3: Object Storage — S3 / GCS (1TB–1PB, standard scale)
  All files in object storage. Application stores only metadata in DB.
  Presigned URLs for direct client upload/download (bypasses application).
  Breaking point: egress cost becomes dominant (especially without CDN).
  Signal: monthly egress bill > 50% of total infrastructure cost.

Stage 4: CDN in Front of Object Storage (any scale with static/popular content)
  CDN caches popular objects at edge nodes globally.
  At 95% hit ratio: S3 egress drops 95%; delivery latency drops to 5ms globally.
  Breaking point: private/per-user content can't be CDN-cached (no shared cache key).
  Signal: CDN cache hit ratio < 70% (data too dynamic or too personalised).

Stage 5: Multi-Region Object Storage (global user base with residency requirements)
  S3 buckets per region; cross-region replication for DR; geo-routing for access.
  Breaking point: this handles essentially any scale.
```

---

## Migration Strategy

**From files-on-disk to S3 + presigned URLs**

```text
Step 1: Set up S3 bucket with correct ACLs (private by default).
        Configure lifecycle rules: move objects > 90 days to S3-IA.

Step 2: Dual-write new uploads: local disk + S3.
        Existing files: batch migration job (read from disk, write to S3, verify hash).
        Migration: run at off-peak hours; throttle to not affect application.

Step 3: Switch download path to S3 presigned URLs.
        Old files: served from disk via application.
        New files: served via presigned URL.
        Feature flag controls which path each file uses.

Step 4: Complete migration of all existing files.
        Validate: every file_id has a corresponding S3 key with matching SHA256.

Step 5: Remove local disk storage.
        Update application to reject any writes to local disk.

Step 6: Add CDN in front of S3 for public/semi-public content.

Rollback: at any step, disable S3 path; serve from local disk.
Data risk: dual-write ensures no data loss until Step 5.
```

---

## Organisational Ownership

```text
Platform / Storage team owns:
  - S3 bucket policy, lifecycle rules, cross-region replication.
  - CDN configuration (cache rules, invalidation API).
  - Presigned URL generation library (published as internal SDK).
  On-call: S3 unavailability, CDN outage, upload failure rate > threshold.

Application team owns:
  - Metadata schema (what to store in DB alongside the S3 key).
  - Access control logic (who can generate presigned URLs for which objects).
  - Virus scanning integration (if required).
  - File retention and deletion logic.
  On-call: application-level upload failures, access denied errors.
```

---

## Incident Response

**Symptom**: Upload failures / "file not found" errors / CDN returning 404

```text
Diagnose:
  1. Is S3 available? aws s3 ls s3://bucket-name → if hangs: S3 regional issue.
  2. Are presigned URLs generating? Test URL generation endpoint directly.
  3. CDN "file not found": check if object exists in S3 (CDN miss → 404 means S3 also 404).
  4. Upload failure: check multipart upload state:
     aws s3api list-multipart-uploads --bucket bucket-name
     → Stale multipart uploads? Abort them (lifecycle rule: abort after 7 days).

Mitigation:
  S3 regional outage: failover to secondary region (if multi-region configured).
  Presigned URL expired: client must request a new URL (short TTL is intentional).
  CDN 404 for existing file: manual CDN cache invalidation for the affected path.
```

---

## Design Review Checklist

```text
□ Presigned URLs used for upload? (bypasses application servers; critical for scale)
□ Presigned URL scoped to specific object key? (no path traversal possible)
□ Presigned URL TTL appropriate? (upload: 1hr; download: 5min for private; 24hr for semi-private)
□ Multipart upload for files > 100MB? Client can resume interrupted uploads?
□ Content-type validation server-side? (not trusting client-provided Content-Type)
□ Virus scanning before file is accessible? (async scan after upload complete)
□ CDN configured for public content? Cache hit ratio target defined?
□ Private content: signed CDN URLs (not public S3 URLs)?
□ S3 lifecycle rules: move old objects to cheaper storage tier?
□ Deduplication by content hash? (saves storage for identical files)
□ Backup/DR strategy? (cross-region S3 replication or periodic backup)
□ Alert: upload failure rate > 1%?
□ Alert: CDN cache hit ratio < 70%?
```

---

## Threat Model

```text
T1: Path Traversal via Presigned URL (Elevation of Privilege)
  Attack: manipulate the object key in a presigned URL to access another user's file.
  Mitigation: server determines object key (e.g., uploads/{user_id}/{uuid});
              presigned URL is for that specific key only; client cannot modify the key.

T2: Public Bucket Misconfiguration (Information Disclosure)
  Attack: S3 bucket accidentally set to public-read; all files accessible without auth.
  Mitigation: S3 Block Public Access enabled at account level;
              regular audit of bucket policies via AWS Config;
              alert: any bucket policy change triggers immediate security review.

T3: Malicious File Upload (Tampering / Availability)
  Attack: upload malware, oversized files, or zip bombs.
  Mitigation: enforce max file size at presigned URL generation (Content-Length header limit);
              virus scanning (ClamAV or cloud service) before file is accessible;
              file type validation (magic bytes, not just extension).

T4: Presigned URL Sharing (Information Disclosure)
  Attack: user shares their presigned download URL; unauthorized users access the file.
  Mitigation: short TTL (5 minutes for sensitive files);
              watermark sensitive documents with user identity;
              log all presigned URL generations for audit trail.
```

---

*End of Pattern 6: Handling Large Blobs*
# Pattern 7: Long-Running Tasks

> "The user clicked 'export report'. You have 10 seconds before the HTTP connection times out. The report takes 10 minutes. What now?"

---

## The Shape

An operation takes longer than a synchronous HTTP request can accommodate. It must be executed in the background, with the client able to track progress and retrieve results.

---

## When You See This Pattern

- Video transcoding (minutes to hours)
- Report generation (seconds to minutes)
- ML model training or inference
- Data export / bulk operations
- Email campaign send (send to 10M addresses)
- Virus scanning uploaded files
- Background index rebuilds

The signal: **the operation takes longer than ~10 seconds, or the client doesn't need to wait for completion before continuing**.

---

## Building Blocks

### The Async Job Pattern

```text
Synchronous (wrong for long tasks):
  Client: POST /reports/generate
  Server: [runs for 10 minutes]
  Client: HTTP timeout at 30 seconds.
  Client: Did it start? Is it running? Do I retry?

Async (correct):
  Client: POST /reports/generate → 202 Accepted { job_id: "job-123" }
  Server: enqueues the job, returns immediately

  Worker (background): processes the job
  
  Client: GET /jobs/job-123 → { status: "RUNNING", progress: 45% }
  Client: GET /jobs/job-123 → { status: "COMPLETE", result_url: "..." }
```

**The 202 Accepted pattern**: The HTTP 202 response means "I've accepted your request and will process it asynchronously." Always include a job_id and a polling URL in the response body.

---

### Job Queue + Workers

```text
┌──────────────────────────────────────────────────────────┐
│                    Job Queue (Kafka/SQS)                  │
│  [transcode:video-123] [report:user-456] [export:org-789] │
└──────────────────────────────────────────────────────────┘
         ↑                    ↓
    API enqueues          Workers consume
    
Worker 1: processes transcode:video-123
Worker 2: processes report:user-456
Worker 3: processes export:org-789

Workers are stateless. Add more workers to scale throughput.
```

**Job visibility and concurrency control (SQS pattern)**:
```text
Worker: ReceiveMessage → job becomes "invisible" (visibility timeout: 10 minutes)
Worker: processes job for 8 minutes
Worker: job complete → DeleteMessage (removes from queue)

If worker crashes at minute 5:
  Visibility timeout expires at 10 minutes
  Job reappears in queue
  Another worker picks it up
  [must be idempotent: already-processed steps should be skipped]
```

---

### Job State Tracking

The client needs to query job status. Store job state in Redis or the database.

```sql
CREATE TABLE jobs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type        VARCHAR(50) NOT NULL,       -- 'VIDEO_TRANSCODE', 'REPORT_GENERATE'
  status      VARCHAR(20) NOT NULL,       -- 'QUEUED', 'RUNNING', 'COMPLETE', 'FAILED'
  input       JSONB NOT NULL,
  result      JSONB,
  progress    INTEGER,                    -- 0-100
  error       TEXT,
  created_by  UUID,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  started_at  TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  expires_at  TIMESTAMPTZ                -- for cleanup
);
```

Worker updates progress:
```python
def transcode_video(job_id, video_id):
    update_job(job_id, status='RUNNING', progress=0)
    
    for i, chunk in enumerate(video_chunks):
        transcode_chunk(chunk)
        progress = int((i + 1) / total_chunks * 100)
        update_job(job_id, progress=progress)  # update every chunk
    
    result_url = upload_result()
    update_job(job_id, status='COMPLETE', result={'url': result_url})
```

---

### Worker Reliability

**Idempotency in workers**: A job may be processed more than once (worker crash, visibility timeout, retry). Design workers to be idempotent:

```python
def process_report_job(job):
    # Check if output already exists (idempotency check)
    existing = s3.head_object(bucket='reports', key=f'reports/{job.id}.pdf')
    if existing:
        # Already generated. Just record the result URL.
        update_job(job.id, status='COMPLETE', result={'url': s3_url(job.id)})
        return
    
    # Generate report (idempotent operations)
    data = fetch_report_data(job.params)
    pdf = generate_pdf(data)
    s3.put_object(bucket='reports', key=f'reports/{job.id}.pdf', body=pdf)
    update_job(job.id, status='COMPLETE', result={'url': s3_url(job.id)})
```

**Dead letter queues**: After N retries, move failed jobs to a DLQ. Alert engineering. Never silently drop failed jobs.

**Timeout management**:
```python
# Worker must heartbeat to extend visibility timeout for long jobs
def process_with_heartbeat(job, queue_message):
    heartbeat_thread = threading.Thread(
        target=extend_visibility_loop,
        args=(queue_message, interval=60, max_extension=3600)
    )
    heartbeat_thread.start()
    try:
        actual_work(job)
    finally:
        heartbeat_thread.stop()
```

---

## Canonical Problem: Video Processing Pipeline (YouTube-Style)

### Requirements
- Users upload videos (up to 4K, up to 10GB)
- Transcoding to multiple resolutions: 360p, 720p, 1080p, 4K
- Thumbnail generation
- Available for playback: within 30 minutes of upload
- Scale: 500 hours of video uploaded per minute

### Architecture

```text
Upload:
  Client → API: POST /videos (metadata)
  API → presigned S3 URL → Client → S3 (multipart)
  S3 event → message → Kafka: video-uploads topic

Processing:
  TranscodeOrchestrator consumes video-uploads
  For each video: submits 4 transcode jobs (one per resolution)
    Job: { video_id, s3_source_key, target_resolution, output_key }
    Jobs enqueued in: jobs:transcode queue (SQS)
  
  TranscodeWorker pool (autoscaling, GPU instances):
    Consume from jobs:transcode
    Download source from S3 → transcode → upload output to S3
    Update job status in DB
    Emit: TranscodeComplete event → Kafka

  ThumbnailWorker:
    Consume TranscodeComplete for 720p jobs
    Extract thumbnail frames
    Upload thumbnails to S3
    Update video record

  IndexWorker:
    When all resolutions complete: mark video as AVAILABLE in DB
    Update search index

Status API:
  GET /videos/{id}/status → check jobs table, return progress
```

**The autoscaling key insight**: Video transcoding is CPU/GPU-intensive. At 500 hours/minute upload rate, you need massive parallelism. Workers are stateless → horizontal scale with autoscaling groups. Scale up when queue depth grows; scale down when idle. This is the canonical long-running task autoscaling pattern.

---

## Deep Dive: Duplicate Work

**The scenario**: A worker picks up a transcode job. It processes for 45 minutes. The visibility timeout was set to 30 minutes. The job reappears in the queue. A second worker starts transcoding the same video.

**Prevention**:
1. Set visibility timeout longer than the maximum job duration + buffer
2. Heartbeat to extend visibility timeout
3. Idempotent workers: check if output already exists before starting

**Detection**:
```sql
-- Find jobs with multiple concurrent workers:
SELECT job_id, COUNT(*) as worker_count
FROM active_job_leases
GROUP BY job_id
HAVING COUNT(*) > 1;
-- Should always be empty. Alert if not.
```

---

## Interview Calibration

**Staff-level additions**:
- Distinguishes queue depth scaling (autoscale workers when queue grows) from time-based scaling
- Addresses the "job stuck forever" failure mode (job never completes, never fails, heartbeat keeps extending TTL)
- Proposes a job timeout watchdog: "a background process scans for jobs that have been RUNNING for more than 2× their expected duration and moves them to FAILED"
- Addresses priority: "video transcoding jobs for premium users should be prioritised. Use separate queues with separate worker pools for different priority tiers."

---

*End of Pattern 7: Long-Running Tasks*

---

## Evolution Path

```text
Stage 1: Synchronous Processing in API Handler (0–10 jobs/sec, fast jobs)
  API handler runs the job inline. Simple. Blocks until complete.
  Breaking point: jobs take > 10 seconds; HTTP timeout fires.
  Signal: client timeouts; API gateway timeout errors.

Stage 2: Background Threads + DB Status Table (10–100 jobs/sec)
  API returns immediately with job_id; background thread processes.
  Status stored in DB; client polls GET /jobs/{id}.
  Breaking point: background threads exhaust memory; no retry on crash.
  Signal: thread count at max; jobs lost when server restarts.

Stage 3: Queue + Worker Pool (100–10K jobs/sec)
  Jobs enqueued to SQS/Kafka; dedicated worker pool processes.
  Workers are stateless; autoscale based on queue depth.
  Breaking point: all workers same instance type; GPU-heavy jobs mixed with CPU-light jobs.
  Signal: GPU workers starved by CPU-light jobs; cost efficiency poor.

Stage 4: Typed Job Queues + Specialised Workers (10K+ jobs/sec)
  Separate queues per job type: GPU transcoding, CPU reporting, IO export.
  Separate autoscaling groups per queue type.
  Spot instances for GPU workers (70% cost reduction).
  Breaking point: rarely hit. Adding queues + worker types handles any scale.

Stage 5: Dedicated Workflow Engine (complex, long-lived, versioned)
  Temporal: handles retries, timeouts, versioning, activity heartbeats natively.
  For jobs that last hours/days, have conditional steps, or require external signals.
```

---

## Migration Strategy

**From synchronous to async job processing**

```text
Step 1: Add job_id to API response (even while still synchronous).
  Response: { job_id: "job-123", status: "COMPLETE", result: {...} }
  Clients: update to use job_id for future status polling.

Step 2: Add jobs table to DB. Start persisting job records.
  No behaviour change yet — jobs still execute synchronously.

Step 3: Move execution to background thread.
  API returns 202 Accepted with job_id immediately.
  Background thread executes and updates job status in DB.
  Monitor: job completion rate, error rate.

Step 4: Replace background thread with queue + workers.
  Workers consume from queue. Same DB status table.
  Client polling endpoint unchanged.
  Rollback: re-enable background thread processing via feature flag.

Step 5: Add autoscaling for workers based on queue depth.
Step 6: Migrate GPU-heavy jobs to dedicated spot-instance workers.
```

---

## Organisational Ownership

```text
Platform / Jobs team owns:
  - Queue infrastructure (SQS/Kafka/RabbitMQ).
  - Worker fleet management (autoscaling, spot instance lifecycle).
  - Job status API (standard endpoint all teams use).
  - DLQ monitoring and alerting.
  On-call: queue unavailability, worker fleet health, DLQ depth growing.

Application team owns:
  - Job logic (what the worker does).
  - Idempotency of job execution.
  - Progress reporting (updating job status during execution).
  - Retry logic (which errors are retriable; which are permanent failures).
  On-call: their job type's failure rate, correctness issues.
```

---

## Incident Response

**Symptom**: Jobs stuck in QUEUED for hours / DLQ depth alert / job failure rate spike

```text
Diagnose:
  1. Queue depth: is it growing (workers can't keep up) or stuck (workers crashed)?
     aws sqs get-queue-attributes --attribute-names ApproximateNumberOfMessages
  2. Worker health: kubectl get pods -n jobs | grep worker
     → CrashLoopBackOff: workers are crashing on a specific job type.
  3. DLQ: which jobs are in DLQ? Sample 5 and check their error_message.
     → Same error type? Likely a code bug or downstream service outage.
     → Random errors? Likely transient failures; check retry configuration.

Mitigation:
  A. Workers crashed: kubectl rollout restart deployment/job-workers
     → If OOM: reduce batch size or increase memory limit.
  B. Poison message in queue: identify job_id, move to DLQ manually, resume processing.
  C. Downstream dependency down: circuit break; move jobs to waiting state; retry when dependency recovers.

Recovery: queue depth decreasing; DLQ not growing; job completion rate back to baseline.
```

---

## Design Review Checklist

```text
□ Workers idempotent? (safe to process same job twice after crash/retry)
□ Visibility timeout > maximum job duration + buffer?
□ Workers heartbeat to extend visibility timeout for long jobs?
□ DLQ configured? Max retries before DLQ defined?
□ DLQ monitored with alert?
□ Stuck job detector: alert if job in RUNNING for > 2× expected duration?
□ Job progress visible externally (% complete for UX)?
□ Autoscaling configured? Trigger: queue depth > N for > 2 minutes?
□ Spot instances used for GPU-heavy workers? Interruption handling implemented?
□ Job output stored in S3 (not DB)? Job result URL returned to client?
□ Maximum job age: what happens to jobs never completed? Fail after N days?
□ On-call team defined for worker fleet and queue infrastructure?
```

---

## Threat Model

```text
T1: Job Injection (Elevation of Privilege)
  Attack: attacker enqueues jobs on behalf of other users or with elevated permissions.
  Mitigation: job_id tied to submitting user_id at enqueue time;
              worker validates user's permissions before executing job;
              queue ACLs (only authorised services can enqueue).

T2: Resource Exhaustion via Job Flooding (DoS)
  Attack: attacker submits millions of jobs exhausting worker capacity and queue storage.
  Mitigation: per-user job submission rate limit;
              per-user max concurrent jobs limit;
              queue size cap with 429 on overflow.

T3: Sensitive Data in Job Payload (Information Disclosure)
  Attack: job payloads logged or exposed via status API contain PII/secrets.
  Mitigation: job payload stored encrypted in queue;
              status API returns job metadata only (not input payload);
              job logs scrubbed of PII before shipping to log aggregator.
```

---

*End of Pattern 7: Long-Running Tasks*
# Pattern 8: Search Systems

> "Full-text search is a different problem from database lookup. The moment users can type anything and expect intelligent results, you've left the relational world."

---

## The Shape

Users type natural language queries and expect relevant results from a large corpus of text. Relevance matters as much as correctness. Results must be fast (<100ms). The index must stay reasonably fresh.

---

## When You See This Pattern

- Product search (Amazon, Flipkart)
- Document search (Google Drive, Notion)
- User search (LinkedIn, Twitter)
- Log search (Kibana)
- Autocomplete / typeahead

The signal: **users can search with free-form text, and ranking by relevance matters**.

---

## Building Blocks

### Inverted Index

The fundamental data structure of search. Maps each unique term to the list of documents containing it.

```text
Documents:
  Doc 1: "PayPal payment gateway integration"
  Doc 2: "Stripe payment API for developers"
  Doc 3: "PayPal developer documentation"

Inverted Index:
  "paypal"      → [Doc1, Doc3]
  "payment"     → [Doc1, Doc2]
  "gateway"     → [Doc1]
  "integration" → [Doc1]
  "stripe"      → [Doc2]
  "api"         → [Doc2]
  "developer"   → [Doc2, Doc3]
  "documentation" → [Doc3]

Query: "paypal payment"
  → Lookup "paypal": [Doc1, Doc3]
  → Lookup "payment": [Doc1, Doc2]
  → Intersection (AND) or union (OR): Doc1 appears in both → most relevant
```

Building and querying an inverted index at scale is what Elasticsearch, Solr, and Lucene do.

---

### TF-IDF: Basic Relevance Ranking

Not all occurrences are equal. A document about "payment" that uses the word 100 times is more relevant than one that uses it once.

```text
TF (Term Frequency): how often the term appears in this document
IDF (Inverse Document Frequency): how rare the term is across all documents

Score = TF × IDF

"payment" appears in 90% of documents → low IDF (common term → low signal)
"PayPal Instant Transfer" → high IDF (rare term → high signal → more relevant when it matches)

High-relevance document: uses rare terms frequently.
Low-relevance document: uses common terms rarely.
```

Modern search (Elasticsearch BM25, neural search) is more sophisticated, but TF-IDF captures the intuition.

---

### Autocomplete / Typeahead

User types "pay" → suggest "PayPal", "Payment gateway", "Pay later"

```text
Implementation options:

1. Prefix trie (in-memory):
   TRIE: p → pa → pay → paym → payme → paymen → payment
              → payp → paypa → paypal
   
   Query "pay": returns all words in the subtree rooted at "pay"
   O(prefix_length) lookup. Extremely fast.
   
2. Redis ZSET:
   ZADD autocomplete:products 0 "payment" 0 "paypal" 0 "pay later"
   ZRANGEBYLEX autocomplete:products "[pay" "[pay\xff"
   → Returns all strings starting with "pay" in alphabetical order
   
3. Elasticsearch prefix query:
   { "query": { "prefix": { "name": "pay" } } }
```

**Popularity weighting**: "Apple" should rank above "Apple cider" for a phone store. Weight suggestions by historical query frequency:
```text
ZADD autocomplete:products <popularity_score> "apple"
ZADD autocomplete:products <lower_score> "apple cider"
ZREVRANGEBYSCORE → returns most popular first
```

---

### Index Freshness

How quickly do new/updated documents appear in search results?

```text
Indexing pipeline:

Write to DB → CDC (Debezium) → Kafka → Elasticsearch indexer
  
Latency:
  DB commit → Kafka: ~100ms (CDC)
  Kafka → Elasticsearch indexer: ~100ms (consumer lag)
  Elasticsearch index: ~1s (refresh interval)
  Total: 1-2 seconds from write to searchable

Elasticsearch refresh interval:
  Default: 1 second (near-real-time search)
  For high-write scenarios: increase to 30s (reduces indexing overhead 30×)
  Trade-off: writes to search index are batched → better throughput, slightly less fresh
```

---

## Canonical Problem: Product Search (E-Commerce)

### Requirements
- 10M products
- 50M search queries/day (~580 QPS average, 5K peak)
- Results within 100ms
- Relevance: price, rating, in-stock status, keyword match
- Typo tolerance

### Architecture

```text
Indexing path (writes):
  Product Service → DB → CDC (Debezium) → Kafka → ES Indexer → Elasticsearch

Search path (reads):
  Client → API → Elasticsearch cluster (3 nodes, 10 shards, 1 replica)
  
Elasticsearch query:
  {
    "query": {
      "function_score": {
        "query": { "multi_match": { "query": "red running shoes", "fields": ["name^3", "description", "category"] }},
        "functions": [
          { "filter": { "term": { "in_stock": true }}, "weight": 1.5 },
          { "field_value_factor": { "field": "rating", "modifier": "log1p" }}
        ]
      }
    },
    "suggest": {
      "spell_check": { "term": { "field": "name" }}
    }
  }
```

**Typo tolerance**: Elasticsearch uses edit distance (Levenshtein) for fuzzy matching:
```json
{ "query": { "fuzzy": { "name": { "value": "nikie shoes", "fuzziness": "AUTO" }}}}
```
"nikie" → matches "nike" (edit distance 1).

**The dual-write problem**: Search index and database can diverge. The CDC pipeline is asynchronous — search may lag by 1-2 seconds. For product search, this is acceptable. For inventory "in stock" flags, this requires careful TTL management or a direct DB lookup on the result set.

---

## Interview Calibration

**What makes a Staff-level answer**:
- Never says "use Elasticsearch" without explaining the inverted index
- Explicitly separates the **indexing path** (writes) from the **search path** (reads)
- Addresses index freshness tradeoffs: "if I set refresh_interval=30s for better write throughput, search results are 30 seconds stale"
- Identifies that Elasticsearch is not a primary datastore: "all data lives in PostgreSQL; Elasticsearch is a derived index rebuilt from CDC"
- Discusses the "rebuild index from scratch" scenario (ES cluster failure): "since it's derived from Kafka, we can replay from the beginning of the product topic"

---

*End of Pattern 8: Search Systems*

---

## Evolution Path

```text
Stage 1: LIKE Query on DB (0–100K documents)
  SELECT * FROM docs WHERE name LIKE '%query%'. Simple. Catastrophically slow at scale.
  Breaking point: > 100K rows or any query taking > 500ms.
  Signal: LIKE query showing seq scan in EXPLAIN; p99 latency > SLO.

Stage 2: Full-Text Search in PostgreSQL (100K–5M documents)
  PostgreSQL tsvector + GIN index. Decent keyword search.
  Breaking point: relevance ranking is basic; no fuzzy matching; no facets.
  Signal: user complaints about relevance; engineers hacking around missing features.

Stage 3: Elasticsearch / OpenSearch (5M–500M documents)
  Purpose-built search with BM25 ranking, fuzzy matching, facets, autocomplete.
  CDC pipeline keeps index in sync with DB.
  Breaking point: keyword search misses semantic matches; users phrase queries naturally.
  Signal: "relevant" results not surfaced; user search abandonment rate rising.

Stage 4: Hybrid Keyword + Vector Search (semantic understanding required)
  Add embedding model; store dense vectors alongside inverted index.
  Hybrid scoring: BM25 + cosine similarity.
  Breaking point: generic embeddings don't capture domain-specific meaning.
  Signal: domain-specific terminology not matching correctly (e.g., medical, legal).

Stage 5: Domain-Fine-Tuned Embeddings + Neural Re-ranking (best relevance)
  Fine-tune embedding model on domain data. Re-rank top-K with learning-to-rank model.
  This is where Google, Amazon, LinkedIn operate.
  Requires: ML infrastructure (training pipeline, serving infrastructure, A/B testing).
```

---

## Incident Response

**Symptom**: Search returning no results / search latency spike / index out of sync

```text
Diagnose:
  1. curl http://elasticsearch:9200/_cluster/health
     → Red: data loss (some shards unassigned). Yellow: some replicas missing. Green: OK.
  2. Replication lag: check Kafka consumer lag for search-indexing consumer group.
  3. Index staleness: compare doc count in ES vs DB:
     curl http://elasticsearch:9200/products/_count
     SELECT COUNT(*) FROM products;
  4. Search latency: GET /_nodes/stats/indices/search → avg query latency.

Mitigation:
  Red cluster: immediately check unassigned shards; likely node failure.
    GET /_cluster/allocation/explain
  Index lag growing: scale up indexing consumers.
  Full re-index needed: trigger from Kafka beginning-of-topic replay.
    POST /products/_delete_by_query {"query":{"match_all":{}}}  ← careful in production
    Then: reset consumer group offset to beginning; consumer re-indexes all documents.
```

---

## Design Review Checklist

```text
□ Elasticsearch is a derived index — primary data lives in PostgreSQL?
□ CDC pipeline (Debezium) keeping index in sync? Lag monitored?
□ Shard count sized correctly? (10–65GB per shard; ES 8.x guidance)
□ Replica count ≥ 1? (lose one node without data loss)
□ Index rebuild procedure documented and tested?
□ Tenant isolation enforced? (user A cannot see user B's private documents)
□ Search results access-controlled at application layer (not just index)?
□ Autocomplete: using prefix queries or edge-ngram? Response time < 100ms?
□ Fuzzy matching: fuzziness=AUTO configured?
□ Alert: cluster health != green?
□ Alert: indexing consumer lag > threshold?
□ Alert: search latency p99 > SLO?
□ Alert: index doc count diverges from DB by > 1%?
```

---

## Threat Model

```text
T1: Tenant Data Leakage (Information Disclosure)
  Attack: user searches for documents from another tenant.
  Mitigation: every document tagged with tenant_id;
              every search query filtered by tenant_id (enforced at search layer);
              penetration test: verify cross-tenant isolation quarterly.

T2: Search Query Injection (Tampering)
  Attack: attacker injects Elasticsearch query DSL via search input.
  Mitigation: use template-based queries (not raw user input in query DSL);
              sanitise user input; use field-level search (not generic query_string).

T3: Index Enumeration (Information Disclosure)
  Attack: use search to enumerate private records (e.g., "show all records where SSN starts with 123").
  Mitigation: rate limit search QPS per user; minimum query length enforced;
              sensitive fields excluded from search index entirely.
```

---


---

## Migration Strategy

**From PostgreSQL full-text search to Elasticsearch**

```text
Step 1: Deploy Elasticsearch alongside existing PostgreSQL full-text search.
        Bulk-index all existing documents from PostgreSQL into Elasticsearch.
        Dark launch: run all queries against both; compare results.

Step 2: A/B test: 5% of search traffic goes to Elasticsearch.
        Metric: zero-results rate, result click-through rate, latency.

Step 3: Ramp to 100% if metrics improve. Maintain dual indexing for 30 days.

Step 4: Remove PostgreSQL full-text index (frees storage; reduces write overhead).
        Elasticsearch is now the sole search path. CDC pipeline keeps it in sync.

Rollback: feature flag returns traffic to PostgreSQL full-text in seconds.
```

---

## Organisational Ownership

```text
Search Platform team owns: Elasticsearch cluster, CDC indexing pipeline, index schema.
  On-call: cluster health, index lag, search error rate.

Product team owns: query construction, relevance tuning, search result quality.
  On-call: zero-results rate, CTR drop, user complaints about relevance.
```

---

*End of Pattern 8: Search Systems*
# Pattern 9: Notification Systems

> "A notification is a message that someone asked to receive. Spam is a message nobody asked for. The line between them is preference management."

---

## The Shape

Events in your system need to be communicated to users via multiple channels: in-app, push notification, email, SMS. At scale, this means fan-out to millions of recipients, across multiple channels, with delivery guarantees, preference management, and deduplication.

---

## When You See This Pattern

- Social media activity notifications ("Bob liked your post")
- Payment confirmations and fraud alerts
- System alerts and operational notifications
- Marketing campaigns (email, push)
- IAM security events ("New login from unknown device")

---

## Building Blocks

### Fan-Out on Write vs Fan-Out on Read

```text
Fan-out on Write:
  When Event E happens: immediately compute all N recipients and enqueue N notification tasks
  
  Pros: notifications delivered with minimal additional latency per recipient
  Cons: one event → N tasks; celebrity events cause massive spikes

Fan-out on Read:
  When Event E happens: store the event once
  When user opens app: "give me all notifications since my last check"
  
  Pros: no write amplification; storage is one record regardless of recipients
  Cons: notifications are "pull"; user doesn't know about the event until they open app

Hybrid (Instagram/Facebook approach):
  Regular users: fan-out on write (push notifications delivered promptly)
  Celebrities (high follower count): fan-out on read (aggregate on demand)
```

---

### Channel Routing

```text
Notification Event
      │
      ▼
Notification Service
  (looks up: user preferences + event type → determines channels)
      │
      ├──→ In-App (real-time, SSE/WebSocket)
      ├──→ Push Notification (APNs/FCM)
      ├──→ Email (SendGrid/SES)
      └──→ SMS (Twilio) [high-priority only]

User preferences (stored per user per event type):
  user-123:
    payment.received: { push: true, email: true, sms: false, in_app: true }
    marketing.campaign: { push: false, email: true, sms: false, in_app: false }
    security.new_login: { push: true, email: true, sms: true, in_app: true }
```

---

### Deduplication and Rate Limiting

```text
Problem: User likes and unlikes a post 5 times in 30 seconds.
  → 5 "liked your post" notifications to the author?
  
Solution: Notification collapsing
  Within a 5-minute window: dedup "liked" events from the same event source
  Send one notification: "Bob and 4 others liked your post"

Problem: Marketing team sends 3 campaigns in one day to the same user.
  Solution: per-user notification rate limit
  
  Redis: INCR notif:count:user-123:email:20240115
         EXPIRE notif:count:user-123:email:20240115 86400
  If count > daily_limit: suppress (or queue for next day)
```

---

### Reliable Delivery

Notifications must be delivered at-least-once. Missing a "your payment of ₹5,000 succeeded" is a support ticket. Missing a "fraud detected on your account" is a security incident.

```text
Delivery pipeline with retries:

Notification → Queue (Kafka) → Delivery Worker
                                    │
                                    ├── Attempt 1: FCM → OK: mark delivered
                                    ├── Attempt 2 (if fail): FCM → OK: mark delivered
                                    ├── Attempt 3 (if fail): FCM → fail: DLQ
                                    └── DLQ: manual investigation or fallback channel

Retry strategy:
  Attempt 1: immediate
  Attempt 2: 30 seconds
  Attempt 3: 5 minutes
  Attempt 4: 1 hour
  Give up: DLQ + alert
```

---

## Canonical Problem: Notification Service (IAM Security Alerts)

### Requirements
- Security events: new login, role change, account locked → notify user within 30 seconds
- Marketing events: monthly report, feature announcements → no real-time requirement
- User can configure: which events trigger which channels
- 10M users, 100K security events/day, 10M marketing events/week

### Architecture

```text
Event Sources:
  IAM Service emits: LoginSuccess, RoleGranted, AccountLocked, PasswordChanged

Notification Orchestrator:
  Consumes events from Kafka: iam.security-events
  For each event:
    1. Load user notification preferences from Redis (TTL=5min)
    2. Apply collapsing rules (is there a recent similar notification to this user?)
    3. Determine channels: push + email + SMS for security events
    4. Enqueue delivery tasks to channel-specific queues

Channel Workers:
  push-worker: calls FCM/APNs → updates delivery status
  email-worker: calls SES → updates delivery status  
  sms-worker: calls Twilio (only for AccountLocked/SuspiciousLogin) → updates delivery status

Delivery Status:
  notifications table: (notification_id, user_id, event_type, channels, status, delivered_at)
  User can see: "Your account was locked. Notification sent via email and SMS at 10:05 AM."
```

---

## Interview Calibration

**Staff-level additions**:
- Separates "notification fanout" (who receives it) from "channel delivery" (how it's sent) — different scaling concerns
- Addresses preference management as a first-class data model
- Handles the "send notification at user's local time" requirement: "batch marketing emails into a time-zone-aware delivery scheduler; don't send at 3am local time"
- Considers notification fatigue: "even if preferences allow it, rate-limit so a single user never receives more than N notifications per day per channel"

---

---

## Evolution Path

```text
Stage 1: Synchronous Delivery in API Handler (0–1K notifications/sec)
  Send notification inline with the business operation. Simple. Tight coupling.
  Breaking point: email provider slow → payment latency increases.
  Signal: notification provider latency inflating API p99.

Stage 2: Async via DB Queue (1K–10K/sec)
  Write notification task to DB; worker polls and sends.
  Breaking point: DB polling expensive; worker throughput limited.
  Signal: notification delivery lag growing; DB CPU from polling > 20%.

Stage 3: Message Queue + Channel Workers (10K–1M/sec)
  Separate queue per channel (push, email, SMS).
  Workers per channel scale independently.
  Breaking point: fan-out to large recipient lists overwhelms queue.
  Signal: single event causes queue depth to spike; lag grows for all events.

Stage 4: Fan-Out Service + Pre-Computed Recipient Lists (1M+ events/sec)
  Fan-out service computes recipient list asynchronously.
  For celebrities/large groups: fan-out on read (compute on delivery, not on write).
  Breaking point: rarely. This handles most notification systems.

Stage 5: Multi-Region Notification Platform
  Push delivery via regional FCM/APNs endpoints (lower latency).
  Email/SMS: route through nearest regional provider.
  Breaking point: never hit by most systems.
```

---

## Incident Response

**Symptom**: Notification delivery rate drop / DLQ depth growing / user reports missing alerts

```text
Diagnose:
  1. Check channel-specific delivery rates:
     push_delivery_rate, email_delivery_rate, sms_delivery_rate (Prometheus).
  2. FCM/APNs error response codes:
     INVALID_REGISTRATION → stale device tokens (clean token table).
     QUOTA_EXCEEDED → rate limit hit (back off; spread sends).
     UNAVAILABLE → provider outage (queue for retry).
  3. Email bounce rate spiking → domain reputation issue.
     Check SES sending statistics; check bounce rate (should be < 2%).

Mitigation:
  FCM quota exceeded: throttle push; queue excess; flush over time.
  Email provider down: failover to secondary provider (SendGrid → SES or vice versa).
  Missing security alert: escalate to SMS immediately if push + email failing.

Recovery: delivery rate by channel returns to > 98% for 5 minutes.
```

---

## Design Review Checklist

```text
□ Per-user, per-channel rate limit defined? (e.g., max 10 push/hour)
□ Notification collapsing implemented? (N events → "N things happened" summary)
□ Unsubscribe mechanism for every notification type?
□ DLQ for undeliverable notifications? Monitored with alert?
□ Fallback channel defined for critical notifications? (push fails → email)
□ Stale device token cleanup? (remove INVALID_REGISTRATION tokens immediately)
□ Sending time zone awareness? (no 3am marketing emails)
□ Fan-out strategy for high-follower-count events? (fan-out on read for celebrities)
□ Alert: delivery rate < 95% per channel?
□ Alert: DLQ depth growing?
□ Alert: email bounce rate > 2% (domain reputation risk)?
```

---

## Threat Model

```text
T1: Notification Spoofing (Spoofing)
  Attack: send push/email impersonating the platform ("your account has been compromised").
  Mitigation: DKIM/SPF/DMARC configured for email domain;
              push notifications signed by APNs/FCM certificates;
              user education: platform never asks for credentials via notification.

T2: Notification Flooding as DoS (Availability)
  Attack: trigger millions of notifications to exhaust FCM quota or email provider rate limit.
  Mitigation: rate limit notification generation per event source;
              per-user daily notification cap across all events.

T3: Preference Tampering (Tampering)
  Attack: modify another user's notification preferences (disable their security alerts).
  Mitigation: preference changes require auth as the owning user;
              security alert preferences locked (cannot be disabled by user or admin);
              audit log for all preference changes.
```

---


---

## Migration Strategy

**From synchronous notification to async channel-based delivery**

```text
Step 1: Extract notification logic into a dedicated Notification Service.
        Initially: service calls are synchronous (same behaviour, cleaner boundary).

Step 2: Add queue (Kafka) between business events and Notification Service.
        Business events: fire-and-forget to Kafka. Notification Service: consumes.
        Monitor: notification delivery rate vs previous (must be equal or better).

Step 3: Add per-channel workers (push, email, SMS) as separate consumers.
        Each channel scales independently.

Step 4: Add preference management (per-user, per-channel, per-event-type).
        Roll out gradually: let users opt-out before enforcing preferences.

Rollback: re-enable synchronous notification as fallback via feature flag.
```

---

## Organisational Ownership

```text
Platform / Messaging team owns: queue infrastructure, channel workers (FCM, SES, Twilio SDKs),
  delivery rate monitoring, DLQ management.
  On-call: delivery rate drops, provider outages, DLQ growth.

Product teams own: which events trigger which notifications, notification content/templates.
  On-call: their notifications being incorrect or missing.

Security team: approves which events trigger security-category notifications (can't be disabled by users).
```

---

*End of Pattern 9: Notification Systems*
# Pattern 10: Rate Limiting

> "Rate limiting is how a system says 'I know you want more, but I can only give you this much right now.'"

---

## The Shape

A system must limit how much of a resource any single consumer can use in a given time window. Too much consumption from one consumer degrades service for all others — or exhausts the system entirely.

---

## When You See This Pattern

- API gateway (50 requests/second per API key)
- Authentication (5 login attempts per minute per IP)
- Email sending (100 emails/hour per user)
- Payment initiation (10 payments/minute per user)
- Database query rate (prevent runaway queries)

---

## Algorithms

### Token Bucket

A bucket holds N tokens. Each request consumes one token. Tokens refill at rate R per second. If the bucket is empty: reject or wait.

```text
Capacity: 100 tokens
Refill rate: 10 tokens/second

T=0:  bucket=100 → 50 requests → bucket=50
T=1s: bucket=60  (50 + 10 refilled) → 30 requests → bucket=30
T=2s: bucket=40  → 40 requests → bucket=0
T=3s: bucket=10  → 15 requests → 10 served, 5 rejected (bucket empty)

Allows bursts up to bucket capacity.
Sustained rate cannot exceed refill rate.
```

**Best for**: APIs that need to allow short bursts but enforce sustained limits.

---

### Leaky Bucket

Requests enter a queue. The queue drains at a fixed rate. If the queue is full: reject.

```text
Queue size: 100 requests
Drain rate: 10 requests/second

→ Smooths bursty traffic into a steady stream
→ Excess requests are dropped
→ No burst allowed (all requests processed at exactly drain rate)
```

**Best for**: Outbound rate limiting (sending emails, calling external APIs). You want smooth output, not bursty output.

---

### Fixed Window Counter

Count requests in the current time window. Reset at the window boundary.

```text
Limit: 100 requests/minute

Window 1 (10:00:00–10:01:00): 99 requests → OK
Window 2 (10:01:00–10:02:00): 0 requests so far

At 10:00:59: 1 more request → total=100 → OK (limit exactly reached)
At 10:01:00: window resets → counter=0 → 100 more requests allowed

Attack: burst 100 at 10:00:59 + 100 at 10:01:00 = 200 requests in 2 seconds.
Fixed window is vulnerable to boundary attacks.
```

---

### Sliding Window Log

Maintain a timestamp log of all requests. At each request: count how many logs fall within the window. If over limit: reject.

```text
Limit: 100 requests per minute
At T=10:30:45:
  Count timestamps in [10:29:45, 10:30:45] → 98 requests
  98 < 100 → allow. Add 10:30:45 to log.

Accurate. No boundary attack.
Memory: one entry per request in the window (O(limit) per user).
```

---

### Sliding Window Counter (Approximation)

Combines fixed windows with interpolation. Memory efficient, slightly approximate.

```text
Current window (10:01): 30 requests (40% elapsed)
Previous window (10:00): 100 requests

Approximate count = 100 × (1 - 0.4) + 30 = 60 + 30 = 90

If limit is 100: allow (90 < 100).
This approximation is within ~1% of the true sliding window count.
Redis implements this natively.
```

**Best for**: High-scale rate limiting where memory efficiency matters.

---

## Distributed Rate Limiting

Single-node rate limiting is easy (in-process counter). Distributed rate limiting (multiple API servers, shared rate limit) requires coordination.

```text
Problem:
  Rate limit: 100 req/min per API key
  3 API servers, each with local counter
  API key sends 35 req to Server A, 35 to Server B, 35 to Server C
  Each server sees: 35/100 → allow all
  Total: 105 requests served → limit violated

Solution: shared counter in Redis

Redis INCR + EXPIRE:
  MULTI
    INCR rate:api-key-123:20240115-10:30
    EXPIRE rate:api-key-123:20240115-10:30 60
  EXEC
  
  INCR returns the new count.
  If count > limit: reject.
  EXPIRE sets the window TTL.
  
  All API servers share the same Redis counter.
  Atomic INCR: no race condition.
```

**The Redis failure problem**: If Redis is down, do you:
- Allow all traffic (fail open): may allow abuse during Redis downtime
- Deny all traffic (fail closed): service outage for all users
- Fall back to local limits (approximate): allow some abuse, service continues

For most systems: fail open with alerting. For authentication: fail closed or local limits.

---

## Canonical Problem: Rate Limiting for a Payment API

### Requirements
- 10 payment initiations per user per minute (fraud prevention)
- 1000 payments per merchant per minute (throughput limit)
- 50 failed auth attempts per IP per hour (brute force prevention)
- Response time: <10ms for the rate limit check itself

### Architecture

```python
class RateLimiter:
    def check(self, key: str, limit: int, window_seconds: int) -> bool:
        """Sliding window counter using Redis."""
        now = time.time()
        window_key = f"rl:{key}:{int(now // window_seconds)}"
        
        pipe = redis.pipeline()
        pipe.incr(window_key)
        pipe.expire(window_key, window_seconds * 2)  # keep 2 windows for overlap
        count, _ = pipe.execute()
        
        return count <= limit

# Usage:
def initiate_payment(user_id, merchant_id, ...):
    if not rate_limiter.check(f"user:{user_id}:payments", limit=10, window_seconds=60):
        raise RateLimitExceeded("Too many payment attempts. Try again in 1 minute.")
    
    if not rate_limiter.check(f"merchant:{merchant_id}:payments", limit=1000, window_seconds=60):
        raise RateLimitExceeded("Merchant rate limit exceeded.")
    
    # Proceed with payment
```

**Response headers** (standard practice):
```http
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 7
X-RateLimit-Reset: 1700000060
Retry-After: 32  (if rate limited: seconds until reset)
```

---

## Interview Calibration

**What separates Staff level**: Understanding that rate limiting is a **multi-dimensional problem**. Not just "how many per minute" but: per user? per IP? per API key? per merchant? How do you handle bursts vs sustained abuse? What happens when Redis is unavailable? What's the user experience for a legitimately throttled user (clear error message, retry-after header)?

---

*End of Pattern 10: Rate Limiting*

---

## Evolution Path

```text
Stage 1: In-Process Counter (0–10K req/sec, single server)
  Local HashMap. Atomic increment. Zero latency.
  Breaking point: multiple servers → each has its own counter → under-counting.
  Signal: rate limit not working; abuse continues despite limits.

Stage 2: Redis Centralised Counter (10K–1M req/sec)
  Atomic INCR in Redis. Shared across all servers. Correct.
  Breaking point: Redis SPOF; or Redis throughput limit for very high QPS.
  Signal: Redis latency > 5ms (adding > 5ms to every request).

Stage 3: Redis Cluster with Key Sharding (1M+ req/sec)
  Shard rate limit keys across Redis cluster nodes.
  Each key lands on one shard; atomic INCR within shard.
  Breaking point: rarely hit. Redis Cluster handles millions of ops/sec.

Stage 4: CDN-Level Rate Limiting (DDoS protection)
  CloudFront or Cloudflare enforces rate limits at edge.
  Traffic never reaches origin for rate-limited requests.
  Breaking point: never. CDN absorbs DDoS at Tbps scale.
```

---

## Incident Response

**Symptom**: Rate limit not enforcing / all users getting 429 / Redis failure

```text
Diagnose:
  1. redis-cli PING → if down: rate limiter in fail-open or fail-closed mode.
  2. Check 429 rate by endpoint and user:
     rate(http_responses_total{status="429"}[5m]) by (endpoint, user_id)
     → Spike from one user? Legitimate rate limit hit. From all users? Configuration bug.
  3. Check Redis key expiry:
     redis-cli TTL rl:{key} → if -1: key never expires (bug in EXPIRE call).
     → Keys accumulating forever → Redis OOM → Redis crash.

Mitigation:
  Redis down (fail-open mode): rate limiting suspended; log all requests; alert security.
  Redis down (fail-closed mode): all API requests return 429; immediate user impact.
  Preferred for auth endpoints: fail-closed. For API: fail-open with alert.
  Redis key TTL bug: redis-cli SCAN + DEL keys with TTL=-1. Fix code immediately.
```

---

## Design Review Checklist

```text
□ Lua script used for atomic INCR + EXPIRE? (not MULTI/EXEC which has a race condition)
□ Rate limit scoped correctly? (user_id + IP pair, not just one dimension)
□ Retry-After header returned on 429 responses?
□ Redis failure mode defined: fail-open or fail-closed? Justified for this endpoint?
□ CDN-level rate limiting for DDoS scenarios (before traffic hits origin)?
□ Adaptive rate limiting: does limit tighten when downstream is stressed?
□ Per-user daily cap on top of per-minute limit?
□ Rate limit bypass tested? (IP rotation, account sharing, header manipulation)
□ Alert: 429 rate > 5% of requests for any endpoint?
□ Alert: Redis commands rejected (OOM)?
□ Key expiry memory budget calculated? (100M users × 5 keys × 300 bytes = 150GB)?
```

---

## Threat Model

```text
T1: Rate Limit Bypass via IP Rotation (DoS)
  Attack: botnet with thousands of IPs; each IP under per-IP limit.
  Mitigation: rate limit by user_id (post-auth); by device fingerprint (pre-auth);
              threat intelligence IP blocklist; CAPTCHA after N failures from any IP to any account.

T2: Account Enumeration via Timing (Information Disclosure)
  Attack: "user not found" (fast) vs "wrong password" (slow) reveals valid accounts.
  Mitigation: constant-time response for all auth failures;
              same error message ("invalid credentials") for both cases.

T3: Rate Limit Key Exhaustion (DoS)
  Attack: generate millions of unique rate limit keys to exhaust Redis memory.
  Mitigation: Redis maxmemory with allkeys-lru eviction policy;
              rate limit key format designed to bound key space (not attacker-controlled);
              alert on Redis memory > 80%.

T4: Distributed Credential Stuffing (Spoofing)
  Attack: valid username-password pairs from breach databases; each pair tried once (avoids lockout).
  Mitigation: anomaly detection on login success rate from new IPs;
              login velocity alerts per account regardless of IP diversity;
              MFA for sensitive operations.
```

---


---

## Migration Strategy

**From per-server in-process rate limiting to centralised Redis rate limiting**

```text
Step 1: Deploy Redis rate limiter alongside existing in-process limiter.
        Shadow mode: Redis checks run but results ignored. Log would-be rate limit hits.

Step 2: Compare Redis results with in-process results for 48 hours.
        Validate: consistent behaviour; no false positives on legitimate traffic.

Step 3: Enable Redis rate limiter for 10% of traffic (A/B).
        Monitor: 429 rate, legitimate user impact.

Step 4: Ramp to 100%. Remove in-process rate limiter after 7-day stable operation.

Rollback: disable Redis path; in-process rate limiter resumes.
Data risk: brief under-counting possible during transition. Acceptable for rate limiting.
```

---

## Organisational Ownership

```text
Platform / Security team owns: Redis rate limiting infrastructure, rate limit policies.
  On-call: Redis failure (rate limiting suspended), policy misconfiguration.

Application teams own: specifying their rate limit requirements to platform team,
  handling 429 responses gracefully (Retry-After header).
  On-call: their users experiencing unexpected rate limiting.

Security team: sets rate limit thresholds for security-sensitive endpoints (auth, payment).
  Approves any changes to these thresholds.
```

---

*End of Pattern 10: Rate Limiting*
# Pattern 11: Scheduling Systems

> "The hardest part of running a job at 9am every day is: what happens when it runs at 8:59 and 9:01 on the same day?"

---

## The Shape

Work needs to be executed at a specific time, at regular intervals, or after a delay. This work must be reliable (not missed), distributed (not all on one machine), and idempotent (safe to retry).

---

## When You See This Pattern

- Daily report generation
- Email campaign scheduling ("send at 9am user's local time")
- Subscription billing (charge every 30 days)
- Reminder notifications (remind user 1 hour before event)
- Data pipeline runs (ETL every hour)
- Certificate rotation, token cleanup, session expiry

---

## Building Blocks

### Cron (Single Node)

```cron
# Run at 9:00am every day
0 9 * * *  /usr/bin/generate-daily-report.sh

# Run every hour
0 * * * *  /usr/bin/process-pending-payments.sh
```

Works for simple cases. Fails when:
- The server running cron goes down: job is missed
- The job takes longer than the interval: concurrent runs overlap
- You need to know if the job succeeded or failed
- You need to run the job across multiple servers

---

### Distributed Scheduling (Leader-Based)

```text
Multiple worker nodes. One holds the "scheduler" leadership lock.
Only the leader schedules jobs. Followers are standby.

Node A: acquires leader lock (etcd/Redis)
  → schedules jobs on their triggers
  → if job trigger fires: enqueue job to Kafka/SQS

Node A crashes:
  Leader lock TTL expires
  Node B acquires lock → becomes new leader
  Job scheduling continues within 30 seconds

Jobs themselves are processed by a worker pool (not the scheduler).
Scheduler only enqueues; workers execute.
```

---

### Delayed Queues

For "execute this specific task in 30 minutes":

```text
Redis Sorted Set approach:
  ZADD delayed_jobs <execute_at_timestamp> <job_payload>
  
  Scheduler poll (every second):
    ZRANGEBYSCORE delayed_jobs -inf <now> LIMIT 10  → get due jobs
    ZREM delayed_jobs <job_ids>                      → atomically remove
    RPUSH job_queue <job_payloads>                   → move to execution queue
    
  Job becomes available exactly when its timestamp passes.
```

---

## Deep Dive: Clock Drift and Missed Executions

**Clock drift**: If the scheduler's clock is 5 minutes behind wall clock, jobs scheduled for 9:00 don't execute until 9:05. For billing, a 5-minute drift means some customers are charged 5 minutes late — likely acceptable. For market open trades — catastrophic.

**Fix**: NTP synchronisation on all machines. Verify with:
```bash
timedatectl status | grep "System clock synchronized"
chronyc tracking | grep "System time"
```

**Missed execution**: Scheduler is down from 8:55am to 9:05am. The 9:00am job is missed entirely.

**Fix options**:
1. **Catch-up mode**: on recovery, run any jobs missed during downtime (requires idempotency)
2. **Skip mode**: missed jobs are skipped, log an alert (acceptable for periodic aggregations)
3. **Overlap check**: before running a job, check if it's already been run for this period

```sql
-- Overlap check:
INSERT INTO job_executions (job_id, scheduled_for, status)
VALUES ('daily-report', '2024-01-15', 'RUNNING')
ON CONFLICT (job_id, scheduled_for) DO NOTHING;
-- 0 rows inserted → already running or completed. Skip.
-- 1 row inserted → you're the runner. Proceed.
```

---

## Canonical Problem: Email Campaign Scheduler

### Requirements
- Marketing team creates campaigns: "send to all users who signed up in 2023, at 9am their local time"
- 50M users across 400+ timezones
- Each email takes ~100ms to send via SES
- Must track: sent, opened, bounced, unsubscribed

### Architecture

```text
T-24h: Campaign created, audience computed
  → batch job: SELECT user_id, email, timezone FROM users WHERE condition
  → writes 50M recipient rows to: campaign_recipients table
  → status: PENDING

T-1h: Scheduler (per-timezone batches)
  For each timezone offset:
    When local time = 9:00am:
      SELECT user_id, email FROM campaign_recipients
      WHERE campaign_id = X AND timezone_offset = Y AND status = 'PENDING'
      LIMIT 10000
    → enqueue to: email-delivery queue (Kafka)
    → UPDATE status = 'QUEUED'

Email workers (horizontal pool):
  Consume from email-delivery queue
  Call SES: send email
  → Success: UPDATE status = 'SENT', sent_at = NOW()
  → Bounce: UPDATE status = 'BOUNCED'
  → Failure: retry up to 3 times → DLQ

Tracking:
  Each email contains a unique tracking pixel URL: /track/{recipient_id}/open
  When loaded: UPDATE status = 'OPENED', opened_at = NOW()
  Each link wrapped with: /track/{recipient_id}/click?url={encoded_url}
```

**The 50M email throughput problem**: SES limit: ~14 emails/second/account by default (increased with approval to ~millions/sec). At 14/sec: 50M emails = 41 days. Solution: request SES sending limit increase; use multiple SES regions; use a dedicated email provider (SendGrid, Mailgun) built for bulk.

---

*End of Pattern 11: Scheduling Systems*

---

## Evolution Path

```text
Stage 1: Single-Server Cron (0–100 jobs/day)
  crontab on one server. Simple. Zero infrastructure.
  Breaking point: server fails → jobs missed with no detection.
  Signal: job missed without alerting; manual discovery.

Stage 2: DB-Backed Scheduler + Leader Lock (100–10K jobs/day)
  Multiple server replicas; one holds leader lock (Redis SET NX).
  Only leader fires jobs. Followers are hot standby.
  Breaking point: complex job definitions; no retry/timeout; versioning is manual.
  Signal: job logic becomes complex enough that cron expressions can't express it.

Stage 3: Delayed Queue (Redis ZSET or SQS delay) (10K–1M jobs/day)
  Jobs stored with execute_at timestamp. Scheduler polls for due jobs.
  Breaking point: scheduler poll becomes DB bottleneck; or need for exactly-once at high rate.
  Signal: poll latency > 1s; scheduler CPU dominates.

Stage 4: Managed Scheduler (1M+ jobs/day, complex requirements)
  AWS EventBridge Scheduler: serverless, exactly-once, flexible recurrence.
  Temporal: workflows with scheduled starts, retries, versioning.
  Breaking point: none at this stage for most systems.
```

---

## Incident Response

**Symptom**: Scheduled job missed / job ran twice / billing job didn't fire

```text
Diagnose:
  1. Check job_executions table:
     SELECT * FROM job_executions WHERE job_id=? AND scheduled_for=? 
     → No row: job was never picked up.
     → Two rows with status=RUNNING: job ran twice (missed deduplication).
  2. Check leader lock:
     redis-cli GET scheduler:leader
     → If empty: no leader (TTL expired, new election in progress).
     → If stale leader: leader node crashed but TTL hasn't expired.
  3. Check scheduler pod health:
     kubectl get pods -n scheduler

Mitigation:
  Missed job: manually trigger the job; verify idempotency; add catch-up logic.
  Job ran twice: identify duplicate rows; check deduplication logic; verify ON CONFLICT.
  No leader: Redis key expired; next poll cycle will elect new leader (up to TTL seconds).
  
Recovery: verify idempotency of all affected jobs before re-running.
```

---

## Design Review Checklist

```text
□ Job execution is idempotent? (safe to run twice — always)
□ Overlap prevention: INSERT ON CONFLICT DO NOTHING on (job_id, scheduled_for)?
□ Missed job policy: catch-up (rerun) or skip? Documented per job type?
□ Long-running jobs: heartbeat to extend lock TTL?
□ Clock drift mitigation: NTP synchronised on all scheduler nodes?
□ Alert: job missed its window by > N minutes?
□ Alert: job failure rate > threshold?
□ Alert: same job ran twice in same window?
□ Job timeout: what happens if a job never completes? Stuck job alert?
□ Scheduler leader: TTL set appropriately (> max GC pause, < max acceptable miss window)?
```

---

## Threat Model

```text
T1: Scheduled Job Injection (Elevation of Privilege)
  Attack: attacker schedules jobs with elevated permissions or targeting other users.
  Mitigation: job queue accessible only from authorised services (mTLS or IAM role);
              job payload validated and signed at creation;
              jobs execute with minimum required permissions (not admin).

T2: Job Timing Attack (Information Disclosure)
  Attack: predict when subscription billing runs; time fraudulent account creation to avoid billing.
  Mitigation: add jitter to scheduled times; don't publish exact execution times.

T3: Cron Expression Injection (Tampering)
  Attack: inject malicious cron expression via admin API to run jobs at dangerous frequency.
  Mitigation: validate cron expressions against allowlist of frequencies;
              minimum interval enforced (e.g., no more than once per minute);
              admin job creation requires two-party approval.
```

---


---

## Migration Strategy

**From single-server cron to distributed scheduler with leader election**

```text
Step 1: Deploy new distributed scheduler alongside existing cron.
        Duplicate all cron jobs in the new scheduler (disabled by default).

Step 2: Enable new scheduler for one low-risk job. Disable that job in cron.
        Validate: job runs at correct time; exactly once; idempotent.

Step 3: Migrate remaining jobs one at a time (low-risk → high-risk order).
        For each: enable in new scheduler → validate for 7 days → disable in cron.

Step 4: Decommission cron server after all jobs migrated.

Rollback: re-enable any job in cron at any point. Both systems can run concurrently.
Critical: ensure each job is only enabled in ONE system at a time (not both).
```

---

## Organisational Ownership

```text
Platform team owns: scheduler infrastructure, leader election mechanism, job queue.
  On-call: scheduler leader failure, job missed alerts.

Application teams own: job logic, idempotency, expected duration, failure handling.
  On-call: their job's failure rate, correctness issues.

FinOps: reviews scheduled job resource usage; flags jobs consuming excessive resources.
```

---

*End of Pattern 11: Scheduling Systems*
# Pattern 12: Distributed Transactions

> "You cannot have a database transaction that spans two databases. You can have a saga that pretends to."

---

## The Shape

A business operation must update state in multiple independent services or databases atomically. Either all updates succeed, or none take effect. Standard ACID transactions don't work across service boundaries.

---

## When You See This Pattern

- Payment: debit account A + credit account B (different bank systems)
- Order: reserve inventory + create order + charge payment (3 services)
- IAM: create identity + provision all external systems + assign roles

*(This pattern overlaps significantly with Pattern 3: Multi-Step Processes. The distinction: Pattern 3 focuses on workflow design; Pattern 12 focuses on the consistency mechanisms.)*

---

## The Options

### Two-Phase Commit (2PC)

A coordinator asks all participants to "prepare" (vote yes/no). If all vote yes, the coordinator tells all to "commit." If any vote no, all are told to "rollback."

```text
Phase 1 (Prepare):
  Coordinator → Service A: "Prepare: debit $100 from account 123"
  Coordinator → Service B: "Prepare: credit $100 to account 456"
  Service A: locks row, writes to WAL, responds: READY
  Service B: locks row, writes to WAL, responds: READY

Phase 2 (Commit):
  Coordinator → Service A: "Commit"
  Coordinator → Service B: "Commit"
  Both apply the changes and release locks.
```

**The blocking problem**: If the coordinator crashes after Phase 1 but before Phase 2, both services are locked waiting for instruction. They cannot proceed until the coordinator recovers (or times out). The system is **blocking**.

**When to use 2PC**: Within a single database cluster (PostgreSQL can 2PC across schemas). Within a single organisation where all services support XA protocol. Avoid for microservices across teams — the coupling is too tight.

---

### Saga Pattern (Preferred for Microservices)

Covered in detail in Pattern 3. The key design principles for distributed consistency:

```text
Compensating transactions must:
  1. Be idempotent (safe to run multiple times)
  2. Be retryable (will eventually succeed, or alert for manual intervention)
  3. Not require knowing the outcome of other concurrent sagas

Saga guarantees:
  - NOT ACID atomicity (intermediate states are visible)
  - Eventual consistency (system reaches consistent state after all compensations run)
  - Isolation: partial — other transactions can see intermediate saga state
```

**The saga isolation problem**: During a payment saga, the account is debited (step 1) before the merchant is credited (step 3). Another saga might read the account balance during this window and see a debited-but-not-yet-settled state. This is acceptable for most business processes; design read models to be aware of in-flight saga states.

---

### Outbox + CDC for Cross-Service Consistency

The most production-reliable pattern for ensuring two services stay consistent without distributed locks:

```text
Service A (Order Service):
  BEGIN;
    INSERT INTO orders (id, status, user_id, ...) VALUES (...);
    INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', {...});
  COMMIT;
  
CDC (Debezium) watches outbox table → publishes to Kafka → Service B consumes

Service B (Inventory Service):
  Consumes OrderCreated:
    UPDATE inventory SET reserved = reserved + 1 WHERE product_id = ?
    (idempotent: check if already reserved for this order_id first)
```

The key property: if Service A's transaction commits, the event will be published (at-least-once). Service B will eventually process it. Services are eventually consistent. No distributed lock, no 2PC.

---

## Canonical Problem: Cross-Bank Payment

### Requirements
- Debit user's bank account (Bank A system)
- Credit merchant's bank account (Bank B system)  
- Exactly once. No double debit. No double credit.
- Must handle: Bank B unavailable, Bank A timeout, network failures

### Architecture

```text
Payment Orchestrator:

State: payment_id, status, debit_idempotency_key, credit_idempotency_key

Step 1: Create payment record (local DB, INITIATED)
Step 2: Call Bank A API: debit user account
  Idempotency key: payment_id + "-debit"
  If timeout: retry with same key (Bank A deduplicates)
  → DEBITED

Step 3: Call Bank B API: credit merchant
  Idempotency key: payment_id + "-credit"
  If timeout: retry with same key (Bank B deduplicates)
  → CREDITED

Step 4: Mark SETTLED

Failure at Step 3 (Bank B unavailable):
  Retry 3 times with backoff
  If still failing after 15 minutes:
    Compensate: Call Bank A API: refund debit
    Idempotency key: payment_id + "-refund"
    Mark payment REFUNDED
```

**The critical insight**: Idempotency keys at the external API call level prevent double charges even with aggressive retries. The key format (`payment_id + "-debit"`) ensures that even if the orchestrator retries Step 2 10 times, Bank A applies the debit exactly once.

---

## Interview Calibration

**Staff-level question**: "What happens if the payment orchestrator crashes after debiting Bank A but before crediting Bank B?"

**Staff-level answer**: "The orchestrator's state machine persists every transition to the database before executing the step. On restart, it reads the last known state (DEBITED), skips Step 2 (already done), and retries Step 3. The idempotency key on Step 3 ensures Bank B doesn't double-credit even if it received a partial request before the crash. If Step 3 permanently fails, the compensation (refund) is triggered with its own idempotency key."

---

*End of Pattern 12: Distributed Transactions*

---

## Evolution Path

```text
Stage 1: Single Database Transaction (same DB, same team)
  All operations in one ACID transaction. Simple. Correct.
  Breaking point: operations span different databases or different service teams.
  Signal: engineers adding cross-service HTTP calls inside DB transactions.

Stage 2: 2PC — Two-Phase Commit (multiple DBs, same organisation, XA support)
  Coordinator prepares all participants, then commits all atomically.
  Breaking point: coordinator SPOF; blocking failure on coordinator crash.
  Signal: coordinator crashes → participants locked indefinitely; system stuck.

Stage 3: Saga with Choreography (multiple services, simple linear flows)
  Each service reacts to events; compensation events for rollback.
  Breaking point: hard to track saga state; "why is this stuck?" takes 30 minutes to debug.
  Signal: mean time to diagnose stuck saga > 15 minutes.

Stage 4: Saga with Orchestration (complex flows, compliance requirements)
  Central orchestrator tracks all steps; visibility into any saga's current state.
  Breaking point: orchestrator DB becomes write bottleneck at very high saga rate.
  Signal: orchestrator DB write latency > 50ms at peak.

Stage 5: Outbox + CDC as Coordination Layer
  No explicit saga orchestrator. DB is the single source of truth.
  Outbox table guarantees event delivery. CDC propagates changes.
  Appropriate for systems where eventual consistency is acceptable.
```

---

## Incident Response

**Symptom**: Payment stuck / compensation not completing / saga in COMPENSATING for > 1 hour

```text
Diagnose:
  1. SELECT id, current_step, status, updated_at FROM sagas
     WHERE status IN ('IN_PROGRESS', 'COMPENSATING')
     AND updated_at < NOW() - INTERVAL '30 minutes';
  2. For each stuck saga: check the step's downstream service health.
  3. Check: has compensation been attempted? How many times?
     SELECT attempt_count, error_message FROM saga_step_executions
     WHERE saga_id=? AND step_name='compensate_debit' ORDER BY created_at DESC LIMIT 5;

Mitigation:
  A. Stuck in COMPENSATING — downstream unavailable:
     → Add to manual_review_queue; alert finance team.
     → Document: which sagas, which step, what manual action required.
  B. Stuck in IN_PROGRESS — orchestrator recovered but not resuming:
     → Manually trigger saga resume via admin API.
     → If no admin API exists: add one before next on-call shift.
  C. Coordinator DB down:
     → Sagas cannot advance. Queue them. Resume on DB recovery.
     → ETA for DB recovery → communicate to stakeholders.

Recovery: all stuck sagas either completed or moved to manual_review_queue.
```

---

## Design Review Checklist

```text
□ 2PC avoided for cross-team/cross-company services?
□ Every step call idempotent? Idempotency key derived deterministically?
□ Every reversible step has compensating action? Compensation is idempotent?
□ Irreversible steps placed last in the saga?
□ Saga state persisted to DB before each external call?
□ Stuck saga detector: alert if saga in same state for > 2× expected duration?
□ Manual review queue for permanently stuck compensations?
□ Admin API to inspect, force-advance, or cancel any saga?
□ Saga versioning strategy: what happens to in-flight sagas during code deploy?
□ Outbox pattern used to guarantee event delivery alongside DB writes?
□ Alert: compensation rate > N%? (rising compensation = upstream failure signal)
□ Alert: saga completion rate drop > 5%?
```

---

## Threat Model

```text
T1: Saga Replay Attack (Tampering)
  Attack: replay saga initiation event to trigger second execution (double payment).
  Mitigation: idempotency key on saga initiation (client-generated UUID);
              saga creation: INSERT ON CONFLICT DO NOTHING on idempotency_key.

T2: Compensation Abuse (Elevation of Privilege)
  Attack: deliberately fail step N after benefit from step N-1 to trigger refund.
  Mitigation: compensation requires same auth as original action;
              rate limit compensations per user per day;
              anomaly detection: flag accounts with high compensation rate.

T3: Orchestrator Takeover (Spoofing)
  Attack: impersonate orchestrator to send fake step-complete signals.
  Mitigation: step completions authenticated via mTLS certificate;
              orchestrator validates sender identity before advancing state.
```

---


---

## Migration Strategy

**From 2PC to Saga orchestration**

```text
Step 1: Identify all cross-service operations currently using 2PC or synchronous chains.
        Document: steps, participants, compensation logic (what undoes each step).

Step 2: Build saga orchestrator with state machine matching current behaviour.
        Run in shadow mode: orchestrator tracks but doesn't control execution.
        Validate: orchestrator's simulated state matches actual system state.

Step 3: Enable orchestrator for one low-risk saga type (e.g., user enrollment).
        Monitor: completion rate, stuck saga count, compensation rate.

Step 4: Migrate remaining saga types one at a time.
        High-value sagas (payment) last, after lower-risk types have proven the pattern.

Step 5: Remove 2PC coordinator. All distributed operations use Saga.

Rollback: re-enable synchronous/2PC path via feature flag. Orchestrator can coexist.
```

---

## Organisational Ownership

```text
Platform / Workflow team owns (if using Temporal/Conductor): workflow engine infrastructure.
  On-call: orchestrator unavailability, stuck saga alerts.

Domain teams own: saga definitions, step implementations, compensation logic.
  On-call: their saga's completion rate, compensation failures.

Finance / Compliance: approves compensation logic for financial transactions.
  Must sign off: "this compensation correctly reverses the financial effect."
```

---

*End of Pattern 12: Distributed Transactions*
# Pattern 13: Multi-Region Systems

> "Latency is physics. Light takes 67ms to travel from Mumbai to London. You cannot engineer around the speed of light; you can only move the data closer."

---

## The Shape

A system serves users across multiple geographic regions. Users in each region expect low latency. The system must survive a region-wide failure. Some data must stay within specific geographic boundaries (compliance).

---

## When You See This Pattern

- Global payment platforms
- Global IAM / SSO systems
- Consumer apps with users on multiple continents
- Systems with data residency requirements (GDPR, RBI data localisation)

---

## The Core Tension

```text
Strong consistency ←——————————————→ Low latency
(same data everywhere)               (read from local region)

Single source of truth ←————————————→ Regional autonomy
(simple, correct)                      (available during partition)

Data anywhere ←—————————————————————→ Data localisation
(fastest routing)                       (compliance requirement)
```

---

## Building Blocks

### Active-Passive

One region is active (handles all writes). Other regions are passive (read replicas).

```text
Primary: Mumbai (writes + reads)
Replicas: Singapore, London, New York (reads only, replicated from Mumbai)

Normal operation:
  Mumbai users: read/write from Mumbai (5ms)
  Singapore users: reads from Singapore replica (5ms), writes to Mumbai (80ms)
  
Failure (Mumbai down):
  Promote Singapore to primary
  Other regions redirect to Singapore
  RTO: 5-30 minutes (manual or automated failover)
  RPO: seconds (async replication lag at time of failure)
```

**Best for**: Systems where consistency is critical and the write latency cost is acceptable. Banks often use this: writes always go to the primary; a 80ms write latency is fine for a payment that processes in 2 seconds.

---

### Active-Active

Multiple regions each accept writes. Data is replicated between them.

```text
Mumbai:    accepts writes from India users
Singapore: accepts writes from SEA users
London:    accepts writes from EU users

Cross-region replication: Mumbai ↔ Singapore ↔ London

A user in India writes to Mumbai.
Another user in SEA might read their data from Singapore.
Replication lag: 100-500ms cross-region.
```

**The conflict problem**: User A (Mumbai) and User B (Singapore) simultaneously update the same record. Both writes succeed locally. Replication brings conflicting versions together.

**Conflict resolution strategies**:
- **Last Write Wins (LWW)**: timestamp decides winner. Simple. Clock skew causes bugs.
- **Application-defined merge**: app code decides how to merge conflicts. Complex but correct.
- **CRDTs**: data structures that merge automatically (counters, sets). Not all data fits CRDTs.
- **Avoid conflicts**: route all writes for a given entity to its "home region" (e.g., all updates to user X go to Mumbai because user X is an India user). Other regions read-only for X.

---

### Geo-Routing

Route users to the nearest healthy region automatically.

```text
DNS + Health Checks:
  api.example.com → Route 53 latency-based routing
    → Mumbai endpoint (if healthy and user is in India)
    → Singapore endpoint (if healthy and user is in SEA)
    → London endpoint (if healthy and user is in EU)

On region failure:
  Health check fails → Route 53 automatically removes unhealthy endpoint
  All traffic re-routes to next-nearest healthy region
  DNS TTL: 30–60 seconds (how quickly clients follow the change)
```

---

## Canonical Problem: Multi-Region IAM Platform

### Requirements
- Users across 5 regions: India, SEA, EU, US-East, US-West
- Login latency: p99 < 200ms
- Regulatory: Indian users' PII must stay in India (RBI data localisation)
- Availability: 99.99% (4.38 minutes downtime/month)

### Architecture

```text
Region: India (Mumbai) — PRIMARY for Indian users
Region: Singapore — PRIMARY for SEA users
Region: London — PRIMARY for EU users (GDPR data residency)

Data Classification:

User PII (name, phone, email):
  → India users: stored ONLY in Mumbai (regulatory requirement)
  → SEA users: stored ONLY in Singapore
  → EU users: stored ONLY in London

Global data (role definitions, system config, JWKS):
  → Replicated to ALL regions (read-only replicas)
  → Writes go to a designated primary (e.g., Mumbai for IAM config)
  → Consistency: eventual (minutes), acceptable for config

Session tokens:
  → Issued per-region, verifiable in all regions
  → JWT: signed with region's private key; all regions have JWKS to verify
  → User logging in from India gets a token signed by Mumbai's key
  → User makes API call to Singapore region: Singapore verifies using Mumbai's JWKS

Login flow (India user, logging in from India):
  DNS → Mumbai endpoint
  Authenticate against Mumbai's user store → issue JWT
  JWT signed with Mumbai's key (kid: mumbai-2024-01)
  User uses JWT to call any region's APIs
  Any region: verify JWT using JWKS (includes Mumbai's public key)
```

**Cross-region token verification without round-trips**: Each region caches the JWKS from all other regions (TTL=5 minutes). Token verification is local — no network call to the issuing region. This is why JWT is used rather than opaque tokens: JWT is self-contained and verifiable with a public key.

**The data sovereignty boundary**: When a Singapore user contacts the India region (e.g., they travel to India), the India region cannot serve their account data (it's in Singapore). Options:
- Redirect the user to the correct region
- Cross-region proxy request (Singapore → Mumbai API call to serve Singapore user who is in India)
- Accept the 80ms cross-region latency for "roaming" users

---

## Deep Dive: Conflict Resolution in Active-Active

**Scenario**: User changes their phone number. They're in India, using the Mumbai region. Simultaneously, their bank's admin changes the same phone number for the same user via the Singapore admin panel.

**LWW (Last Write Wins)**:
```text
Mumbai write at T=10:00:00.100: phone="+91-98765"
Singapore write at T=10:00:00.095: phone="+65-8765"

Mumbai write has later timestamp → wins.
Phone is "+91-98765".
Singapore admin's update silently discarded.
```
Risk: if timestamps are skewed (NTP drift), wrong value wins.

**Application merge**:
```text
Conflict detected: two versions with same vector clock base.
Application rule: "most recently updated by the user (not admin) wins"
Or: "alert the admin that a conflict occurred; display both versions for human resolution"
```

**Conflict avoidance (recommended)**:
```text
User profile is "owned by" the user's home region.
India user's profile: only Mumbai accepts writes. Singapore reads it (via replication).
No conflict possible: there is only one writer.
Singapore admin who wants to update India user: must do so via Mumbai API.
```

This is the simplest solution and the right default for IAM systems where data ownership is clear.

---

## Interview Calibration

**Staff-level additions**:
- Immediately asks about data residency requirements (determines architecture)
- Separates "user data" from "global config data" — different consistency requirements
- Proposes the "home region" ownership model to avoid conflicts
- Addresses the JWKS distribution problem for JWT verification across regions
- Considers: "what does the UX look like when a user's home region is down? Can they log in at all?"

---

*End of Pattern 13: Multi-Region Systems*

---

## Evolution Path

```text
Stage 1: Single Region (0–10M users in one geography)
  All infrastructure in one region. Simple. Fast for local users.
  Breaking point: users in other geographies complain about latency;
                  or: single-region failure causes total outage.
  Signal: p99 latency from non-primary regions > 300ms; SLA violation on region failure.

Stage 2: CDN (static content served globally)
  Edge nodes serve static assets from nearest location.
  Dynamic requests still go to origin region.
  Breaking point: dynamic API latency still high for distant users.
  Signal: CDN hit ratio high, but API latency from distant regions still > SLO.

Stage 3: Active-Passive (single primary, DR replica in second region)
  Primary handles all traffic. DR replica is warm standby.
  Failover: promote DR replica; update DNS; ~5–30 minute RTO.
  Breaking point: failover RTO > SLA; or distant users' latency > SLO in normal operation.
  Signal: DR drill shows failover takes > 15 minutes; user latency from distant regions violates SLO.

Stage 4: Active-Active (traffic served from multiple regions simultaneously)
  Each region handles its users' requests. Cross-region replication synchronises data.
  Write conflicts: resolved by home-region ownership (each user writes to their home region).
  Breaking point: this handles essentially any global scale.
  Signal: never the technical bottleneck; cost and operational complexity are the limits.
```

---

## Incident Response

**Symptom**: Region-wide latency spike / DNS failover needed / data residency alert

```text
Diagnose:
  1. Check: is this one region or all regions?
     Dashboard: p99 latency by region.
     → One region: regional issue (infra, ISP, AZ failure).
     → All regions: global upstream (DNS, CDN, BGP).
  2. For single-region failure:
     → Check cloud provider status page.
     → Check AZ health in affected region.
  3. Replication lag to DR region:
     SELECT now() - pg_last_xact_replay_timestamp() AS lag FROM pg_stat_replication;

Mitigation:
  A. Region degraded (not down):
     → Increase circuit breaker sensitivity; shed non-critical traffic.
     → Scale up in the healthy region to absorb redirected traffic.
  B. Region fully down:
     → DNS failover: update Route 53 health check; traffic routes to DR region.
     → Promote DR replica to primary (Patroni / manual).
     → Communicate to stakeholders: estimated data loss (= replication lag at failure time).
  C. Data residency violation detected:
     → Immediately stop cross-region data transfer for affected tenant.
     → Alert compliance team; initiate incident report.

Recovery: DNS propagated (check TTL); DR region handling traffic; latency returning to SLO.
```

---

## Design Review Checklist

```text
□ Data residency requirements documented per entity type?
□ Each entity type has a defined "home region" (single write authority)?
□ Cross-region replication lag monitored? Alert threshold defined?
□ Failover runbook documented and drilled quarterly?
□ RTO and RPO defined and validated in DR drills?
□ DNS TTL set appropriately for desired failover speed?
□ Users in non-primary regions: acceptable write latency to home region documented?
□ Cross-region data transfer cost included in monthly budget?
□ JWKS / global config replication to all regions for self-sufficient token validation?
□ Alert: replication lag > 10 seconds?
□ Alert: cross-region request rate spike (unexpected data flowing between regions)?
□ Active-active: conflict detection and resolution strategy implemented and tested?
```

---

## Threat Model

```text
T1: Data Residency Violation (Information Disclosure / Compliance)
  Attack (misconfiguration): user data flows to wrong region (e.g., Indian PII to EU servers).
  Mitigation: tenant_id → region mapping enforced at API layer;
              cross-region data transfer blocked by network ACLs for PII data;
              automated audit: weekly scan for data in wrong region.

T2: Region Takeover via DNS Hijack (Spoofing)
  Attack: compromise DNS to redirect traffic to attacker-controlled servers.
  Mitigation: DNSSEC enabled; registrar account 2FA;
              certificate pinning in mobile clients (partial mitigation);
              monitor DNS records for unexpected changes (alert within 5 minutes).

T3: Split-Brain Exploitation (Tampering)
  Attack: trigger a network partition; write to both sides; exploit data divergence.
  Mitigation: home-region ownership prevents conflicting writes;
              quorum required for leader election (prevents split-brain);
              fencing tokens on all storage writes.
```

---


---

## Migration Strategy

**From single-region to active-passive multi-region**

```text
Step 1: Establish DR region with read replica of primary DB.
        No user traffic to DR region yet. Validate: replication lag < 5s sustained.

Step 2: Deploy application tier in DR region (minimum capacity, passive).
        DR region: can serve traffic but receives none.

Step 3: DR drill (quarterly): failover all traffic to DR region for 30 minutes.
        Validate: RTO < target; RPO acceptable; all critical functionality works.

Step 4: Enable geo-routing for read traffic to DR region (reads only, no writes).
        Serves users in DR region geography with lower latency.

Step 5 (optional): promote to active-active for a subset of traffic.
        Start with read-only APIs. Then stateless APIs. Write APIs last.

Rollback at any step: all writes remain in primary region.
```

---

## Organisational Ownership

```text
Infrastructure / SRE team owns: multi-region deployment, DNS failover, replication monitoring.
  On-call: region failure, replication lag spike, failover trigger.

Platform / DB team owns: cross-region DB replication configuration and health.
  On-call: replication slot failure, lag exceeding threshold.

Compliance / Legal: approves data residency decisions for each tenant/data type.
  Must sign off: "this data is allowed to be in region X."

Application teams: must test their service works correctly in DR region during drills.
```

---

*End of Pattern 13: Multi-Region Systems*
# Pattern 14: Event-Driven Systems

> "Events are facts. They happened. They cannot be undone. Build your system around facts, not instructions."

---

## The Shape

Services communicate by publishing and subscribing to events (things that happened) rather than by direct API calls (instructions to do something). This decouples producers from consumers and enables many consumers to react to the same event independently.

---

## When You See This Pattern

- Audit systems (every state change in IAM produces an audit event)
- Analytics pipelines (user actions → Kafka → analytics DB)
- Data synchronisation (order placed → update inventory, loyalty points, fraud model)
- Workflow triggers (payment received → trigger shipment, receipt email, accounting entry)
- CQRS read model updates (write to DB → event → update read model)

---

## Building Blocks

### Events vs Commands

```text
Command: an instruction to do something
  → "ProcessPayment" — I'm telling you to process this payment
  → Can be rejected
  → One receiver (point-to-point)
  → Represents an intent

Event: a fact that something happened
  → "PaymentProcessed" — this payment has been processed (past tense)
  → Cannot be rejected (it already happened)
  → Many receivers (fan-out)
  → Represents history
```

**Why this matters for system design**: An event-driven system treats Kafka topics as an **event log** — an append-only ledger of what happened. Any service can subscribe, process at its own pace, replay from any point, and derive its own view of the world.

---

### Event Ordering

Within a Kafka partition, events are strictly ordered. Across partitions, they are not.

```text
Partition by entity ID:
  All events for user-123 → Partition 0 (hash(123) % partitions)
  All events for user-456 → Partition 1

Guarantee: events for user-123 arrive in order.
No guarantee: relative order of user-123 and user-456 events.

This is usually correct: a consumer processing user-123's events (e.g., their audit log)
doesn't need to know the relative order of user-456's events.
```

**When you need global ordering**: You usually don't. If you think you do, re-examine the requirement. Global ordering requires a single partition → single bottleneck → not scalable.

---

### Event Replay

Kafka retains events for a configurable period (default: 7 days; can be infinite). This enables:

```text
New service joins the system 6 months later:
  Set consumer group offset to: beginning of topic
  Replay all 6 months of events
  Build up state from scratch
  
  No need to query other services for historical data.
  The event log IS the history.

Service bug discovered: audit log had a bug for 2 weeks:
  Fix the bug
  Reset consumer offset to 2 weeks ago
  Replay events through fixed code
  Audit log is rebuilt correctly
```

---

### Event Schema Evolution

Events are a public contract. Consumers depend on their structure. Changing them breaks consumers.

```text
Safe changes:
  → Add new optional field (consumers that don't know it ignore it)
  → Add new event type (consumers subscribe to what they care about)

Breaking changes (avoid):
  → Remove a field consumers depend on
  → Change a field's type (string → integer)
  → Rename a field

Solution: Schema Registry (Confluent Schema Registry with Avro)
  → Enforces backward/forward compatibility at publish time
  → Producers cannot publish schemas that break existing consumers
  → Consumers can process events from producers on different schema versions
```

---

## Canonical Problem: IAM Audit System

### Requirements
- Every IAM operation (login, role change, password change, account lock) must be recorded
- Audit records must be immutable (cannot be deleted or modified)
- Query: "all operations by admin-X in the last 30 days" — within 5 seconds
- Retention: 7 years (regulatory)
- Volume: 1M audit events/day

### Architecture

```text
Event Sources:
  Identity Service:    emits IdentityProvisioned, IdentitySuspended
  Auth Service:        emits LoginSucceeded, LoginFailed, PasswordChanged
  Authz Service:       emits RoleGranted, RoleRevoked
  Session Service:     emits SessionCreated, SessionInvalidated

All use Outbox Pattern:
  DB transaction includes: audit event written to outbox table
  Debezium CDC → Kafka topic: iam.audit-events

Audit Consumer:
  Consumes iam.audit-events
  Writes to: ClickHouse (columnar, fast aggregation queries)
    + S3 Parquet (long-term archive, queryable via Athena)

Audit Query API:
  GET /audit?actor=admin-X&from=2024-01-01&to=2024-01-31
  → Query ClickHouse (fast for recent data)
  → Or Athena (for data >90 days old, in S3 Parquet)

Immutability guarantee:
  Events are written to Kafka (append-only)
  Kafka → ClickHouse: append-only inserts
  No UPDATE or DELETE operations on audit data
  S3 Parquet: write-once files
  DB-level: audit_events table has no UPDATE/DELETE permissions for any role
```

**The schema evolution story**: Six months in, the team realises they need to add `ip_address` to all auth events. Old events don't have it (it wasn't captured). New events include it. Consumers must handle both:

```python
class LoginSucceededEvent:
    user_id: str
    timestamp: datetime
    ip_address: Optional[str]  # None for events before the schema change
    # Forward-compatible: new field is optional
```

---

## Deep Dive: Exactly-Once Processing

Kafka guarantees at-least-once delivery. The audit consumer might process the same event twice (producer retry, consumer restart before committing offset).

```text
Duplicate event received:
  Event: { event_id: "evt-789", type: "RoleGranted", ... }
  
Audit consumer:
  First delivery: INSERT INTO audit_events (event_id, ...) VALUES ("evt-789", ...)
  Second delivery: INSERT INTO audit_events (event_id, ...) VALUES ("evt-789", ...)
  → Duplicate record!

Fix: ON CONFLICT DO NOTHING with unique constraint on event_id

CREATE UNIQUE INDEX ON audit_events(event_id);

INSERT INTO audit_events (event_id, type, ...)
VALUES ("evt-789", "RoleGranted", ...)
ON CONFLICT (event_id) DO NOTHING;
-- Second insert: silently skipped. Exactly-once effect.
```

---

## Interview Calibration

**Staff-level additions**:
- Distinguishes events (facts) from commands (instructions) — this signals deep understanding
- Addresses event ordering per entity, not global ordering
- Treats the event log as the system of record: "we can rebuild any downstream system by replaying the event log"
- Addresses the schema registry requirement immediately: "without a schema registry, producers can break consumers silently"
- Quantifies retention: "7 years at 1M events/day with 500 bytes/event = ~1.3 TB/year; use Parquet compression (~0.3× ratio) → 400GB/year; 7 years = 2.8 TB total, fits comfortably in S3"

---

*End of Pattern 14: Event-Driven Systems*

---

## Evolution Path

```text
Stage 1: Direct HTTP Calls Between Services (0–10 services)
  Service A calls Service B synchronously. Simple. Tight coupling.
  Breaking point: Service B being slow makes Service A slow (cascading).
  Signal: service-to-service latency correlations; cascading failure incidents.

Stage 2: Message Queue (async decoupling) (10–50 services)
  Service A publishes to queue; Service B consumes at its own pace.
  Breaking point: many services need the same event; point-to-point queues multiply.
  Signal: N×M queue connections for N producers and M consumers.

Stage 3: Event Broker — Kafka (50+ services, event streaming)
  Services publish to topics; multiple consumer groups subscribe independently.
  Event log is durable and replayable.
  Breaking point: schema evolution breaks consumers; event ordering complex.
  Signal: schema change breaks downstream consumers; incidents from silent incompatibility.

Stage 4: Schema Registry + Event Catalog (mature event-driven system)
  All event schemas registered; backward compatibility enforced at publish time.
  Internal event catalog: teams discover events to subscribe to.
  Breaking point: never a technical limit; organisational complexity is the challenge.
```

---

## Incident Response

**Symptom**: Consumer lag growing / events not processing / audit events missing

```text
Diagnose:
  1. kafka-consumer-groups.sh --describe --group <group-name>
     → Per-partition lag. Growing on all partitions? Consumer too slow.
     → Growing on one partition? Hot partition or poison message.
  2. Consumer pod health: kubectl get pods -n events | grep consumer
     → CrashLoopBackOff: poison message causing consumer crash.
  3. Schema registry: is it reachable? Producer schema compatibility check failing?

Mitigation:
  A. Consumer too slow: kubectl scale deployment/event-consumer --replicas=+N
     (max = partition count; cannot add more consumers than partitions)
  B. Poison message: identify offset; skip it; move to DLQ:
     kafka-consumer-groups.sh --reset-offsets --topic T --partition P --to-offset N+1
  C. Schema registry down: producers fall back to last cached schema;
     consumers continue with cached schema; lag may grow.
     Priority: restore schema registry.

Recovery: consumer lag decreasing; no pods in CrashLoopBackOff.
```

---

## Design Review Checklist

```text
□ Schema registry enforcing backward compatibility for all topics?
□ All consumers idempotent? (ON CONFLICT DO NOTHING on event_id)
□ Consumer lag is the primary health metric? Alert threshold defined?
□ DLQ configured for each consumer? DLQ monitored with alert?
□ Kafka retention long enough for consumer to catch up from failure? (min 7 days)
□ Partition key distributes load uniformly? Hot partition scenario considered?
□ Event replay tested from a new consumer group?
□ Schema evolution strategy documented? (add fields: backward-compatible; rename: not)
□ Outbox pattern used for all events that must be atomically consistent with DB writes?
□ Consumer group offsets backed up? (can restore consumer position after full failure)
□ Alert: under-replicated Kafka partitions?
□ Alert: consumer lag > N minutes worth of events?
```

---

## Threat Model

```text
T1: Event Injection (Tampering)
  Attack: attacker publishes events to Kafka topic impersonating a legitimate service.
  Mitigation: Kafka ACLs (SASL/SCRAM or mTLS per service certificate);
              each service can only write to its own topics (not others');
              event signature (HMAC) for high-value events.

T2: Consumer Isolation Failure (Information Disclosure)
  Attack: Consumer A reads events intended for Consumer B (different tenant).
  Mitigation: tenant_id in event payload; consumer validates tenant_id before processing;
              Kafka ACLs: consumer group can only read topics for its tenant.

T3: Event Replay Attack (Tampering)
  Attack: replay historical events to re-trigger business actions (double payment, double credit).
  Mitigation: all consumers idempotent (deduplicate by event_id);
              event timestamps validated (reject events older than 24 hours for financial events).
```

---


---

## Migration Strategy

**From synchronous API calls to event-driven communication**

```text
Step 1: Identify which synchronous calls are fire-and-forget (no response needed).
        These are the first candidates for events (lowest risk).

Step 2: Introduce event publishing alongside existing synchronous calls.
        Dual-path: both synchronous call AND event published for the same action.
        Consumers: read from events but take no action (shadow mode).

Step 3: Validate shadow consumers match synchronous behaviour for 1 week.

Step 4: Enable consumers (stop synchronous call; rely on event).
        Monitor: end-to-end latency (write to consumer completing), correctness.

Step 5: Remove synchronous fallback after 30 days stable.

Rollback: re-enable synchronous call; disable event consumer. Parallel run is safe.
```

---

## Organisational Ownership

```text
Data Platform team owns: Kafka cluster, schema registry, topic management.
  On-call: broker failure, under-replicated partitions, consumer lag alerts.

Application teams (producers) own: event schema design, outbox pattern implementation,
  partition key selection.
  On-call: their events not flowing (outbox processor failure).

Application teams (consumers) own: consumer idempotency, consumer lag, DLQ management.
  On-call: their consumer's lag, correctness of downstream effects.

Platform team arbitrates: schema compatibility disputes, topic naming conventions,
  consumer group access control.
```

---

*End of Pattern 14: Event-Driven Systems*
# Pattern 15: API Gateway Pattern

> "The API gateway is where your system meets the world. It is the last place to enforce consistency, security, and control before your service topology becomes the world's problem."

---

## The Shape

A single, managed entry point for all external traffic. It owns all cross-cutting concerns that would otherwise be duplicated across every service: authentication, rate limiting, routing, SSL termination, observability.

---

## When You See This Pattern

Every system with more than one external-facing service needs an API gateway. The pattern is asked in interviews via:
- "Design a microservices API gateway"
- "Design the authentication layer for a payment system"
- "How would you secure a platform with 50 internal services?"

---

## Building Blocks

### What the Gateway Owns

```text
Client
  │
  │ HTTPS
  ▼
┌─────────────────────────────────────────────────────────┐
│                     API Gateway                         │
│                                                         │
│  1. SSL/TLS Termination                                 │
│     (encrypt client↔gateway; internal traffic = HTTP)   │
│                                                         │
│  2. Authentication                                      │
│     (validate JWT; extract user identity; inject header) │
│                                                         │
│  3. Authorisation (coarse-grained)                      │
│     (does this token have any access to /payments/?)    │
│                                                         │
│  4. Rate Limiting                                       │
│     (per API key, per user, per endpoint)               │
│                                                         │
│  5. Path-Based Routing                                  │
│     (/payments/* → payment-service)                     │
│     (/users/* → user-service)                           │
│                                                         │
│  6. Request ID Injection                                │
│     (X-Request-ID header for distributed tracing)       │
│                                                         │
│  7. Logging / Metrics                                   │
│     (every request: method, path, status, latency, user) │
│                                                         │
│  8. Response Caching                                    │
│     (GET /products/123 with TTL = 5 minutes)            │
└──────────────────────────────────────────────────────────┘
  │              │               │
  ▼              ▼               ▼
Payment       User           Auth
Service       Service        Service
```

---

### Authentication at the Gateway

```text
Request arrives with: Authorization: Bearer <JWT>

Gateway:
  1. Fetch JWKS from Auth Service (cached, TTL=5min)
  2. Verify JWT signature
  3. Check exp (not expired)
  4. Check iss (expected issuer)
  5. Check aud (expected audience)
  6. Extract: user_id, roles, tenant_id from JWT claims
  7. Inject headers: X-User-Id: 123, X-User-Roles: payment_user,viewer

Downstream services:
  Trust X-User-Id header (set by gateway; cannot be forged by clients)
  Don't re-validate JWT (the gateway already did)
  
  But: validate that the request came through the gateway
  (using mTLS between gateway and services, or a shared secret in the header)
```

**The mTLS alternative**: Services accept requests only from the gateway. They present certificates to each other. No service accepts requests from external IPs. Provides defence-in-depth: even if a service is accidentally exposed, it won't accept requests from non-gateway sources.

---

### Gateway Bottleneck

The gateway is a single point of failure and a potential performance bottleneck.

```text
At 100K req/sec:
  Gateway CPU for JWT validation: 0.1ms × 100K = 10 CPU-seconds/second
  On 8-core gateway: 10/8 = 1.25 CPUs dedicated to JWT validation
  → Fine for 100K req/sec

At 1M req/sec:
  Gateway CPU for JWT validation: 12.5 CPUs dedicated to JWT validation
  → Horizontal scaling: 3-4 gateway nodes
  → Or: cache JWKS validation results (valid JWT → 1min cache hit)
```

**Gateway caching strategy**:
```text
Cache: { JWT_hash → validated_claims }
TTL: min(JWT exp - now, 60 seconds)

On cache hit: skip JWT signature verification entirely
On cache miss: verify signature, cache result

Cache hit rate for typical session: ~95% (user makes many requests with same JWT)
CPU savings: 95% of JWT validation work eliminated.
```

---

### Multi-Region Gateway

```text
Region: India                          Region: EU
  ┌──────────────┐                     ┌──────────────┐
  │ API Gateway  │                     │ API Gateway  │
  │  (Mumbai)    │                     │  (London)    │
  └──────┬───────┘                     └──────┬───────┘
         │                                    │
  India services                        EU services
  (India user data stays in India)      (EU user data stays in EU)

DNS (Route 53 geo-routing):
  India users → Mumbai gateway
  EU users → London gateway
  Cross-region: if Mumbai is down, India users → next closest (Singapore)
```

---

## Canonical Problem: IAM Platform API Gateway

### Requirements
- All external APIs authenticated via JWT (issued by ForgeRock)
- Rate limiting: 1000 req/min per API key for partner APIs; no limit for internal services
- Routing: /auth/*, /tokens/*, /admin/*, /api/*
- Observability: every request logged with user_id, latency, status
- Multi-region: Mumbai primary, Singapore DR

### Design

```text
Technology: Kong (open-source) or AWS API Gateway or Envoy

Configuration:

Routes:
  /auth/*      → auth-service:8080    (no auth required for login)
  /tokens/*    → token-service:8080   (no auth required for token ops)
  /admin/*     → admin-service:8080   (requires admin role)
  /api/*       → api-gateway-router:8080 → (routes to individual services)

Plugins (applied in order):
  1. JWT Verification Plugin:
     JWKS URL: https://iam-internal/jwks
     Cache TTL: 300 seconds
     Skip paths: ["/auth/login", "/auth/refresh", "/tokens/introspect"]
  
  2. Rate Limiting Plugin:
     Partner API keys: 1000/minute (sliding window, Redis-backed)
     Internal service tokens: unlimited
  
  3. Request Transformer Plugin:
     Add: X-Request-ID = uuid() if not present
     Add: X-Gateway-Region = "mumbai"
  
  4. Response Transformer Plugin:
     Add: X-Request-ID = (echo back)
     Remove: internal headers (X-Internal-Service, X-Trace-Id)
  
  5. Logging Plugin:
     Log: timestamp, method, path, status, latency, user_id, api_key, region
     Destination: Kafka → Elasticsearch

mTLS to downstream services:
  Gateway presents certificate (SAN: api-gateway.iam.internal)
  Services accept only from this certificate
```

---

## Deep Dive: Gateway Bottleneck at Scale

**The scenario**: 500K req/sec. Every request does JWT validation. The gateway becomes CPU-bound.

**Solution hierarchy**:

1. **Cache JWT validation results** (most impactful, no infra change):
   - Cache `hash(JWT) → {user_id, roles, expiry}` in Redis
   - 95% cache hit rate → 95% reduction in JWT validation CPU

2. **Horizontal scale the gateway** (stateless → easy):
   - 5 gateway nodes behind a load balancer
   - Redis cluster for shared cache

3. **Move auth to a sidecar** (service mesh approach):
   - Each service has an Envoy sidecar that validates JWTs
   - Gateway does routing only; no auth
   - Distributes auth CPU across all service nodes

4. **Short-circuit for trusted internal services**:
   - Internal services (machine-to-machine) use client credentials tokens
   - These are simpler to validate (HMAC, not RSA signature)
   - Or: use mTLS certificate identity — no JWT needed for service-to-service

---

## Interview Calibration

**Staff-level question**: "What happens when the Auth Service that issues JWKS is down?"

**Staff-level answer**: "The gateway caches JWKS with a TTL of 5-10 minutes. During Auth Service downtime, the gateway continues validating JWTs against the cached JWKS for up to 10 minutes. This is the designed degradation window. For tokens that expire during this window: validation fails gracefully — the user's token is invalid, they must re-authenticate (which will also fail until Auth Service recovers). Security is maintained: no tokens are accepted that the auth service hasn't signed. After 10 minutes: if Auth Service is still down, JWT validation fails for all new tokens but existing valid tokens continue to work until their own expiry."

This demonstrates understanding of: caching strategy, failure modes, security vs availability tradeoffs, and time-bounded degradation.

---
---

## Evolution Path

```text
Stage 1: No Gateway (services exposed directly)
  Each service handles its own auth and routing. Simple. Doesn't scale.
  Breaking point: 10 services × auth logic = 10 places to update security fixes.
  Signal: security incident caused by inconsistent auth implementation across services.

Stage 2: Reverse Proxy (nginx/HAProxy)
  Route by path. SSL termination. Basic.
  Breaking point: no auth, no rate limiting, no observability built in.
  Signal: need auth enforcement or rate limiting → can't do it at nginx config complexity.

Stage 3: API Gateway (Kong, AWS API Gateway, Envoy)
  Auth, rate limiting, routing, observability in one layer.
  Breaking point: gateway is SPOF; or gateway performance becomes bottleneck.
  Signal: gateway CPU > 70% from JWT validation at peak; or gateway outage = total outage.

Stage 4: Gateway + Service Mesh (east-west traffic secured separately)
  API gateway handles north-south (client → services).
  Service mesh (Istio/Envoy sidecar) handles east-west (service → service).
  Breaking point: Istio control plane operational complexity.
  Signal: team can't manage Istio policy complexity; frequent misconfiguration incidents.

Stage 5: Distributed Edge Gateway (global, multi-region)
  Gateway deployed in every region. Nearest edge serves client.
  Global config sync: control plane distributes routing rules to all edges.
  Breaking point: never a technical limit; operational complexity is the challenge.
```

---

## Migration Strategy

**From a reverse proxy (nginx) to a full API gateway (Kong)**

```text
Step 1: Deploy Kong alongside nginx. Both running. No traffic to Kong yet.
        Configure Kong to mirror nginx routing rules exactly.

Step 2: Dark launch — 1% of traffic to Kong. Compare: response codes, latency.
        Kong must match nginx exactly before proceeding.

Step 3: Migrate auth enforcement to Kong plugin (JWT validation).
        Initially: allow both authenticated (Kong) and unauthenticated (nginx path) requests.
        Validate: JWT validation working correctly for 24 hours.

Step 4: Enable rate limiting in Kong.
        Shadow mode: record would-be rate limited requests but don't block.
        Validate: no legitimate users would be blocked.

Step 5: Ramp traffic to Kong: 10% → 50% → 100% over 1 week.
        Decommission nginx reverse proxy after 30 days stable.

Rollback at any step: shift traffic back to nginx (load balancer weight adjustment, seconds).
Data risk: none (gateway is stateless; no data migration required).
```

---

## Organisational Ownership

```text
Platform / Security team owns:
  - Gateway infrastructure (capacity planning, HA configuration).
  - Auth plugin configuration (JWT validation, JWKS, algorithm allowlist).
  - Rate limiting policies (per-endpoint limits, global DDoS protection).
  - Routing rules (path → service mappings).
  On-call: gateway unavailability, auth failures, rate limit misconfiguration.

Application teams own:
  - Registering their services with the gateway.
  - Specifying their rate limit requirements.
  - Ensuring their services enforce auth at the service level (defence in depth).
  - Testing that their service works correctly behind the gateway.
  On-call: application-specific errors behind the gateway.

Security team:
  - Approves all new routing rules and auth policy changes.
  - Reviews gateway configuration changes before production deployment.
  - Owns the threat model; drives security testing (pen test quarterly).

Ownership boundary conflict:
  "User getting 401" — is it a gateway auth misconfiguration (platform)
  or an expired token the user should refresh (application/client)?
  Resolution: gateway logs the specific JWT validation failure reason.
  Token expired → application/client issue. Gateway config → platform issue.
```

---

## Incident Response

**Symptom**: All API calls failing / auth service unreachable / 401/403 spike

```text
Diagnose:
  1. Is the gateway itself down?
     curl https://api.platform.com/health → if connection refused: gateway down.
  2. Is auth service (JWKS source) down?
     curl http://auth-service/jwks → if down: JWT validation using cached JWKS.
     Check: when does cache expire? (TTL = 5-10 minutes typically)
  3. 401 spike: JWT expired or malformed?
     Sample 10 failing requests: decode JWT (jwt.io) → check exp, iss, aud.
  4. 403 spike: permissions changed or route misconfiguration?
     Check: recent gateway config deployments.

Mitigation:
  A. Gateway down: Kubernetes restarts automatically.
     If persistent: kubectl describe pod <gateway-pod> → OOM? Config error?
     Emergency bypass: direct traffic to services (only if behind VPN/internal network).
  B. Auth service down: gateway serves existing JWT validations from JWKS cache.
     New sessions cannot be created. Existing sessions continue until token expiry.
     Fix: restore auth service; new tokens can be issued.
  C. JWKS cache expired + auth service still down: gateway must reject new unverifiable tokens.
     Impact: users with expired tokens cannot make API calls.
     Mitigation: extend JWKS cache TTL (redeploy gateway config); or restore auth service.

Recovery: API success rate returns to > 99% for 5 minutes.
```

---

## Design Review Checklist

```text
□ Gateway horizontally scalable? (stateless; no local state)
□ JWKS cached at gateway? Cache TTL defined? What happens on auth service failure?
□ JWT validation cached by JWT hash? (95% CPU reduction for repeated tokens)
□ Circuit breaker per upstream service? (auth service down → fast fail, not timeout cascade)
□ Rate limiting: by user_id? by API key? by IP? Correct scoping for each endpoint?
□ Request ID injected for distributed tracing?
□ mTLS between gateway and upstream services? (prevents bypassing gateway)
□ Emergency bypass plan documented? (what if gateway itself needs emergency maintenance?)
□ Routing rules version-controlled and reviewed before deployment?
□ Alert: gateway error rate > 1%?
□ Alert: gateway latency p99 > 50ms (excluding upstream latency)?
□ Alert: auth service JWKS unreachable (cache TTL countdown started)?
□ Multi-region: gateway deployed in each region? Config sync working?
```

---

## Threat Model

```text
T1: JWT Algorithm Confusion (Spoofing)
  Attack: attacker sends JWT with alg:none or alg:HS256 (when RS256 expected).
  Mitigation: gateway explicitly allowlists accepted algorithms;
              never accepts alg:none;
              JWT library configured with expected algorithm (not from JWT header).

T2: SSRF via Gateway Routing (Elevation of Privilege)
  Attack: craft request to route to internal services not meant for external access.
  Mitigation: allowlist-based routing (only explicitly configured upstreams);
              internal services unreachable from external network (VPC isolation);
              gateway blocks paths: /actuator, /admin, /internal, /health (if internal-only).

T3: API Key Leakage (Information Disclosure + Spoofing)
  Attack: API key exposed in URL, logs, or source code.
  Mitigation: API keys only in Authorization header (never URL params);
              automated secret scanning in CI (gitleaks, truffleHog);
              API key rotation every 90 days; immediate revocation API.

T4: Tenant Escape (Elevation of Privilege)
  Attack: tenant A's request accesses tenant B's resources.
  Mitigation: gateway extracts tenant_id from JWT; injects X-Tenant-Id header;
              gateway never accepts X-Tenant-Id from external clients;
              all upstreams validate resource tenant_id == header tenant_id.

T5: DDoS on Gateway (Availability)
  Attack: flood gateway with requests to exhaust its capacity.
  Mitigation: CDN/WAF upstream absorbs DDoS before it reaches gateway;
              rate limiting at gateway (per IP, per API key);
              autoscaling gateway instances based on CPU/request rate.
```

---

*End of Pattern 15: API Gateway Pattern*
# Pattern 16: Search, Ranking, and Recommendation

> "There are two types of search: finding what you asked for, and finding what you should have asked for. Ranking is the difference."

---

## The Shape

Users find things via text queries, but relevance matters as much as recall. At Staff level, "design a search system" now almost always includes ranking, personalisation, and increasingly, semantic/vector search alongside keyword search.

---

## When You See This Pattern

- Product search with ranking (Amazon, Flipkart)
- Content discovery (Netflix "what to watch")
- People search (LinkedIn, Twitter)
- Recommendation feeds ("because you watched")
- Semantic search ("find contracts similar to this one")
- Fraud signal similarity ("find transactions similar to known fraud")

The signal: **not just "find results matching X" but "find the BEST results for this user's query and context."**

---

## Building Blocks

### The Three Layers of Modern Search

```text
Layer 1: Retrieval (find candidates)
  Keyword: inverted index (Elasticsearch, Solr)
  Semantic: vector similarity (Pinecone, pgvector, Weaviate)
  
Layer 2: Ranking (order candidates by relevance)
  BM25 (term frequency with saturation)
  Learning-to-rank (ML model trained on clicks)
  
Layer 3: Personalisation (adjust for this user)
  User embedding: "this user prefers X category"
  Context: "this user is in Mumbai, show local products"
  Re-ranking: apply personalisation after initial ranking
```

---

### BM25 (the industry standard)

BM25 replaced TF-IDF as the standard because it handles long documents better (saturation: a word appearing 100 times isn't 10× more relevant than 10 times).

```text
BM25 score(D, Q) = sum over terms t in Q:
  IDF(t) × (TF(t,D) × (k1+1)) / (TF(t,D) + k1 × (1 - b + b × |D|/avgdl))

k1 = 1.2 (controls TF saturation)
b = 0.75 (controls document length normalisation)
|D| = document length
avgdl = average document length in corpus
```

In practice: Elasticsearch uses BM25 by default. You configure it, not implement it.

---

### Vector / Semantic Search

Traditional search fails at: "find shoes that look like this image", "find contracts with similar meaning", "recommend songs that feel like this one."

Vector search finds items by **meaning** (embedding similarity) not **keywords**.

```text
Offline (indexing):
  Each product → ML model → 768-dimension embedding vector
  Store: vector DB (Pinecone, Weaviate) or pgvector

Online (query):
  User query: "comfortable shoes for standing all day"
  → ML model → query embedding vector
  → ANN search: find K nearest neighbors in embedding space
  → Return products with similar embeddings

Distance metric: cosine similarity
  0 = completely different meaning
  1 = identical meaning
```

**Approximate Nearest Neighbor (ANN)**: Exact nearest neighbor search is O(N×d) per query (N=documents, d=dimensions). Intractable at 100M products. ANN algorithms (HNSW, IVF) trade small accuracy loss for 100-1000× speedup.

---

### Hybrid Search (Best of Both)

```text
Keyword search (BM25):
  "nike running shoes" → exact term matches
  Good for: specific product names, SKUs, exact phrases

Semantic search (vector):
  "comfortable shoes for long shifts" → meaning matches
  Good for: natural language, synonyms, concepts

Hybrid:
  Run both. Merge results with weighted scoring:
  final_score = α × bm25_score + (1-α) × vector_similarity
  
  Tune α based on query type detection:
    Looks like a product name? → α=0.8 (keyword dominant)
    Looks like natural language? → α=0.2 (semantic dominant)
```

---

### Learning-to-Rank (LTR)

An ML model trained on user behavior (clicks, purchases, dwell time) to predict the optimal ordering of search results.

```text
Training data:
  Query: "running shoes"
  Result A: shown at position 1, user clicked → positive signal
  Result B: shown at position 3, user didn't click → negative signal
  
  Features: BM25 score, vector similarity, product rating, price, recency, in-stock

Model: gradient-boosted trees (XGBoost) or neural network
  Trained to predict: will user click on this result?
  
Online: fetch top 100 candidates from BM25+vector, re-rank with LTR model
```

**Why not just use LTR directly**: LTR is too slow for full corpus retrieval. Use BM25/vector to retrieve top K candidates (fast), then LTR to re-rank (slow but only K items).

---

### Recommendation Systems

**Collaborative filtering** ("users like you also liked"):
```text
User-Item matrix:
           item1  item2  item3  item4
  userA:     5      ?      3      ?
  userB:     4      2      ?      5
  userC:     ?      3      4      2
  
  userA is similar to userB (both liked item1, item3)
  Recommend to userA: items userB liked that userA hasn't seen → item4
```

**Matrix factorisation (ALS, SVD)**: Decompose the User-Item matrix into User-Embeddings × Item-Embeddings. Then: recommendation = user_embedding · item_embedding.

**Two-tower model (modern approach)**:
```text
Tower 1: User Tower
  Input: user_id, recent_actions, demographics
  Output: user_embedding (128-dim)

Tower 2: Item Tower
  Input: item_id, category, price, description
  Output: item_embedding (128-dim)
  
  Pre-compute item embeddings for all items (offline)
  
At query time:
  Compute user embedding (fast, online)
  ANN search: find top K items closest to user embedding
  Re-rank with contextual features (price, location, inventory)
```

---

## Canonical Problem: Design Netflix Recommendation

### Requirements
- 250M users, 15K titles
- "What to watch" homepage: personalised top 40 titles
- Refresh: daily (not real-time)
- Latency: p99 < 100ms for homepage load

### Architecture

```text
Offline (daily batch):
  User watch history → Spark job → ALS matrix factorisation → User embeddings (250M × 128-dim)
  Title metadata → content embedding model → Title embeddings (15K × 128-dim)
  Store: user_embeddings table (DynamoDB or BigTable)
  Store: title_embeddings in vector DB (HNSW index for ANN)

Online (per homepage request):
  GET /recommendations?user_id=123
  
  1. Load user embedding from DynamoDB: ~5ms
  2. ANN search: top 200 candidate titles (from HNSW index): ~10ms
  3. Re-rank with contextual features (currently trending, user's device, time of day): ~5ms
  4. Filter: remove already-watched, unavailable in region: ~2ms
  5. Return top 40: JSON response
  
  Total: ~22ms → well within 100ms SLO

Daily refresh:
  Batch job (Spark/Flink) runs at 3am
  Recomputes user embeddings for all 250M users
  Updates DynamoDB
  Next day's homepage loads from fresh embeddings
```

---

## Capacity Estimation

```text
User embeddings storage:
  250M users × 128 dimensions × 4 bytes/float = 128 GB
  DynamoDB storage: $0.25/GB/month = $32/month (tiny)

ANN index for titles:
  15K titles × 128 dimensions = trivial (< 10MB)
  Fits in-process on every API server

Recommendation QPS:
  250M users × 1 homepage load/day = 250M/86400 = 2890 req/sec average
  Peak (prime time 8-10pm): 10× = 28,900 req/sec
  
  At 22ms/request and 28,900 req/sec:
    Concurrent requests = 28,900 × 0.022 = 636 concurrent
    10 API servers (c6g.4xlarge, 1000 concurrent each): easily handles peak
```

---

## SCC Mapping

```text
STATE:
  User embeddings (daily stale is acceptable for recommendations)
  Item embeddings (updated when catalog changes)
  ANN index (rebuilt nightly from item embeddings)

COORDINATION:
  Offline batch job must complete before online tier reads new embeddings
  Use a version key: "today's embeddings are version 2024-01-15"
  API servers check version; load new embeddings atomically

CONCENTRATION:
  New release (blockbuster): every user's embedding points toward it
  Every user's recommendation includes the new title
  ANN search: same item appears in 100M users' results simultaneously
  Fix: pre-warm new title in CDN before launch
```

---

## Technology Selection

```text
Elasticsearch vs Vector DB for semantic search:
  Elasticsearch 8.x: now supports vector search (kNN plugin)
    Pro: one system for keyword + vector
    Con: vector search is slower than dedicated vector DBs
  Pinecone (managed vector DB):
    Pro: purpose-built ANN, sub-10ms at 100M vectors
    Con: additional infrastructure, cost
  pgvector (PostgreSQL extension):
    Pro: no new infrastructure, exact+approximate search
    Con: slower than Pinecone at scale (>10M vectors)
  
  Decision: pgvector for <10M vectors, Pinecone/Weaviate above that.
```

---

*End of Pattern 16: Search, Ranking, and Recommendation*

---

---

## Evolution Path

```text
Stage 1: Keyword Search Only (simple, fast to build)
  Elasticsearch BM25. Keyword matching. No ranking intelligence.
  Breaking point: users phrase queries naturally; exact keywords don't match intent.
  Signal: high "zero results" rate; user reformulation rate high; search abandonment.

Stage 2: Fuzzy + Autocomplete (better UX, same index)
  fuzziness=AUTO; edge-ngram for autocomplete. Same infrastructure.
  Breaking point: still purely keyword-based; semantic meaning ignored.
  Signal: "comfortable running shoes" doesn't match "athletic footwear for long distances."

Stage 3: Hybrid Keyword + Vector Search (semantic understanding)
  Add embedding model. Store vectors alongside inverted index.
  Hybrid scoring: BM25 + cosine similarity.
  Breaking point: generic embeddings miss domain-specific nuance.
  Signal: domain terminology ("NBFC," "term loan," "STR") not matching correctly.

Stage 4: Personalisation + Learning-to-Rank (relevance tuned to user behaviour)
  Re-rank top-K results using ML model trained on click/purchase signals.
  User embeddings incorporated into ranking.
  Breaking point: model requires significant click data (cold start problem for new items).
  Signal: new items never surfaced despite being relevant.

Stage 5: Fine-Tuned Domain Embeddings (highest relevance, most investment)
  Fine-tune embedding model on domain-specific data (payment docs, legal contracts, medical records).
  Breaking point: ML infrastructure complexity; retraining pipeline required.
  Signal: this is where Google, LinkedIn, Amazon operate. Requires dedicated ML team.
```

---

## Incident Response

**Symptom**: Search returning wrong results / vector index corrupted / recommendation cold

```text
Diagnose:
  1. Compare result set between "keyword only" and "hybrid" modes.
     → If keyword results correct but hybrid wrong: vector index issue.
  2. Check vector index health (Pinecone/pgvector):
     SELECT COUNT(*) FROM product_embeddings; -- should match products count
  3. For recommendations: check user embedding freshness.
     SELECT MAX(updated_at) FROM user_embeddings; -- stale if batch job failed.

Mitigation:
  A. Vector index corrupted: rebuild from stored embeddings (re-insert all vectors).
     Temporary: fall back to keyword-only search (feature flag).
  B. Stale user embeddings: manually trigger batch embedding job.
     Temporary: serve popularity-based recommendations (not personalised).
  C. Zero results for valid queries: check index lag (CDC pipeline).
     New documents not yet indexed. Check Kafka consumer lag for indexing consumer group.
```

---

## Design Review Checklist

```text
□ Search index is derived (from DB via CDC) — not the primary store?
□ Embedding model chosen? Domain-specific or general? Justified?
□ ANN index type chosen? (HNSW for recall, IVF for memory efficiency)
□ Hybrid scoring weights (α for BM25, 1-α for vector) tuned and documented?
□ Cold start problem addressed for new items? (popularity boost for new items)
□ Learning-to-rank model: training data pipeline defined? Retraining cadence?
□ Tenant isolation enforced in search results? (user cannot find other tenants' documents)
□ Zero-results rate monitored? Alert if > 5% of queries return no results?
□ Recommendation freshness: how stale are user embeddings? Acceptable?
□ A/B testing infrastructure for ranking experiments?
```

---

## Threat Model

```text
T1: Cross-Tenant Data Discovery (Information Disclosure)
  Attack: crafted query surfaces documents from other tenants.
  Mitigation: tenant_id filter on every search and ANN query;
              penetration test quarterly.

T2: Embedding Inversion Attack (Information Disclosure)
  Attack: from a document's embedding vector, infer the original document content.
  Mitigation: embeddings do not uniquely reconstruct content at current model sizes;
              do not expose raw embedding vectors in API responses;
              encrypt embedding store at rest.

T3: Recommendation Poisoning (Tampering)
  Attack: attacker generates fake engagement signals (bot clicks) to boost their content.
  Mitigation: anomaly detection on click patterns (bot detection);
              rate limit clicks per user per item per hour;
              weight signals by user reputation score.
```

---


---

## Migration Strategy

**From keyword search to hybrid keyword + vector search**

```text
Step 1: Add embedding model and store vectors for existing documents.
        Dark launch: compute embeddings for all documents; store in pgvector or Pinecone.
        No query path changes yet. Validate: embedding quality on sample queries.

Step 2: A/B test hybrid scoring on 5% of search traffic.
        Metric: click-through rate (CTR), zero-results rate, search abandonment.
        Compare: hybrid vs keyword-only.

Step 3: If metrics improve: ramp to 50% → 100%.
        If metrics regress: roll back (disable vector component; feature flag).

Rollback: disable vector search via feature flag. Keyword-only resumes in seconds.
Data risk: none (embeddings are additive; original index unchanged).
```

---

## Organisational Ownership

```text
Search Platform team owns:
  - Elasticsearch / vector DB infrastructure.
  - Embedding model serving (inference endpoint).
  - CDC pipeline (DB → Kafka → search index).
  On-call: cluster health, index lag, search error rate.

Product team owns:
  - Query construction, boosting rules, relevance tuning.
  - A/B test design and analysis.
  On-call: relevance regressions, zero-results rate.

ML team (if present):
  - Embedding model training and fine-tuning.
  - Learning-to-rank model pipeline.
```

---

*End of Pattern 16: Search, Ranking, and Recommendation*
# Pattern 17: Analytics and Aggregation Systems

> "The difference between data and insight is a GROUP BY clause on a billion rows."

---

## The Shape

The system must answer aggregate questions over large datasets in near-real-time or batch: "how many payments in the last hour by status?", "what is the fraud rate by merchant category in the last 7 days?", "what is the p99 latency for this API endpoint in the last 5 minutes?"

---

## When You See This Pattern

- Design a metrics platform (like Datadog, Prometheus)
- Design an analytics dashboard (e-commerce sales analytics)
- Design a fraud analytics system
- Design a monitoring system
- Design a financial reporting system

---

## The Two Architectures

### Lambda Architecture

```text
Batch Layer (high accuracy, high latency):
  All historical data → Spark/Hadoop batch job → precomputed batch views
  Runs: hourly or daily
  Accuracy: perfect (processes complete dataset)
  Latency: minutes to hours

Speed Layer (low accuracy, low latency):
  Recent data → streaming processor (Flink/Kafka Streams) → realtime views
  Runs: continuously
  Accuracy: approximate (only recent data)
  Latency: seconds

Serving Layer:
  Query = batch view (for old data) + realtime view (for recent data)
  Result: complete, near-real-time answer

Problem: two codebases for the same logic (batch and streaming).
         Synchronisation complexity.
```

### Kappa Architecture (Preferred)

```text
Single Layer:
  All data → Kafka → streaming processor → queryable store
  
  For historical reprocessing:
    Reset consumer offset to the beginning of the Kafka topic
    Replay all events through the same streaming code
    Overwrite the queryable store with recomputed results
    
Advantage: one codebase, one processing model.
Works because Kafka retains events (configurable retention: days to forever).
```

---

## The Aggregation Spectrum

```text
Pre-aggregation (fastest reads, most expensive writes):
  Write path: update aggregates on every event
    INSERT INTO hourly_stats (merchant_id, hour, payment_count, payment_total)
    VALUES (123, '2024-01-15-14', 1, 100.00)
    ON CONFLICT (merchant_id, hour)
    DO UPDATE SET payment_count = hourly_stats.payment_count + 1,
                  payment_total = hourly_stats.payment_total + 100.00;
  
  Read path: SELECT * FROM hourly_stats WHERE merchant_id = 123 AND hour = '2024-01-15-14'
  Result: sub-millisecond
  Cost: every payment write also hits hourly_stats → 2× write load

On-demand aggregation (slowest reads, cheapest writes):
  Write path: append event to raw event table
  Read path: SELECT COUNT(*), SUM(amount) FROM payments WHERE merchant_id=123 AND ...
  Result: seconds to minutes on large tables
  Cost: read is expensive; write is cheap

Materialized aggregation (middle ground):
  Write: append event to raw table
  Background: materialized view refreshes every 60s
  Read: SELECT from materialized view
  Result: <10ms reads, up to 60s stale
  Cost: refresh cost amortised across 60s window
```

---

## Canonical Problem: Design a Payment Analytics Dashboard

### Requirements
- Merchants want: real-time payment volume, failure rate, top categories — refreshed every minute
- Finance wants: daily reconciliation report (total volume by payment method) — batch
- Fraud team wants: live transaction stream with anomaly signals — sub-10s latency
- 10M transactions/day, 100K merchants

### Architecture

```text
Write Path:
  Payment Service → emits PaymentCompleted event → Kafka: payment.events

Real-Time Aggregation (Flink):
  Consumes payment.events
  Tumbling window: aggregate per merchant, per minute
  Output → ClickHouse: merchant_minute_stats
    (merchant_id, minute, count, total_amount, success_count, failure_count)

Fraud Stream (Flink):
  Consumes payment.events
  Applies anomaly detection rules:
    - >3 failed payments in 5 minutes from same IP → fraud signal
    - Amount > 10× user's 30-day average → outlier signal
  Output → fraud.signals Kafka topic → real-time fraud alert service

Batch Reconciliation (Spark, daily at 2am):
  Read from payment.events (90-day Kafka retention)
  OR read from ClickHouse (raw event store)
  Compute: daily_reconciliation_report table
  Output: CSV to S3 (finance team downloads)

Query Layer:
  Merchant Dashboard: SELECT FROM merchant_minute_stats WHERE merchant_id=? AND minute>?
  Finance Report: SELECT FROM daily_reconciliation_report WHERE date=?
  Fraud Analyst: Subscribe to fraud.signals via Kafka consumer
```

---

## Capacity Estimation

```text
Event volume:
  10M payments/day = 116 payments/sec average, 1K/sec peak
  Event size: ~500 bytes
  Throughput: 116 × 500 = 58 KB/sec average (tiny)

ClickHouse raw event store:
  10M events/day × 500B = 5 GB/day compressed (ClickHouse ~10× compression)
  → 500 MB/day actual
  → 1 year = 183 GB
  Cost: trivial

Flink aggregation:
  1K events/sec input
  Output: 100K merchants × 1 row/minute = 100K rows/minute to ClickHouse
  ClickHouse handles 100K+ inserts/sec: no problem

Merchant dashboard query:
  SELECT FROM merchant_minute_stats WHERE merchant_id = ? AND minute > NOW()-1hr
  = 60 rows maximum
  ClickHouse: <5ms even on 1TB tables
```

---

## Technology Selection

```text
ClickHouse vs Druid vs Pinot for real-time analytics:
  ClickHouse:
    Best for: general analytics, strong SQL, batch + near-real-time
    Ingestion latency: seconds (via Kafka → ClickHouse)
    QPS: 100K+ aggregation queries/sec on single node
    Operational simplicity: high (single binary, no ZooKeeper)
    
  Apache Druid:
    Best for: high-cardinality dimensions, sub-second query latency
    Ingestion latency: sub-second (native Kafka integration)
    Complex to operate (ZooKeeper, multiple node types)
    
  Apache Pinot:
    Best for: user-facing analytics (LinkedIn uses it for profile views)
    Sub-second at very high QPS
    Complex to operate
    
  Decision: ClickHouse for most analytics use cases. Pinot/Druid when
  sub-second latency at very high QPS is required (LinkedIn-scale).

Flink vs Kafka Streams vs Spark Streaming:
  Flink: most powerful, true streaming (not micro-batch), stateful windowing
  Kafka Streams: embedded in your app, no separate cluster needed, simple
  Spark Streaming: micro-batch, good for batch+streaming hybrid, slower than Flink
  
  Decision: Kafka Streams for simple aggregations in existing Java services.
            Flink for complex event processing (pattern matching, windowing, CEP).
```

---

## SCC Mapping

```text
STATE:
  Raw event log (Kafka): the source of truth
  Aggregated views (ClickHouse): derived, queryable
  Problem: aggregated views can diverge from raw data if Flink bug exists
  Fix: ability to rebuild aggregated views by replaying Kafka topic

COORDINATION:
  Exactly-once aggregation: Flink with checkpointing + Kafka exactly-once semantics
  Without: aggregates can be double-counted on Flink restart

CONCENTRATION:
  Hot merchant: one merchant (Flipkart, Amazon) with 10% of all transactions
  Their minute-level aggregation row is updated 100× more than average
  Fix: partition ClickHouse by merchant shard; hot merchants get dedicated partition
```

---

*End of Pattern 17: Analytics and Aggregation Systems*

---
---

## Evolution Path

```text
Stage 1: Direct DB Aggregation Queries (0–10M rows)
  SELECT COUNT(*), SUM(), GROUP BY on OLTP DB. Works.
  Breaking point: queries start taking > 10 seconds; OLTP DB performance degrades.
  Signal: analytics queries competing with OLTP queries for DB resources.

Stage 2: Dedicated Read Replica for Analytics (10M–100M rows)
  Route analytics queries to read replica. OLTP DB isolated.
  Breaking point: replica still uses row-based storage; aggregations slow.
  Signal: GROUP BY on 100M rows still > 5 seconds even on replica.

Stage 3: Materialized Views + Scheduled Aggregation (100M–1B rows)
  Pre-compute common aggregations. Refresh on schedule (hourly/daily).
  Breaking point: freshness gap (hourly refresh = 1-hour stale data).
  Signal: real-time dashboards unacceptable with > 5-minute stale data.

Stage 4: ClickHouse / Columnar Analytics DB (1B+ rows, near-real-time)
  Purpose-built OLAP. Columnar storage. 100-1000× faster aggregations.
  Kafka → ClickHouse consumer: events available for query within seconds.
  Breaking point: complex joins across large tables; or transactional requirements.
  Signal: ClickHouse query joins multiple large tables > 30 seconds.

Stage 5: Lambda/Kappa Architecture (petabyte scale, mixed latency requirements)
  Kappa: Kafka → streaming processor (Flink) → ClickHouse (all queries).
  Lambda: Kafka → Spark batch (accuracy) + Flink streaming (recency) → serving layer.
  Breaking point: never. This is Google/Meta scale.
```

---

## Incident Response

**Symptom**: Dashboard showing wrong numbers / analytics pipeline stopped / ClickHouse slow

```text
Diagnose:
  1. Compare ClickHouse count vs PostgreSQL count for recent data:
     SELECT COUNT(*) FROM payments WHERE created_at > NOW() - INTERVAL '1 hour';
     -- Run on both. If ClickHouse < PostgreSQL: consumer lag.
  2. Kafka consumer lag for analytics consumer:
     kafka-consumer-groups.sh --describe --group analytics-consumer
  3. ClickHouse slow query log:
     SELECT query, query_duration_ms FROM system.query_log
     WHERE query_duration_ms > 10000 ORDER BY event_time DESC LIMIT 10;

Mitigation:
  A. Consumer lag: scale analytics consumer pods.
     ClickHouse handles high ingest rate; consumer is usually the bottleneck.
  B. ClickHouse slow: check merge operations (MergeTree merges can cause slowness).
     SELECT * FROM system.merges ORDER BY elapsed DESC;
     If heavy merging: reduce ingest rate temporarily; let merges complete.
  C. Wrong numbers: check for duplicate events (consumer reprocessed events).
     Analytics consumers must be idempotent (use ReplacingMergeTree in ClickHouse).
```

---

## Design Review Checklist

```text
□ Analytics DB separated from OLTP DB? (ClickHouse not PostgreSQL for aggregations)
□ Kafka → ClickHouse consumer idempotent? (ReplacingMergeTree for deduplication)
□ Consumer lag monitored? Alert threshold defined?
□ Historical reprocessing tested? (reset consumer offset → replay → verify counts match)
□ Data retention policy defined? (ClickHouse: hot data period; S3 Parquet: cold archive)
□ Partition key in ClickHouse distributes query load? (not a hot partition)
□ Dashboard freshness SLO documented and communicated to users?
□ Reconciliation job: periodic comparison of ClickHouse counts vs PostgreSQL?
□ Alert: analytics consumer lag > 5 minutes worth of events?
□ Alert: ClickHouse query p99 > SLO?
□ On-call defined for analytics pipeline?
```

---

## Threat Model

```text
T1: Analytics Data Leakage (Information Disclosure)
  Attack: analytics query surfaces PII or sensitive business metrics to wrong users.
  Mitigation: row-level security in ClickHouse (tenant_id filter enforced);
              PII fields hashed or masked before loading into analytics store;
              analytics access requires explicit role (not default).

T2: Query Resource Exhaustion (DoS)
  Attack: malicious or accidental query uses full cluster resources (full table scan).
  Mitigation: query complexity limits (max_execution_time, max_memory_usage in ClickHouse);
              quota per user/role; kill runaway queries automatically.

T3: Historical Data Manipulation (Tampering)
  Attack: modify historical analytics to hide fraud or inflate metrics.
  Mitigation: append-only analytics store (no UPDATE/DELETE on historical records);
              ClickHouse: ReplacingMergeTree allows correction via new record, not modification;
              write access to analytics store limited to pipeline service accounts only.
```

---


---

## Migration Strategy

**From scheduled batch ETL to CDC streaming pipeline**

```text
Step 1: Deploy Kafka and Debezium alongside existing ETL.
        ETL continues running. Debezium starts capturing WAL changes to Kafka.
        Validate: Kafka events match ETL source data.

Step 2: New analytics consumer reads from Kafka → writes to ClickHouse shadow table.
        Compare shadow table vs existing analytics table daily.
        When counts match consistently (7 days): proceed.

Step 3: Switch dashboards to read from ClickHouse (shadow becomes primary).
        ETL still runs (fallback). Monitor for 1 week.

Step 4: Disable ETL. ClickHouse is primary. CDC is live.

Rollback: re-enable ETL at any step. ETL and CDC can run concurrently safely.
```

---

## Organisational Ownership

```text
Data Platform team owns:
  - Kafka cluster, Debezium connectors, ClickHouse cluster.
  - Consumer deployment and autoscaling.
  On-call: pipeline failure, consumer lag, ClickHouse slow queries.

Application teams own:
  - Event schema (what data flows into analytics).
  - Dashboard queries and correctness validation.
  On-call: their dashboard's data accuracy, reconciliation discrepancies.

FinOps / Analytics team:
  - Data retention policy (cost vs compliance).
  - Query cost optimisation (review expensive queries).
```

---

*End of Pattern 17: Analytics and Aggregation Systems*
# Pattern 18: Data Synchronisation

> "The question is never 'how do I copy data from A to B.' It's 'how do I keep A and B consistent forever, even when either of them fails.'"

---

## The Shape

Data that lives in one system needs to flow to another — a database to a search index, an OLTP DB to a data warehouse, a primary to a replica, or one microservice's DB to another's.

---

## When You See This Pattern

- OLTP to OLAP (PostgreSQL → ClickHouse for analytics)
- DB to search index (PostgreSQL → Elasticsearch)
- DB to cache (PostgreSQL → Redis)
- Microservice to microservice (events via Kafka)
- Primary to replica (streaming replication)
- On-premise to cloud migration

---

## The Three Sync Strategies

### Strategy 1: Dual Write

Application writes to both systems simultaneously.

```text
payment_service.save_payment(payment):
  postgres.insert(payment)   # primary
  elasticsearch.index(payment)  # search index
```

**Problem**: either can fail independently. If Elasticsearch write fails but Postgres succeeds: they're out of sync. No way to detect or recover automatically. This is the **dual write problem** — guaranteed data divergence at scale.

**When it's acceptable**: low-write-rate, eventual consistency acceptable, divergence is detectable (periodic reconciliation job).

---

### Strategy 2: Outbox + CDC (Recommended)

Application writes to database only. CDC streams changes to all other systems.

```text
payment_service.save_payment(payment):
  BEGIN;
    postgres.insert(payment)
    postgres.insert(outbox_event: {type: PaymentCreated, payload: payment})
  COMMIT;

Debezium (CDC):
  Watches postgres WAL → publishes to Kafka: payment.events

Consumers:
  ElasticsearchSync: updates search index
  RedisCache: updates Redis
  AnalyticsPipeline: writes to ClickHouse
  AuditLog: writes to audit store
```

**Why this is correct**: The database commit is the single atomic source. All downstream systems are derived. If any consumer fails: it re-reads from Kafka. No divergence possible (only lag).

---

### Strategy 3: Periodic Reconciliation

Batch job compares two systems and fixes differences.

```text
Nightly reconciliation job:
  SELECT id, hash FROM postgres ORDER BY id;
  SELECT id, hash FROM elasticsearch ORDER BY id;
  
  diff = postgres_set - elasticsearch_set
  for missing_id in diff:
    elasticsearch.index(postgres.get(missing_id))
  
  extra = elasticsearch_set - postgres_set  
  for extra_id in extra:
    elasticsearch.delete(extra_id)
```

**When to use**: As a safety net alongside CDC (catches edge cases). As the primary strategy when write volume is low and reconciliation lag is acceptable (hourly, daily).

---

## Canonical Problem: Keep Elasticsearch in Sync with PostgreSQL

### Requirements
- 10M products in PostgreSQL
- Search index in Elasticsearch must reflect changes within 2 seconds
- Volume: 1000 product updates/day
- Occasional bulk updates: catalog repricing (10M updates in 2 hours)

### Architecture

```text
Normal operation (CDC):
  Product Service → writes to postgres.products
  Debezium: watches postgres WAL → publishes to Kafka: product.changes
  ES Indexer: consumes product.changes → updates Elasticsearch index
  Latency: ~1-2 seconds from write to searchable

Bulk repricing (special case):
  10M updates in 2 hours = 1,389 updates/sec
  CDC handles this: Debezium ingests all WAL changes
  Kafka: buffers the updates
  ES Indexer: batch-processes, 1000 updates/batch to Elasticsearch
  ES Indexer autoscales during bulk operation
  
Initial bootstrap:
  Debezium snapshot mode: reads all 10M products → publishes to Kafka
  ES Indexer: bulk-indexes from Kafka
  Bootstrap time: ~2 hours for 10M products at 1500 updates/sec
```

---

## Deep Dive: Handling Schema Changes

**Scenario**: Product table adds a new column `sustainability_score`. Elasticsearch index must include it.

```text
Step 1: Add column to PostgreSQL (no downtime — nullable with default)
Step 2: Debezium detects schema change in WAL
Step 3: Schema Registry: Debezium publishes new schema version
Step 4: ES Indexer: receives events with new field → adds to document
Step 5: Backfill: products that existed before the column need sustainability_score
        → Run reconciliation job: update ES documents for all products
Step 6: ES mapping update: add field to Elasticsearch index mapping
        (must do before documents with new field arrive, or use dynamic mapping)
```

**The zero-downtime index migration problem**: Elasticsearch doesn't allow changing field types in an existing index. To change a field type: create a new index, reindex all documents, swap the alias.

```text
Create: product_v2 index (new schema)
Reindex: POST /_reindex { source: product_v1, dest: product_v2 }
  → runs in background, ~1 hour for 10M documents
Update alias: product_alias → product_v2
Delete: product_v1 index
```

During reindex: reads go to old index (via alias); writes go to both old and new (dual-write temporarily, for the duration of reindex).

---

## SCC Mapping

```text
STATE:
  Two representations of the same truth: PostgreSQL (source) + Elasticsearch (derived)
  Problem: they can diverge; source is authoritative; derived must catch up

COORDINATION:
  CDC pipeline provides coordination: "what changed in postgres" → "update elasticsearch"
  Schema registry provides schema coordination: both sides agree on event structure

CONCENTRATION:
  Bulk updates (catalog repricing): 10M changes in 2 hours → Kafka queue builds up
  Kafka handles the burst; ES Indexer must autoscale to drain the queue
  Without autoscaling: 2-hour repricing → 4-hour index lag → stale search results
```

---

---

## Evolution Path

```text
Stage 1: Manual Export/Import (0–1K records/day, infrequent sync)
  CSV export, manual import. Painful. Error-prone. Works for small volumes.
  Breaking point: any volume > 10K records/day; any latency requirement < 1 day.
  Signal: data team manually running export scripts; errors on import.

Stage 2: Scheduled ETL (1K–1M records/day, hourly/daily sync)
  Scheduled job: SELECT from source → transform → INSERT to destination.
  Breaking point: source DB under load from ETL queries; or latency > 1 hour unacceptable.
  Signal: ETL queries causing production DB slowness; analysts complaining about stale data.

Stage 3: CDC (Change Data Capture) — Debezium + Kafka (1M+ records/day, near-real-time)
  Watch source DB WAL; emit change events; consumers update destinations.
  No load on source DB (reads WAL, not query).
  Breaking point: CDC infrastructure complexity; schema changes require careful coordination.
  Signal: team struggles with Debezium configuration; schema change causes pipeline failure.

Stage 4: Managed CDC Platform (at scale, multiple sources/destinations)
  Fivetran, Airbyte, or Debezium Cloud.
  Pre-built connectors; managed schema evolution; monitoring included.
  Breaking point: never a technical limit; cost at high volume may be significant.

Stage 5: Data Mesh (multiple domains, autonomous data producers)
  Each domain publishes its own data products.
  Central data catalog; consumers discover and subscribe.
  Governance: data quality SLOs per domain; ownership clear.
```

---

## Incident Response

**Symptom**: Search index stale / data warehouse showing yesterday's data / CDC pipeline stopped

```text
Diagnose:
  1. Check Debezium connector status:
     curl http://debezium:8083/connectors/postgres-source/status
     → If FAILED: check error message.
  2. Check Kafka consumer lag for each downstream consumer.
  3. Check PostgreSQL replication slot lag:
     SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(
       pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
     FROM pg_replication_slots;
     → If > 1GB: Debezium not consuming; WAL accumulating; disk risk.

Mitigation:
  A. Debezium connector failed:
     → curl -X POST http://debezium:8083/connectors/postgres-source/restart
     → Check: schema change in PostgreSQL broke the connector (most common cause).
     → Fix: update connector schema; restart.
  B. Replication slot lag > 5GB (disk risk):
     → Priority: restore Debezium consumer immediately.
     → If can't: DROP REPLICATION SLOT 'debezium' (lose change history; need full re-sync).
     → Set max_slot_wal_keep_size = '10GB' to prevent disk full (slot invalidated at limit).
  C. Consumer downstream failed:
     → Kafka retains events. Consumer will catch up on recovery.
     → Monitor: consumer lag returning to 0.

Recovery: CDC pipeline metrics show zero lag; destination data matches source.
```

---

## Design Review Checklist

```text
□ CDC chosen over dual-write? (dual-write has consistency gaps; CDC is atomic with source)
□ Debezium replication slot lag monitored? Alert at 1GB? Max set at 10GB?
□ All CDC consumers idempotent? (at-least-once delivery from Kafka)
□ Schema change process defined? (adding nullable column: safe; dropping column: breaking)
□ Schema registry enforcing compatibility for CDC events?
□ Initial snapshot strategy for new consumers? (snapshot mode and duration tested)
□ Destination divergence check: periodic reconciliation job comparing source vs destination counts?
□ What happens if CDC pipeline is down for 24 hours? Data gap? Or Kafka retains?
□ Alert: replication slot lag > 1GB?
□ Alert: Debezium connector in FAILED state?
□ Alert: consumer lag > 10 minutes for each downstream?
□ PII masking/hashing before data lands in analytics destinations?
```

---

## Threat Model

```text
T1: Sensitive Data in Change Events (Information Disclosure)
  Attack: CDC events contain PII/secrets exposed to all Kafka consumers.
  Mitigation: field-level masking in Debezium (before publishing);
              only authorised consumer groups can read sensitive topics;
              encrypt Kafka topics at rest.

T2: Replication Slot Exploitation (Availability)
  Attack: consumer deliberately falls behind; WAL accumulates; PostgreSQL disk fills.
  Mitigation: max_slot_wal_keep_size limits WAL retention;
              monitor slot lag; alert + disable slot if lag exceeds threshold.

T3: Data Injection via CDC (Tampering)
  Attack: attacker modifies source DB records; CDC propagates malicious changes everywhere.
  Mitigation: source DB write access tightly controlled;
              CDC is a mirror — source DB security is the primary defence;
              audit log captures all source DB writes.
```

---


---

## Migration Strategy

**From dual-write to CDC-based sync**

```text
Step 1: Deploy Debezium watching source DB. Publish changes to Kafka.
        No consumers reading from Kafka yet. Validate: events arriving correctly.

Step 2: New consumer reads from Kafka → writes to destination (shadow mode).
        Destination shadow table compared with dual-write destination.
        When they match consistently (48 hours): proceed.

Step 3: Remove dual-write code. Kafka consumer is the only sync mechanism.
        Monitor: destination doc count vs source count.

Step 4: Reconciliation job (nightly): compare source and destination counts.
        Alert if divergence > 0.1%. Investigate and repair.

Rollback: re-enable dual-write. Run parallel until Kafka pipeline stable.
```

---

## Organisational Ownership

```text
Data Platform team owns:
  - Debezium connectors and Kafka topics.
  - Schema registry (compatibility enforcement).
  - Replication slot monitoring (disk risk).
  On-call: connector failures, slot lag > 1GB, schema breaks.

Application / Source team owns:
  - Schema changes (must notify platform team before schema change).
  - Data quality at source.

Destination team owns:
  - Consumer logic and idempotency.
  - Destination correctness (reconciliation job).
  On-call: destination divergence, consumer lag.
```

---

*End of Pattern 18: Data Synchronisation*
# Pattern 19: Platform and Control Plane Design

> "A platform is a product. Its customers are engineers. Its features are APIs. Its downtime is an incident for every team that depends on it."

---

## The Shape

A platform is shared infrastructure that many product teams build on. It provides capabilities — authentication, authorisation, feature flags, configuration management, service discovery, rate limiting, CI/CD — as self-service primitives.

Designing a platform is fundamentally different from designing a product feature:
- The users are engineers, not end users
- The failure mode is not "user sees error" but "all teams using this platform are blocked"
- The API is a long-lived contract, not an implementation detail
- Scale is measured in "teams onboarded" as well as "requests per second"

---

## When You See This Pattern

- "Design the IAM platform for a bank with 200 microservices"
- "Design a feature flag service"
- "Design a developer portal"
- "Design a CI/CD platform for 500 engineers"
- "Design a secrets management platform"
- "How would you design a multi-tenant policy engine?"
- Anything that ends with "...as a service" for internal teams

This pattern is almost universal at companies with large engineering organisations and is directly relevant to your IAM platform background.

---

## What Makes a Platform Different

```text
Product:                              Platform:
  Users are end users                   Users are engineers
  Features are UI/API capabilities      Features are APIs, SDKs, CLIs
  Version by product release            Version by API contract
  Failure affects users                 Failure affects all teams using it
  Scale = users × actions               Scale = teams × services × calls
  Optimise for UX                       Optimise for developer experience (DX)
  One team owns it                      Many teams depend on it
  Can break backwards compat            Cannot break backwards compat (contracts)
```

---

## The Platform Design Principles

### 1. Self-Service First

A platform that requires a ticket or a meeting to onboard a new team is not a platform — it is a shared service with a queue. A real platform: a team can onboard, integrate, and go to production without talking to the platform team.

```text
IAM Platform self-service:
  □ Service account creation: API + CLI, no human approval needed
  □ Role assignment: team lead approves, no IAM team involvement
  □ Token validation: documented API, SDK published, works on first try
  □ Audit log query: self-service dashboard, no data request ticket

Non-self-service (requires human): 
  □ Custom permission model (not covered by standard RBAC)
  □ Emergency access (break-glass procedures)
  □ Regulatory exception (data localisation override)
```

### 2. API Contract Stability

A platform's API is a public contract. Breaking it means breaking every team that uses it — simultaneously.

```text
Versioning strategy:
  /v1/tokens/introspect  → stable for 24 months minimum
  /v2/tokens/introspect  → new features, runs alongside v1
  Deprecation notice:    6 months before v1 retirement

What constitutes a breaking change:
  - Removing a field from response body
  - Changing a field's type
  - Changing HTTP status codes for existing cases
  - Changing required vs optional fields in request

What is NOT a breaking change:
  - Adding new optional fields to response (additive)
  - Adding new endpoints
  - Adding new optional request fields
  - Tightening rate limits (with 30-day notice)
```

### 3. Observability as a First-Class Feature

Platform teams owe their customers visibility into: "is it my bug or your platform?"

```text
IAM Platform observability contract:
  - Public status page: https://status.iam.internal
  - Per-service SLO dashboard: "token_introspection for my service: p99 latency"
  - Incident notifications: platform emails all oncalls when P0/P1 occurs
  - Attribution: every request tagged with service_name in platform logs
```

---

## Canonical Problem 1: Design an IAM Platform

This is your domain. Apply it directly.

### Requirements
- 200 internal services need token validation, authorisation, and audit logging
- Each service has different permission models (RBAC for most, ABAC for payments)
- Multi-tenant: each line of business is a separate tenant
- Compliance: full audit trail for every auth decision (RBI, PCI-DSS)
- Scale: 500K token introspections/sec, 5K logins/sec, 100K permission checks/sec

### Platform Architecture

```text
CONTROL PLANE (low volume, high importance):
  Policy Management API:
    - CRUD for roles, permissions, policies
    - Schema validation (is this a valid Rego policy?)
    - Policy versioning and rollback
    - Approval workflow for permission changes
  
  Tenant Management:
    - Onboard new tenant (line of business)
    - Configure: auth methods, MFA requirements, session policies
    - Data residency rules per tenant
  
  Audit Dashboard:
    - Query audit events across all tenants
    - Compliance reports (who accessed what, when)
    - Anomaly detection (unusual access patterns)

DATA PLANE (high volume, must be fast):
  Token Issuance Service:
    - POST /auth/token (login)
    - POST /auth/refresh (token refresh)
    - SLO: p99 < 200ms
  
  Token Introspection Service:
    - POST /tokens/introspect
    - GET /tokens/{id}/claims
    - SLO: p99 < 5ms (called on every API request)
    - Backed by: Redis cache (TTL=30s) → DB
  
  Policy Evaluation Service (OPA):
    - POST /evaluate { principal, resource, action, context }
    - Returns: ALLOW / DENY + reason
    - SLO: p99 < 2ms (called inline in API handlers)
    - OPA with policy cached in-process

AUDIT PLANE (high volume, append-only):
  Audit Ingestion:
    - Every IAM decision → Kafka topic: iam.audit-events
    - Outbox pattern: guaranteed delivery
  
  Audit Storage:
    - ClickHouse for recent queries (<90 days)
    - S3 Parquet for long-term (90 days - 7 years)
    - Athena for compliance queries on S3
```

### The Multi-Tenant Design

```text
Isolation model: namespace isolation (not cluster isolation)
  Tenant A (Retail Banking):    all data tagged with tenant_id = 'retail'
  Tenant B (Corporate Banking): all data tagged with tenant_id = 'corporate'
  
  Database: shared DB, row-level security (PostgreSQL RLS policy)
  Kafka:    shared topics, messages tagged with tenant_id
  Redis:    namespaced keys: {tenant_id}:{key}
  OPA:      policies namespaced per tenant: data.tenant.retail.allow
  
  Why namespace isolation (not cluster isolation):
    Cluster isolation: separate DB per tenant
    → 200 DB instances for 200 tenants → operational nightmare
    → Cost: 200× infrastructure
    → Namespace isolation: 1 DB, RLS policies → operational simplicity
    → Risk: misconfigured RLS → data leak across tenants
    → Mitigation: automated tests for tenant isolation, quarterly penetration tests

Tenant onboarding (self-service):
  1. Team registers tenant via Platform Portal (web UI + API)
  2. Platform creates: tenant record, default roles, admin service account
  3. Team configures: auth policy, MFA requirements, permitted IP ranges
  4. Platform issues: service account credentials for first integration
  5. Team integrates: SDK or direct API calls
  
  Time to first integration: <1 day for a competent team.
```

### The OPA Integration (Your IAM Context)

```text
Policy evaluation flow:

Service A (payment API) receives a request:
  → calls IAM: POST /evaluate
    { principal: { id: user-123, roles: [payment_user], tenant: retail },
      resource: { type: payment, id: pay-456, amount: 50000 },
      action: approve,
      context: { ip: 1.2.3.4, time: 14:30, mfa: true } }
  
  OPA evaluates against policies:
    data.retail.payments.allow {
      input.principal.roles[_] == "payment_approver"
      input.resource.amount <= 100000
      input.context.mfa == true
      input.principal.id != input.resource.initiator_id  # can't self-approve
    }
  
  Response: { decision: ALLOW, policy_version: v1.3, evaluation_time_ms: 1.2 }
  
  This response is also written to the audit log (asynchronously, via Outbox).

OPA performance:
  - Policy loaded in-process (not a remote call) → <2ms evaluation time
  - Policy reload on change: every 30 seconds (polling) or immediately (webhook from policy management API)
  - Policy version pinned per service: service can pin to v1.3 and not be affected by v1.4 policy changes until they explicitly upgrade
```

---

## Canonical Problem 2: Design a Feature Flag Service

### Requirements
- 500 engineers, 200 services
- Flags can be: global on/off, percentage rollout, user segment, A/B test
- Flag evaluation: <1ms p99 (called on critical paths)
- Flag changes: take effect within 30 seconds
- Audit: who changed what flag, when

### Architecture

```text
Storage (control plane):
  PostgreSQL: flags table (id, name, rules, created_by, updated_at)
  Kafka: flag.change events (CDC from PostgreSQL via Debezium)

Distribution:
  Flag Service: serves flag state via gRPC (SDK polling) or SSE (push)
  SDK (in every service): local in-process cache of all flags
    → On startup: full flag list loaded
    → On update: SSE event triggers cache refresh
    → Evaluation: fully in-process, ~microseconds, no network call

Evaluation types:
  BOOLEAN:    { "flag": "new_checkout", "enabled": true }
  PERCENTAGE: { "flag": "new_ui", "rollout_percentage": 20 }
    → hash(user_id) % 100 < 20 → enabled for this user
    → deterministic: same user always in same bucket
  SEGMENT:    { "flag": "beta", "segments": ["employees", "beta_users"] }
  A/B TEST:   { "flag": "checkout_cta", "variants": [
                  { "id": "control", "weight": 50, "value": "Buy Now" },
                  { "id": "treatment", "weight": 50, "value": "Purchase" }
               ]}

Emergency kill switch:
  All flags: RED flag overrides all other flags
  Used during incidents: disable all new features instantly
  Implemented as: SDK checks RED flag first before evaluating any other flag
```

### The Distributed Cache Consistency Problem

```text
Flag changes must take effect within 30 seconds.

Challenge: 500 services, each with in-process cache.
A flag change must propagate to all 500 service instances within 30 seconds.

Solution: SSE push from Flag Service to all SDK instances

Flag updated → Kafka → Flag Service consumer → SSE broadcast to all connected SDKs
Latency: ~1-2 seconds from change to all SDKs updated

What if an SDK missed the SSE event?
  SDK polls for flag list every 30 seconds as a fallback.
  Guarantees: any flag change takes effect within max(SSE latency, 30s) = 30s
```

---

## The Platform Lifecycle

```text
Stage 1: Internal Tool
  One team builds it for themselves.
  No self-service. No documentation. Works for them.

Stage 2: Shared Service
  Another team wants to use it. Original team onboards them manually.
  "Platform" is now a shared service with a queue.
  Works for 3-5 teams. Doesn't scale to 20.

Stage 3: Platform
  Self-service onboarding. Published API contract. Documentation.
  SDK published to internal package registry.
  Support model: async (not real-time). SLA defined.
  Scales to 50 teams.

Stage 4: Platform Product
  Platform team has a roadmap. Quarterly planning.
  Customer discovery: "what do our consumers need next?"
  Success metrics: teams onboarded, p99 latency, MTTR.
  Developer satisfaction survey.
  Scales to 500 teams.

Stage 5: Open-Source or Industry Standard
  Google open-sources Borg → Kubernetes.
  Netflix open-sources Hystrix, Eureka, Zuul.
  The platform has become infrastructure.
```

---

## Interview Calibration

**What separates Staff-level answers for Platform Design**:

- Immediately distinguishes control plane from data plane. These scale differently, fail differently, and have different SLOs.

- Addresses the API contract stability question: "the data plane API is stable for 24 months; the control plane API is versioned with 6-month deprecation notice."

- Considers the multi-tenant isolation model explicitly: namespace vs cluster isolation, and the specific security risk of each.

- Names the organisational model: "this requires a dedicated platform team. A product team owning shared infrastructure will deprioritise it."

- Quantifies the self-service maturity: "a team can onboard and go to production within one business day without talking to the platform team."

- Addresses the SDK distribution problem: "our SDK adds ~100KB to every service's binary and ~2ms to startup time; this is the tradeoff for <1ms flag evaluation."

---

## SCC Mapping

```text
STATE:
  Control plane: policy definitions, tenant configurations, audit records
  Data plane: session cache, token validation cache, policy evaluation cache
  
  The control plane is the source of truth.
  The data plane is a performance-optimised derived cache.
  
  The critical correctness property:
    A policy change (revoke user's role) must propagate from control plane to data plane
    within 30 seconds. Any longer: security window.

COORDINATION:
  Policy versioning: which version of the policy is each service using?
  Feature flag distribution: all service instances must see the same flag state
  Audit event ordering: events for a given user must be ordered (partition by user_id)

CONCENTRATION:
  Token introspection service: the hottest path (500K req/sec)
    Concentration mitigation: Redis cache (absorbs 95% of reads)
    Remaining 5%: read replicas distribute DB load
  
  Policy evaluation: called on every API request across all services
    Concentration mitigation: OPA in-process (zero network calls for evaluation)
    OPA cache: policy loaded in-process, refreshed every 30 seconds
```

---
---

## Evolution Path

```text
Stage 1: Internal Tool (one team uses it)
  Built for themselves. No documentation. No self-service. Works for 1 team.
  Breaking point: a second team wants to use it → manual onboarding → bottleneck.
  Signal: engineers filing tickets to use the service; 1-week onboarding time.

Stage 2: Shared Service (3–10 teams)
  Documentation exists. Basic API. Still requires manual onboarding steps.
  Breaking point: 10+ teams means 10+ sets of requests; platform team becomes a bottleneck.
  Signal: platform team spends 50%+ of time on onboarding and support rather than features.

Stage 3: Self-Service Platform (10–100 teams)
  Self-service onboarding (portal + API). SDK published. Documentation complete.
  Support: async (docs + Slack; not tickets or meetings).
  Breaking point: platform API becomes a contract that many teams depend on;
                  breaking changes become extremely costly.
  Signal: a platform API change breaks 30 teams simultaneously.

Stage 4: Platform Product (100+ teams)
  Platform team has a roadmap, quarterly planning, customer discovery.
  API versioning with deprecation notices. Breaking changes require 6-month notice.
  SLO published and monitored. Developer satisfaction measured.
  Breaking point: never. This is the mature state for large organisations.

Stage 5: Industry Standard / Open Source
  Platform becomes so valuable it's open-sourced or becomes an industry standard.
  Kubernetes, Kafka, OPA, Envoy — all started as internal platforms.
```

---

## Incident Response

**Symptom**: All teams' auth failing / feature flags returning defaults / IAM control plane down

```text
Diagnose:
  1. Distinguish control plane vs data plane failure:
     Control plane (policy management): admin portal, policy API — affects configuration.
     Data plane (token validation, flag evaluation): affects every API request.
  2. Data plane failure is a P0. Control plane is P1.
  3. Token validation failing:
     → Is Redis cache serving? curl http://token-service/health
     → Is JWKS endpoint reachable? curl http://iam/jwks
     → Are tokens in cache still valid? redis-cli TTL token:{hash}
  4. Feature flag returning defaults:
     → SDK is returning fallback values (flag service unreachable).
     → Check: flag service pods → kubectl get pods -n platform | grep flag-service

Mitigation:
  A. Token validation: if Redis down → fall back to DB validation (slower; circuit breaker).
     If DB down → JWKS cache still valid for TTL duration; serve cached validations.
     If JWKS expired + auth service down: new tokens invalid; existing tokens valid until expiry.
  B. Flag service down: SDK serves last-cached values (TTL = 30s for most SDKs).
     30s degradation window → SDK falls back to default values.
     Priority: restore flag service; most flag changes are not emergency.
  C. Control plane down: no new policy changes possible; existing policies continue working.
     Priority: P1 (affects configuration) but not P0 (doesn't affect traffic).

Recovery: data plane SLO (p99 latency, error rate) returns to normal for 5 minutes.
```

---

## Design Review Checklist

```text
Control Plane
□ API versioning strategy defined? Backward compatibility enforced?
□ Breaking change process: 6-month deprecation notice + migration guide?
□ Self-service onboarding: a new team can onboard without talking to platform team?
□ SDK published to internal package registry? Versioned?
□ Tenant isolation enforced? (tenant A cannot see/modify tenant B's configuration)
□ Audit log: every control plane change logged with actor, timestamp, diff?
□ Admin API access requires 2-factor auth + audit log?

Data Plane
□ Data plane SLO defined and published? (e.g., token introspection p99 < 5ms)
□ Data plane can serve traffic if control plane is down? (cached state is sufficient)
□ Circuit breaker: if DB is slow, serve from cache; don't cascade?
□ JWKS cache TTL defined? What happens when TTL expires + auth service is down?

Observability
□ Per-consumer SLO dashboard published? (each team sees their own latency/error rate)
□ Control plane change notifications sent to affected consumers?
□ Incident communication channel established? (platform → all consumers)
□ On-call rotation defined for platform team? Escalation path documented?
```

---

## Threat Model

```text
T1: Tenant Escape (Elevation of Privilege)
  Attack: tenant A modifies tenant B's policies, flags, or configurations.
  Mitigation: all control plane operations validated against tenant_id in JWT;
              DB-level row security (RLS) as defence in depth;
              quarterly penetration testing of tenant isolation.

T2: Policy Injection (Elevation of Privilege)
  Attack: malicious Rego/ABAC policy grants attacker elevated permissions.
  Mitigation: policy changes require approval workflow (two-party approval for production);
              policy linting: automated checks for dangerous patterns (allow {true});
              dry-run mode: simulate policy before applying.

T3: SDK Compromise (Supply Chain Attack)
  Attack: attacker compromises internal SDK package; malicious code distributed to all teams.
  Mitigation: SDK published to internal registry with code signing;
              SDK changes require security review;
              dependency scanning in all consumer CI pipelines.

T4: Feature Flag Abuse (Tampering)
  Attack: attacker enables a feature flag to expose experimental functionality.
  Mitigation: flag changes require role with explicit "flag_manager" permission;
              flag changes audit-logged with actor identity;
              emergency kill switch (RED flag) can disable all non-critical flags instantly.

T5: Control Plane API DoS (Availability)
  Attack: flood control plane API to prevent legitimate policy updates.
  Mitigation: control plane rate-limited per authenticated client;
              data plane independent of control plane (policy changes don't affect traffic serving);
              control plane behind VPN or internal network only.
```

---


---

## Migration Strategy

**From embedded shared library to dedicated platform service**

```text
Step 1: Extract current auth logic into a dedicated service (no behaviour change).
        All callers still call the library (which now proxies to the service internally).
        Validate: identical behaviour, same latency.

Step 2: Publish SDK that calls the platform service directly.
        Migrate callers one team at a time (low-risk teams first).
        Provide migration guide and support channel.

Step 3: Deprecate the shared library (6-month notice).
        All new teams: must use SDK (no direct library).
        Existing teams: migrate before deprecation date.

Step 4: Remove shared library. Platform service is the only path.
        Monitor: SDK adoption rate; support any remaining blockers.

Rollback: SDK includes a circuit breaker; falls back to direct DB auth if platform unavailable.
```

---

## Organisational Ownership

```text
Platform team owns:
  - IAM service infrastructure (control plane + data plane).
  - SDK published to internal registry.
  - API contract stability (versioning, deprecation, migration support).
  - SLO definition and monitoring.
  On-call: data plane P0 (everything stops); control plane P1 (config only).

Security team:
  - Approves all auth policy changes.
  - Runs quarterly penetration testing on tenant isolation.
  - Owns threat model; drives security requirements for platform team.

Application teams:
  - Integrate with SDK (not platform internals).
  - Report integration issues to platform team.
  - Own their service's auth logic (what permissions are required).
  On-call: their service's auth failures (post-validation by platform team).

Engineering leadership:
  - Funds platform team (central budget, not per-product).
  - Mandates SDK adoption (governance).
  - Reviews platform SLO quarterly.
```

---

*End of Pattern 19: Platform and Control Plane Design*
