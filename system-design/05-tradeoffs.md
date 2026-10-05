# Trade-off Gym — Every decision has a cost

[← System Design index](README.md)

> 54 paired choices. Make your call first, then compare both options on latency, throughput, consistency, complexity, cost and failure behaviour.

## Contents

- **Storage** (2): [Relational Store vs Key-Value Store](#relational-store-vs-key-value-store) · [Row Store vs Column Store](#row-store-vs-column-store)
- **Capacity** (1): [Scale Up vs Scale Out](#scale-up-vs-scale-out)
- **Service Architecture** (2): [Modular Monolith vs Microservices](#modular-monolith-vs-microservices) · [Shared Database vs Database Per Service](#shared-database-vs-database-per-service)
- **Communication** (2): [Synchronous RPC vs Asynchronous Messaging](#synchronous-rpc-vs-asynchronous-messaging) · [REST JSON vs gRPC Protobuf](#rest-json-vs-grpc-protobuf)
- **API Design** (2): [REST Resources vs GraphQL](#rest-resources-vs-graphql) · [Offset vs Cursor Pagination](#offset-vs-cursor-pagination)
- **Realtime Delivery** (2): [Server Sent Events vs WebSocket](#server-sent-events-vs-websocket) · [Short Polling vs Push Updates](#short-polling-vs-push-updates)
- **Caching** (5): [Cache Aside vs Read Through](#cache-aside-vs-read-through) · [Write Through vs Write Behind](#write-through-vs-write-behind) · [TTL Expiry vs Event Invalidation](#ttl-expiry-vs-event-invalidation) · [Local Cache vs Shared Remote Cache](#local-cache-vs-shared-remote-cache) · [LRU vs LFU Eviction](#lru-vs-lfu-eviction)
- **Consistency** (1): [Strong vs Eventual Read Consistency](#strong-vs-eventual-read-consistency)
- **Replication** (2): [Synchronous vs Asynchronous Replication](#synchronous-vs-asynchronous-replication) · [Single Writer vs Multiple Writers](#single-writer-vs-multiple-writers)
- **Partitioning** (3): [Hash Sharding vs Range Sharding](#hash-sharding-vs-range-sharding) · [Tenant Shard Key vs Entity Shard Key](#tenant-shard-key-vs-entity-shard-key) · [Consistent Hash Ring vs Directory Placement](#consistent-hash-ring-vs-directory-placement)
- **Feeds** (1): [Fanout On Write vs Fanout On Read](#fanout-on-write-vs-fanout-on-read)
- **Messaging** (2): [Work Queue vs Replayable Log](#work-queue-vs-replayable-log) · [At-Most-Once vs At-Least-Once Processing](#at-most-once-vs-at-least-once-processing)
- **Data Integration** (2): [CDC vs Table Polling](#cdc-vs-table-polling) · [ETL vs ELT](#etl-vs-elt)
- **Transactions** (4): [Transactional Outbox vs Direct Dual Write](#transactional-outbox-vs-direct-dual-write) · [Saga vs Two Phase Commit](#saga-vs-two-phase-commit) · [Saga Orchestration vs Choreography](#saga-orchestration-vs-choreography) · [Optimistic vs Pessimistic Concurrency](#optimistic-vs-pessimistic-concurrency)
- **Data Architecture** (2): [CQRS vs One Read-Write Model](#cqrs-vs-one-read-write-model) · [Event Sourcing vs Current-State Storage](#event-sourcing-vs-current-state-storage)
- **Coordination** (1): [Distributed Lock vs Conditional Update](#distributed-lock-vs-conditional-update)
- **Data Processing** (1): [Batch vs Stream Processing](#batch-vs-stream-processing)
- **Storage Internals** (1): [B-Tree vs LSM Storage](#b-tree-vs-lsm-storage)
- **Data Modeling** (2): [Normalize vs Denormalize](#normalize-vs-denormalize) · [Document Model vs Relational Model](#document-model-vs-relational-model)
- **Traffic Control** (4): [Token Bucket vs Leaky Bucket](#token-bucket-vs-leaky-bucket) · [Fixed vs Sliding Rate Window](#fixed-vs-sliding-rate-window) · [Queue Excess vs Reject Excess](#queue-excess-vs-reject-excess) · [Local vs Global Rate Limit](#local-vs-global-rate-limit)
- **Reliability** (1): [Retry vs Hedge](#retry-vs-hedge)
- **Media Storage** (2): [Object Storage vs Database Blobs](#object-storage-vs-database-blobs) · [Direct Signed Upload vs API Proxy Upload](#direct-signed-upload-vs-api-proxy-upload)
- **Media Delivery** (1): [Short vs Long Video Segments](#short-vs-long-video-segments)
- **Media Processing** (1): [Precompute Renditions vs Just-In-Time Encoding](#precompute-renditions-vs-just-in-time-encoding)
- **Global Architecture** (1): [Active-Passive vs Active-Active Regions](#active-passive-vs-active-active-regions)
- **Operations** (1): [Managed Service vs Self-Hosted Infrastructure](#managed-service-vs-self-hosted-infrastructure)
- **Compute** (1): [Serverless Functions vs Long-Running Containers](#serverless-functions-vs-long-running-containers)
- **Analytics** (1): [Exact Count vs Approximate Sketch](#exact-count-vs-approximate-sketch)
- **Scheduling** (2): [FIFO vs Priority Scheduling](#fifo-vs-priority-scheduling) · [Process Timer vs Durable Timer](#process-timer-vs-durable-timer)
- **Networking** (1): [Compress Payloads vs Send Uncompressed](#compress-payloads-vs-send-uncompressed)

## Storage

### Relational Store vs Key-Value Store

**Scenario:** Choose storage for orders with flexible reporting versus predictable high-volume key lookups.

|  | A — Relational store | B — Key-value store |
|---|---|---|
| **Wins when** | Cross-row invariants, joins and evolving query shapes matter. | Access is by known keys and partition-local operations. |

| Dimension | Comparison |
|---|---|
| Latency | A supports indexed queries; B avoids general query planning. |
| Throughput | B can scale partition-local writes; A can shard with added work. |
| Consistency | A commonly offers rich transactions; B guarantees depend on product and operation. |
| Complexity | A requires schema/index design; B requires access-pattern-first modeling. |
| Cost | A may need larger nodes; B may multiply denormalized copies. |
| Failure behaviour | A faces lock contention; B faces hot partitions and cross-key coordination. |

**Staff-level view:** A benefits from database tuning; B needs partition and consistency expertise.

---

### Row Store vs Column Store

**Scenario:** A dataset serves both individual order updates and large analytical scans.

|  | A — Row-oriented storage | B — Column-oriented storage |
|---|---|---|
| **Wins when** | Point reads and updates need most fields of a record. | Queries aggregate a few columns across many rows. |

| Dimension | Comparison |
|---|---|
| Latency | A retrieves complete records efficiently; B scans selected columns efficiently. |
| Throughput | A favors OLTP; B favors vectorized analytical throughput. |
| Consistency | Transactions and update semantics depend on engine, not layout alone. |
| Complexity | A models operational indexes; B models partitions, sort keys and ingestion batches. |
| Cost | B often compresses analytics better; A avoids reconstructing scattered columns. |
| Failure behaviour | A broad scans disturb OLTP; B small random updates may be expensive. |

**Staff-level view:** A needs transactional tuning; B needs analytical query and partition design.

---

## Capacity

### Scale Up vs Scale Out

**Scenario:** A service approaches the limits of its current deployment.

|  | A — Larger machines | B — More machines |
|---|---|---|
| **Wins when** | The workload is hard to partition and one larger node meets the target. | Independent work can be distributed and capacity must grow beyond one node. |

| Dimension | Comparison |
|---|---|
| Latency | A avoids extra network hops; B may reduce queueing under load. |
| Throughput | A has a machine ceiling; B depends on partition balance. |
| Consistency | A preserves local coordination; B introduces distributed state questions. |
| Complexity | A simplifies operations; B needs routing, discovery and rebalance. |
| Cost | A can have expensive size tiers; B adds fleet and network overhead. |
| Failure behaviour | A increases single-node blast radius; B tolerates nodes only with redundancy. |

**Staff-level view:** A needs performance tuning; B also needs distributed operations.

---

## Service Architecture

### Modular Monolith vs Microservices

**Scenario:** A small team is deciding how to deploy several business domains.

|  | A — Modular monolith | B — Microservices |
|---|---|---|
| **Wins when** | Domains change together and one team owns the system. | Teams need independent release, scaling or fault boundaries. |

| Dimension | Comparison |
|---|---|
| Latency | A uses local calls; B adds network and serialization latency. |
| Throughput | A shares one scaling unit; B scales hotspots independently. |
| Consistency | A supports local transactions; B needs cross-service consistency design. |
| Complexity | A needs module discipline; B adds contracts and distributed observability. |
| Cost | A has lower platform overhead; B duplicates runtime and infrastructure. |
| Failure behaviour | A deploy failures affect more modules; B has dependency cascades. |

**Staff-level view:** A suits a compact team; B needs service ownership and platform maturity.

---

### Shared Database vs Database Per Service

**Scenario:** Several domain teams need independent releases but share reporting needs.

|  | A — Shared database | B — Service-owned databases |
|---|---|---|
| **Wins when** | One team and cross-domain transactions benefit from local simplicity. | Data ownership and independent changes outweigh cross-service transaction cost. |

| Dimension | Comparison |
|---|---|
| Latency | A can join locally; B composes or projects across services. |
| Throughput | A shares a capacity pool; B scales domain stores independently. |
| Consistency | A supports local cross-domain transactions; B needs sagas or replicated views. |
| Complexity | A risks schema coupling; B adds integration contracts and reconciliation. |
| Cost | A consolidates infrastructure; B duplicates storage and operational work. |
| Failure behaviour | A schema or load incidents spread; B integration lag and partial failure appear. |

**Staff-level view:** B requires durable service ownership and data-contract governance.

---

## Communication

### Synchronous RPC vs Asynchronous Messaging

**Scenario:** An order request triggers downstream work that may take seconds.

|  | A — Synchronous RPC | B — Asynchronous messaging |
|---|---|---|
| **Wins when** | The caller needs an immediate authoritative answer. | Work can return pending and survive temporary consumer unavailability. |

| Dimension | Comparison |
|---|---|
| Latency | A returns a direct result; B reduces submission delay but completion waits. |
| Throughput | B absorbs bursts; neither removes sustained processing limits. |
| Consistency | A can report one operation directly; B exposes delayed state and duplicates. |
| Complexity | A needs deadlines; B needs queues, status, replay and idempotency. |
| Cost | A holds caller resources; B adds broker and retained-message cost. |
| Failure behaviour | A propagates outages; B accumulates backlog and poison messages. |

**Staff-level view:** A needs dependency SLOs; B needs consumer-lag and replay operations.

---

### REST JSON vs gRPC Protobuf

**Scenario:** Internal services exchange typed requests at high volume.

|  | A — REST with JSON | B — gRPC with Protobuf |
|---|---|---|
| **Wins when** | Human inspection and broad web integration are priorities. | Typed contracts, streaming and efficient internal communication matter. |

| Dimension | Comparison |
|---|---|
| Latency | B often reduces encoding overhead; actual network paths dominate. |
| Throughput | B can reduce bytes and CPU for structured payloads. |
| Consistency | Neither defines business consistency; retry safety remains application-specific. |
| Complexity | A has simple tooling; B adds schema generation and compatibility rules. |
| Cost | A may spend more bandwidth; B adds build and proxy integration effort. |
| Failure behaviour | A errors are easy to inspect; B status/deadline propagation needs care. |

**Staff-level view:** A suits broad API teams; B benefits from contract and RPC expertise.

---

## API Design

### REST Resources vs GraphQL

**Scenario:** Several clients need different combinations of product data.

|  | A — REST resources | B — GraphQL query API |
|---|---|---|
| **Wins when** | Resource operations are stable and HTTP caching is valuable. | Clients frequently need varied field combinations and nested data. |

| Dimension | Comparison |
|---|---|
| Latency | A may require more round trips; B can create costly resolver fanout. |
| Throughput | A routes are predictable; B needs query-cost and depth limits. |
| Consistency | Both need explicit freshness across composed data. |
| Complexity | A versions resource contracts; B maintains a schema and resolver layer. |
| Cost | B can reduce payload bytes but increase server query work. |
| Failure behaviour | A exposes endpoint failures; B must represent partial field failures. |

**Staff-level view:** B needs schema governance and resolver performance ownership.

---

### Offset vs Cursor Pagination

**Scenario:** A feed grows continuously while users page through it.

|  | A — Offset pagination | B — Keyset cursor pagination |
|---|---|---|
| **Wins when** | Small stable datasets need random page numbers. | Large ordered datasets need efficient continuation under inserts. |

| Dimension | Comparison |
|---|---|
| Latency | A deep pages scan or skip more rows; B resumes near an index key. |
| Throughput | B avoids repeated deep skipping on suitable indexes. |
| Consistency | A inserts cause shifting pages; B stable ordering reduces skips and duplicates. |
| Complexity | A is simple; B needs an opaque cursor and unique tie-breaker. |
| Cost | A deep queries consume database work; B adds cursor contract maintenance. |
| Failure behaviour | A can miss or repeat rows during changes; B deleted anchors need defined behavior. |

**Staff-level view:** A needs basic query skills; B needs ordering and index expertise.

---

## Realtime Delivery

### Server Sent Events vs WebSocket

**Scenario:** A browser receives frequent progress updates and occasional user commands.

|  | A — Server Sent Events | B — WebSocket |
|---|---|---|
| **Wins when** | Traffic is mainly server-to-client and ordinary HTTP fits deployment. | Both sides exchange frequent low-latency messages. |

| Dimension | Comparison |
|---|---|
| Latency | Both push promptly; B supports bidirectional traffic on one connection. |
| Throughput | Both require connection capacity and bounded output buffers. |
| Consistency | Neither supplies durable delivery; ids and replay are application concerns. |
| Complexity | A has browser reconnect support; B requires more session protocol logic. |
| Cost | A uses separate command requests; B adds gateway lifecycle work. |
| Failure behaviour | Both face reconnect storms; A can suffer proxy buffering. |

**Staff-level view:** A needs streaming HTTP knowledge; B needs persistent-connection operations.

---

### Short Polling vs Push Updates

**Scenario:** A dashboard must refresh while many users remain online.

|  | A — Periodic polling | B — Persistent push |
|---|---|---|
| **Wins when** | Updates are rare and several seconds of delay are acceptable. | Low update delay or high active-user count makes repeated empty reads wasteful. |

| Dimension | Comparison |
|---|---|
| Latency | A delay follows polling interval; B sends when changes arrive. |
| Throughput | A creates empty requests; B consumes persistent connections. |
| Consistency | A can read latest state; B needs reconnect replay or resynchronization. |
| Complexity | A is simple stateless HTTP; B needs connection routing and backpressure. |
| Cost | A spends requests; B spends idle connection resources. |
| Failure behaviour | A synchronized polls create spikes; B reconnect storms do. |

**Staff-level view:** A fits small operational teams; B needs connection-aware monitoring.

---

## Caching

### Cache Aside vs Read Through

**Scenario:** Several services repeatedly load the same reference data.

|  | A — Cache aside | B — Read-through abstraction |
|---|---|---|
| **Wins when** | Applications need explicit control over loading and fallback. | A shared loader contract reduces duplicated caller logic. |

| Dimension | Comparison |
|---|---|
| Latency | Both have fast hits and origin-bound misses. |
| Throughput | Both need miss coalescing and origin protection. |
| Consistency | Both require freshness and invalidation policy. |
| Complexity | A duplicates loading code; B centralizes loader behavior and dependency. |
| Cost | A uses a simple cache; B may require richer cache infrastructure. |
| Failure behaviour | A can bypass cache on error; B loader failure can block all callers. |

**Staff-level view:** A needs caller discipline; B needs loader ownership and operational visibility.

---

### Write Through vs Write Behind

**Scenario:** A fast state layer accepts updates before durable database persistence.

|  | A — Synchronous durable write-through | B — Buffered write-behind |
|---|---|---|
| **Wins when** | Acknowledged writes must already meet the durable-store contract. | Delay or a bounded loss budget is acceptable and batching helps. |

| Dimension | Comparison |
|---|---|
| Latency | A pays durable-write latency; B can acknowledge after buffering. |
| Throughput | B batches or coalesces writes; A writes each accepted operation. |
| Consistency | A has earlier persistence; B exposes lag and requires ordered flush. |
| Complexity | A must handle partial cache failure; B needs recovery and flush protocols. |
| Cost | B lowers database write load but adds durable buffering cost. |
| Failure behaviour | A fails with the store; B risks loss if acknowledged buffer is volatile. |

**Staff-level view:** B needs backlog, recovery and data-loss-budget ownership.

---

### TTL Expiry vs Event Invalidation

**Scenario:** Mutable catalog entries are cached in many places.

|  | A — TTL-only expiry | B — Event-driven invalidation |
|---|---|---|
| **Wins when** | A simple bounded stale window meets product needs. | Changes must become visible faster than an affordable TTL. |

| Dimension | Comparison |
|---|---|
| Latency | A avoids event dependencies; B can force cold misses after writes. |
| Throughput | A spreads reloads with jitter; B can invalidate a hot key fleet-wide. |
| Consistency | A bounds staleness only if refresh succeeds; B depends on event delivery and races. |
| Complexity | A is simpler; B tracks affected keys, ordering and lost events. |
| Cost | A may repeat unnecessary loads; B adds message and purge traffic. |
| Failure behaviour | A stale windows are expected; B silent event loss can surprise users. |

**Staff-level view:** B needs event-lag monitoring and a fallback expiry policy.

---

### Local Cache vs Shared Remote Cache

**Scenario:** Every API instance reads a common set of hot records.

|  | A — In-process cache | B — Shared remote cache |
|---|---|---|
| **Wins when** | Lowest latency and small reusable datasets dominate. | Instances should share capacity and one invalidation target. |

| Dimension | Comparison |
|---|---|
| Latency | A avoids network hops; B adds remote access latency. |
| Throughput | A scales hits per process; B centralizes hot-key load. |
| Consistency | A copies diverge across instances; B still needs database coherence. |
| Complexity | A simplifies deployment; B needs cache cluster operations. |
| Cost | A duplicates memory; B adds network and managed-service charges. |
| Failure behaviour | A loses cache on restart; B outages can affect the whole fleet. |

**Staff-level view:** A needs memory tuning; B needs capacity and failover expertise.

---

### LRU vs LFU Eviction

**Scenario:** A cache sees both recent bursts and repeatedly popular keys.

|  | A — Least recently used | B — Least frequently used |
|---|---|---|
| **Wins when** | Recent recency predicts the next access and popularity changes quickly. | Stable popularity should survive one-time scans. |

| Dimension | Comparison |
|---|---|
| Latency | Both add metadata work; implementation quality determines overhead. |
| Throughput | A can churn under scans; B can protect long-term hot keys. |
| Consistency | Eviction policy changes hit rate, not source-of-truth correctness. |
| Complexity | A tracks recency; B needs frequency approximation and aging. |
| Cost | B metadata and tuning may cost more; higher hits can offset it. |
| Failure behaviour | A evicts useful keys during scans; B retains formerly popular keys without decay. |

**Staff-level view:** A is easier to reason about; B needs workload trace evaluation.

---

## Consistency

### Strong vs Eventual Read Consistency

**Scenario:** A globally distributed application mixes inventory decisions with profile displays.

|  | A — Linearizable reads and writes | B — Eventually consistent reads |
|---|---|---|
| **Wins when** | A decision must reflect completed writes and preserve a single-object authority order. | Availability and local latency matter more than immediate visibility. |

| Dimension | Comparison |
|---|---|
| Latency | A may require coordination; B can read a local replica. |
| Throughput | B spreads reads widely; A is bounded by its coordination path. |
| Consistency | A provides a real-time order; B permits stale or conflicting observations. |
| Complexity | A needs quorum/leader protocols; B needs conflict and stale-data handling. |
| Cost | A pays cross-region coordination; B pays reconciliation and duplicate views. |
| Failure behaviour | During partitions A may reject; B may accept divergent updates. |

**Staff-level view:** Both need expertise; B also needs product-level conflict semantics.

---

## Replication

### Synchronous vs Asynchronous Replication

**Scenario:** A primary database replicates to another failure domain.

|  | A — Synchronous acknowledgement | B — Asynchronous replication |
|---|---|---|
| **Wins when** | Acknowledged writes must reach required replicas before success. | Foreground latency is critical and nonzero RPO is acceptable. |

| Dimension | Comparison |
|---|---|
| Latency | A waits for replicas; B usually acknowledges earlier. |
| Throughput | A depends on required replica capacity; B can buffer lag temporarily. |
| Consistency | A strengthens acknowledged durability; B may lose recent writes on failover. |
| Complexity | A needs quorum membership policy; B needs lag and promotion safeguards. |
| Cost | A adds write-path network cost; B adds catch-up storage and reconciliation. |
| Failure behaviour | A can stop writes on replica loss; B can promote stale state. |

**Staff-level view:** A needs quorum operations; B needs tested RPO and failover procedures.

---

### Single Writer vs Multiple Writers

**Scenario:** A record can be updated from several regions.

|  | A — One authoritative writer | B — Active writers in several regions |
|---|---|---|
| **Wins when** | Updates are noncommutative and invariants need one serialization point. | Local writes and partition-time availability justify conflict semantics. |

| Dimension | Comparison |
|---|---|
| Latency | A remote writers pay network delay; B can accept locally. |
| Throughput | A authority can bottleneck; B distributes independent writes. |
| Consistency | A simplifies ordering; B needs merge rules or coordinated transactions. |
| Complexity | A needs routing and failover; B needs conflict detection and repair. |
| Cost | B adds regional resources and conflict-processing work. |
| Failure behaviour | A loss requires promotion; B partitions can create incompatible updates. |

**Staff-level view:** B requires strong domain modeling and distributed data expertise.

---

## Partitioning

### Hash Sharding vs Range Sharding

**Scenario:** A database must distribute writes while supporting key-range queries.

|  | A — Hash partitioning | B — Range partitioning |
|---|---|---|
| **Wins when** | Point lookups dominate and uniformly spread keys are useful. | Ordered scans and range locality dominate. |

| Dimension | Comparison |
|---|---|
| Latency | A point routes are simple; B range reads can avoid broad scatter. |
| Throughput | A spreads distinct keys; B monotonic inserts can create a hot edge. |
| Consistency | Both move cross-shard invariants into coordination. |
| Complexity | A complicates range scans; B needs split and hotspot management. |
| Cost | A may fan out scans; B may waste capacity on uneven ranges. |
| Failure behaviour | A hot keys remain hot; B hot ranges overwhelm individual owners. |

**Staff-level view:** A needs distribution analysis; B needs range migration and split expertise.

---

### Tenant Shard Key vs Entity Shard Key

**Scenario:** A multi-tenant service contains both tiny and extremely large customers.

|  | A — Partition by tenant | B — Partition by entity |
|---|---|---|
| **Wins when** | Tenant isolation, locality and tenant-wide operations are central. | Large tenants must distribute load across many partitions. |

| Dimension | Comparison |
|---|---|
| Latency | A keeps tenant queries local; B can scatter tenant-wide queries. |
| Throughput | A is capped by the largest tenant shard; B spreads one tenant's work. |
| Consistency | A simplifies tenant-local transactions; B may split their invariants. |
| Complexity | A needs large-tenant escape hatches; B needs routing and aggregation. |
| Cost | A may strand capacity per tenant; B adds cross-partition reads. |
| Failure behaviour | A isolates tenant incidents; B can spread one tenant's load across the fleet. |

**Staff-level view:** A needs placement operations; B needs query and transaction design expertise.

---

### Consistent Hash Ring vs Directory Placement

**Scenario:** Data owners must be found while capacity changes.

|  | A — Algorithmic consistent hashing | B — Explicit partition directory |
|---|---|---|
| **Wins when** | Decentralized predictable placement and limited movement are priorities. | Operators need controlled tenant moves and custom placement. |

| Dimension | Comparison |
|---|---|
| Latency | A computes placement locally; B needs a cached directory lookup. |
| Throughput | A balances via virtual nodes; B can place based on measured load. |
| Consistency | Both require versioned membership or migration epochs. |
| Complexity | A needs ring agreement; B maintains authoritative mapping state. |
| Cost | A has small routing metadata; B adds directory service cost. |
| Failure behaviour | A stale rings misroute; B stale directory caches target old owners. |

**Staff-level view:** B needs migration tooling; A needs careful membership and weight management.

---

## Feeds

### Fanout On Write vs Fanout On Read

**Scenario:** A feed combines posts from followed accounts with highly skewed audiences.

|  | A — Precompute follower timelines | B — Assemble feeds when read |
|---|---|---|
| **Wins when** | Users read frequently and most authors have modest follower counts. | Publication volume or celebrity fanout makes copying expensive. |

| Dimension | Comparison |
|---|---|
| Latency | A makes feed reads cheap; B adds candidate gathering and merge. |
| Throughput | A amplifies writes; B amplifies reads. |
| Consistency | A timelines lag fanout; B sees source updates sooner if queried fresh. |
| Complexity | A needs durable fanout and deletion handling; B needs bounded query fanout. |
| Cost | A stores many references; B spends query CPU and source reads. |
| Failure behaviour | A queues fall behind spikes; B slow authors or shards delay reads. |

**Staff-level view:** A needs queue operations; B needs ranking and query performance expertise.

---

## Messaging

### Work Queue vs Replayable Log

**Scenario:** Events feed independent analytics consumers while some tasks need one worker.

|  | A — Work queue | B — Partitioned event log |
|---|---|---|
| **Wins when** | Jobs need claim, retry and completion semantics. | Multiple consumers need independent replay and ordered history. |

| Dimension | Comparison |
|---|---|
| Latency | A dispatches ready jobs; B consumer lag controls freshness. |
| Throughput | A scales workers with broker semantics; B parallelism follows partitions. |
| Consistency | A often redelivers; B also needs duplicate-safe external effects. |
| Complexity | A manages job lifecycle; B manages offsets, retention and partition ordering. |
| Cost | A retains unfinished work; B retains history for replay. |
| Failure behaviour | A poison jobs loop; B lag can exceed retention. |

**Staff-level view:** A needs retry/dead-letter operations; B needs replay and partition expertise.

---

### At-Most-Once vs At-Least-Once Processing

**Scenario:** A consumer can crash between a side effect and an acknowledgement.

|  | A — At-most-once attempts | B — At-least-once delivery with deduplication |
|---|---|---|
| **Wins when** | Occasional loss is acceptable and duplicate execution is worse. | Accepted work must be retried until handled. |

| Dimension | Comparison |
|---|---|
| Latency | A can acknowledge early; B retries increase worst-case completion time. |
| Throughput | A avoids redelivery work; B pays deduplication and replay overhead. |
| Consistency | A permits loss; B permits duplicates unless effects are idempotent. |
| Complexity | A is simpler; B needs stable ids and atomic effect/receipt handling. |
| Cost | B adds receipt storage and duplicate processing cost. |
| Failure behaviour | A crashes lose work; B crashes can repeat external side effects. |

**Staff-level view:** B needs explicit idempotency boundaries and replay procedures.

---

## Data Integration

### CDC vs Table Polling

**Scenario:** A search index needs changes from a transactional database.

|  | A — Log-based change data capture | B — Polling changed rows |
|---|---|---|
| **Wins when** | Low lag and complete update/delete propagation matter. | Small scale and a simple supported query are sufficient. |

| Dimension | Comparison |
|---|---|
| Latency | A streams after commit; B freshness follows poll interval. |
| Throughput | A avoids repeated scans; B adds source query load. |
| Consistency | A needs snapshot/log continuity; B needs reliable watermarks and delete tracking. |
| Complexity | A operates connectors and offsets; B can start with a scheduled query. |
| Cost | A adds stream infrastructure; B spends database reads. |
| Failure behaviour | A lag can exhaust logs; B watermark bugs skip equal-timestamp updates. |

**Staff-level view:** A needs database-log expertise; B needs query and reconciliation discipline.

---

### ETL vs ELT

**Scenario:** Operational data must enter an analytical warehouse.

|  | A — Transform before loading | B — Load raw then transform |
|---|---|---|
| **Wins when** | Sensitive fields must be removed first or targets cannot transform efficiently. | Warehouse compute and raw-data replay support flexible analytics. |

| Dimension | Comparison |
|---|---|
| Latency | A delays raw availability; B loads early but curated results still wait. |
| Throughput | A scales external processors; B consumes warehouse compute. |
| Consistency | Both need versioned transformations and stable deduplication keys. |
| Complexity | A coordinates separate engines; B centralizes SQL transformations. |
| Cost | A spends transfer and transform infrastructure; B retains raw data and warehouse scans. |
| Failure behaviour | A transform failure blocks ingestion; B raw ingestion can outpace quality controls. |

**Staff-level view:** A needs pipeline engineering; B needs warehouse modeling and governance.

---

## Transactions

### Transactional Outbox vs Direct Dual Write

**Scenario:** A service updates a database and publishes an order event.

|  | A — Outbox in the local transaction | B — Write database then publish directly |
|---|---|---|
| **Wins when** | A committed update must eventually have a corresponding event. | Best-effort notifications tolerate missed publication and reconciliation. |

| Dimension | Comparison |
|---|---|
| Latency | A publication waits for relay; B can publish immediately after commit. |
| Throughput | A adds relay throughput limits; B adds broker latency to request handling. |
| Consistency | A records atomic publication intent; B has a crash gap between systems. |
| Complexity | A needs relay, dedupe and cleanup; B is simpler but incomplete under failures. |
| Cost | A stores pending events; B incurs reconciliation cost if gaps matter. |
| Failure behaviour | A can duplicate publication; B can lose publication or emit before rolled-back data. |

**Staff-level view:** A needs messaging operations; B needs explicit acceptance of best-effort behavior.

---

### Saga vs Two Phase Commit

**Scenario:** A business operation spans independently owned transactional services.

|  | A — Saga with local commits | B — Two phase commit |
|---|---|---|
| **Wins when** | The process is long and compensating business actions are valid. | All participants support prepare and atomic commit is mandatory. |

| Dimension | Comparison |
|---|---|
| Latency | A can return pending; B waits for prepare and decision. |
| Throughput | A avoids holding global locks; B can retain locks while prepared. |
| Consistency | A exposes intermediate states; B gives atomic commit, with isolation separately defined. |
| Complexity | A needs compensation and state tracking; B needs coordinator recovery. |
| Cost | A pays workflow/event overhead; B pays coordination and held-resource cost. |
| Failure behaviour | A compensation can fail; B prepared participants can block. |

**Staff-level view:** A needs business-process design; B needs distributed transaction operations.

---

### Saga Orchestration vs Choreography

**Scenario:** Checkout crosses inventory, payment and fulfillment services.

|  | A — Central durable orchestrator | B — Event choreography |
|---|---|---|
| **Wins when** | The process has many steps, branching and operational visibility needs. | A few stable participants react independently to clear domain events. |

| Dimension | Comparison |
|---|---|
| Latency | A adds coordinator calls; B follows event delivery latency. |
| Throughput | A has orchestration capacity needs; B can spread independent reactions. |
| Consistency | Both require local atomicity, deduplication and compensation semantics. |
| Complexity | A makes flow explicit; B can hide process dependencies across subscribers. |
| Cost | A adds workflow infrastructure; B adds tracing and event-governance work. |
| Failure behaviour | A coordinator outage pauses progress; B cycles and missing events stall invisibly. |

**Staff-level view:** A needs workflow ownership; B needs strong cross-team event contracts.

---

### Optimistic vs Pessimistic Concurrency

**Scenario:** Several clients may update the same record.

|  | A — Version check and retry | B — Lock before modifying |
|---|---|---|
| **Wins when** | Conflicts are rare and transactions are short. | Conflicts are common or retrying expensive work is wasteful. |

| Dimension | Comparison |
|---|---|
| Latency | A avoids waiting until conflict; B queues behind lock holders. |
| Throughput | A scales under low contention; B controls high-contention serialization. |
| Consistency | Both can protect an invariant when all required records and predicates are covered. |
| Complexity | A needs conflict handling; B needs lock ordering and timeout policy. |
| Cost | A spends retry work; B spends held connections and lock memory. |
| Failure behaviour | A retry storms under contention; B deadlocks and long holders block progress. |

**Staff-level view:** A needs version-aware API design; B needs transaction and deadlock expertise.

---

## Data Architecture

### CQRS vs One Read-Write Model

**Scenario:** Queries need complex projections while commands enforce transactional rules.

|  | A — Separate command and query models | B — Shared CRUD model |
|---|---|---|
| **Wins when** | Read shapes and scale differ substantially from write needs. | One schema and indexed queries already meet the requirements. |

| Dimension | Comparison |
|---|---|
| Latency | A gives optimized reads; B avoids projection lag. |
| Throughput | A scales projections independently; B shares database resources. |
| Consistency | A often has eventual views; B can read authoritative transactional state. |
| Complexity | A needs projectors and rebuilds; B has fewer moving parts. |
| Cost | A duplicates data and services; B may need stronger database nodes. |
| Failure behaviour | A stale projections fail independently; B database contention affects both paths. |

**Staff-level view:** A needs event and projection ownership; B suits smaller teams.

---

### Event Sourcing vs Current-State Storage

**Scenario:** The product needs to explain how state changed over years.

|  | A — Immutable business event history | B — Mutable current-state records |
|---|---|---|
| **Wins when** | Replay and historical decision reconstruction are fundamental. | Current queries dominate and ordinary audit logs are sufficient. |

| Dimension | Comparison |
|---|---|
| Latency | A rebuilds may need snapshots; B directly reads current rows. |
| Throughput | A appends efficiently but projections add work; B updates indexes directly. |
| Consistency | A uses aggregate concurrency rules; B uses database transactions. |
| Complexity | A needs event evolution and replay discipline; B needs schema migrations. |
| Cost | A retains history and projections; B usually stores less. |
| Failure behaviour | A bad historical events persist; B accidental overwrites require backups or audit. |

**Staff-level view:** A needs domain-event and replay expertise; B uses familiar database operations.

---

## Coordination

### Distributed Lock vs Conditional Update

**Scenario:** Several workers compete to claim one database-owned job.

|  | A — External distributed lock | B — Atomic database conditional update |
|---|---|---|
| **Wins when** | The resource spans systems and can validate fencing tokens. | The protected state already lives in one transactional store. |

| Dimension | Comparison |
|---|---|
| Latency | A adds coordination hops; B claims at the data authority. |
| Throughput | A serializes via lock service; B uses database contention controls. |
| Consistency | A requires stale-holder fencing; B can atomically check claim generation. |
| Complexity | A needs lease and ownership lifecycle; B needs suitable predicates and indexes. |
| Cost | A adds a service; B adds database claim traffic. |
| Failure behaviour | A expired holders can act; B failed attempts must distinguish lost claims from errors. |

**Staff-level view:** A needs coordination expertise; B needs careful database concurrency design.

---

## Data Processing

### Batch vs Stream Processing

**Scenario:** A business dashboard needs rolling totals from a growing event volume.

|  | A — Scheduled batch computation | B — Continuous stream computation |
|---|---|---|
| **Wins when** | Hours of delay are acceptable and whole-dataset recomputation is manageable. | Seconds of freshness materially change decisions. |

| Dimension | Comparison |
|---|---|
| Latency | A waits for schedule and run; B updates incrementally. |
| Throughput | A scans in efficient blocks; B maintains continuously active keyed state. |
| Consistency | A uses a cutoff snapshot; B needs event-time and late-data semantics. |
| Complexity | A simplifies reruns; B needs checkpoints, state and sink guarantees. |
| Cost | A can use intermittent capacity; B runs continuously. |
| Failure behaviour | A failed jobs delay an entire refresh; B silent lag makes results stale. |

**Staff-level view:** A needs data pipeline skills; B needs stateful streaming operations.

---

## Storage Internals

### B-Tree vs LSM Storage

**Scenario:** A database must balance point reads, scans and sustained write ingestion.

|  | A — B-tree-based storage | B — LSM-based storage |
|---|---|---|
| **Wins when** | Read latency and ordered access dominate a moderate write load. | High write ingestion benefits from buffered sequential runs. |

| Dimension | Comparison |
|---|---|
| Latency | A often offers predictable lookup paths; B may check several levels. |
| Throughput | B batches writes; A updates tree pages and indexes. |
| Consistency | Neither chooses isolation or replication guarantees by itself. |
| Complexity | A tunes pages and caching; B tunes compaction and filters. |
| Cost | A incurs page-write I/O; B incurs compaction and space amplification. |
| Failure behaviour | A fragmentation or latch contention hurts; B compaction debt stalls writes. |

**Staff-level view:** A needs index/storage tuning; B needs compaction and amplification expertise.

---

## Data Modeling

### Normalize vs Denormalize

**Scenario:** Product and merchant information appear together on many screens.

|  | A — Normalized authoritative tables | B — Duplicated read-optimized records |
|---|---|---|
| **Wins when** | Update correctness and flexible relationships dominate. | Predictable read latency justifies maintained copies. |

| Dimension | Comparison |
|---|---|
| Latency | A may join at read time; B precomputes combinations. |
| Throughput | A minimizes update fanout; B reduces read work. |
| Consistency | A has one fact location; B needs a stale-copy policy. |
| Complexity | A has relational joins; B has propagation and rebuild logic. |
| Cost | A stores less duplication; B trades storage and writes for reads. |
| Failure behaviour | A joins can bottleneck; B partial updates make conflicting copies. |

**Staff-level view:** A needs schema/index design; B needs projection and reconciliation ownership.

---

### Document Model vs Relational Model

**Scenario:** A catalog has varied attributes plus shared merchant and category relationships.

|  | A — Document aggregates | B — Relational tables |
|---|---|---|
| **Wins when** | Most operations read and replace a natural bounded aggregate. | Cross-entity relations and constraints drive the workload. |

| Dimension | Comparison |
|---|---|
| Latency | A retrieves one aggregate directly; B uses joins or projections. |
| Throughput | A distributes aggregates; B distributes after explicit partition design. |
| Consistency | A commonly has document-local atomicity; B commonly supports multi-row constraints. |
| Complexity | A flexible fields need validation; B schema changes need coordination. |
| Cost | A duplicates embedded relationships; B pays joins and indexes. |
| Failure behaviour | A oversized aggregates become hot; B poorly indexed joins become expensive. |

**Staff-level view:** A needs aggregate-boundary discipline; B needs relational modeling expertise.

---

## Traffic Control

### Token Bucket vs Leaky Bucket

**Scenario:** An API permits bursts but an outbound vendor needs smooth traffic.

|  | A — Token bucket admission | B — Leaky bucket shaping |
|---|---|---|
| **Wins when** | Short bursts should pass immediately within a budget. | A downstream requires near-constant release rate. |

| Dimension | Comparison |
|---|---|
| Latency | A accepts bursts without waiting; B adds bounded queue delay. |
| Throughput | Both bound long-term rate; B smooths actual dispatch. |
| Consistency | Neither ensures a global quota unless state coordination matches scope. |
| Complexity | A tracks tokens and refill; B manages queue, scheduler and deadlines. |
| Cost | A stores small counters; B retains waiting requests or jobs. |
| Failure behaviour | A burst can overwhelm weak downstreams; B queues can exceed useful deadlines. |

**Staff-level view:** A needs atomic limiter logic; B needs scheduler and queue operations.

---

### Fixed vs Sliding Rate Window

**Scenario:** A public API advertises a rolling per-user quota.

|  | A — Fixed time window | B — Sliding window |
|---|---|---|
| **Wins when** | Low-cost approximate enforcement is sufficient. | Boundary bursts materially harm fairness or capacity. |

| Dimension | Comparison |
|---|---|
| Latency | Both can be fast; exact sliding logs do more state work. |
| Throughput | A uses one counter; B exact logs track accepted requests. |
| Consistency | A permits boundary bursts; B can enforce a true rolling window. |
| Complexity | A is simpler; B chooses exact logs or approximate counters. |
| Cost | A uses little state; B exact memory scales with traffic. |
| Failure behaviour | A synchronized resets spike load; B missing pruning leaks memory. |

**Staff-level view:** A is easy to operate; B needs approximation and cardinality discipline.

---

### Queue Excess vs Reject Excess

**Scenario:** A service receives a burst beyond current execution capacity.

|  | A — Bounded waiting queue | B — Early rejection |
|---|---|---|
| **Wins when** | Jobs remain useful after delay and callers can receive pending status. | Requests will expire before capacity becomes available. |

| Dimension | Comparison |
|---|---|
| Latency | A increases wait time; B fails fast. |
| Throughput | A smooths finite bursts; B preserves capacity for admitted work. |
| Consistency | A requires durable acceptance if loss is unacceptable; B accepts no new work. |
| Complexity | A needs age limits and cancellation; B needs admission policy and retry guidance. |
| Cost | A retains queued state; B loses or defers demand. |
| Failure behaviour | A backlog can become a brownout; B client retries can recreate overload. |

**Staff-level view:** A needs queue SLO ownership; B needs capacity-aware admission and client contracts.

---

### Local vs Global Rate Limit

**Scenario:** Requests from one customer arrive across many gateway instances.

|  | A — Per-instance limit | B — Coordinated shared limit |
|---|---|---|
| **Wins when** | Protecting each instance matters more than an exact aggregate quota. | The business contract requires a strict shared tenant allowance. |

| Dimension | Comparison |
|---|---|
| Latency | A has no remote hop; B consults shared or allocated quota state. |
| Throughput | A scales independently; B can bottleneck on hot tenant counters. |
| Consistency | A fleet allowance changes with instances; B can enforce shared accounting. |
| Complexity | A is simple; B needs atomicity, partitions and outage behavior. |
| Cost | A is cheap; B adds coordination traffic and a quota service. |
| Failure behaviour | A overshoots global intent; B service failure can reject healthy traffic. |

**Staff-level view:** B requires distributed quota operations and explicit fail-open/fail-closed policy.

---

## Reliability

### Retry vs Hedge

**Scenario:** A replicated read has occasional slow responses.

|  | A — Retry after failure or timeout | B — Delayed parallel hedge |
|---|---|---|
| **Wins when** | Failures are clear and spare capacity is limited. | Tail latency dominates and reads are safe to duplicate. |

| Dimension | Comparison |
|---|---|
| Latency | A waits for failure before another try; B can beat a slow original. |
| Throughput | A usually adds less overlap; B spends extra concurrent work. |
| Consistency | Both must preserve read-consistency requirements across replicas. |
| Complexity | A needs retry budget; B also needs duplicate cancellation and alternate routing. |
| Cost | B uses more request capacity to buy lower tails. |
| Failure behaviour | Both amplify overload; B is particularly risky without a hedge budget. |

**Staff-level view:** A needs error classification; B needs tail-distribution and replica-awareness expertise.

---

## Media Storage

### Object Storage vs Database Blobs

**Scenario:** A product stores large files with transactional ownership metadata.

|  | A — Object store plus metadata rows | B — Binary payloads in the database |
|---|---|---|
| **Wins when** | Files are large and direct transfer, CDN or lifecycle tiers matter. | Small payloads must share the same transactional backup and access path. |

| Dimension | Comparison |
|---|---|
| Latency | A fetches bytes separately; B can retrieve bytes with metadata. |
| Throughput | A scales blob traffic independently; B consumes database I/O and replication. |
| Consistency | A needs reconciliation across stores; B can commit metadata and bytes atomically. |
| Complexity | A manages object keys and lifecycle; B grows database and backups. |
| Cost | A usually suits bulk capacity; B increases database storage and restore work. |
| Failure behaviour | A orphaned blobs appear; B large files can crowd operational workloads. |

**Staff-level view:** A needs storage-policy expertise; B needs capacity and backup discipline.

---

### Direct Signed Upload vs API Proxy Upload

**Scenario:** Mobile clients upload large private files.

|  | A — Presigned direct storage upload | B — Upload through application servers |
|---|---|---|
| **Wins when** | Bandwidth offload and resumability matter at scale. | The API must inspect or transform the stream before storage. |

| Dimension | Comparison |
|---|---|
| Latency | A removes an application hop; B adds proxy processing. |
| Throughput | A uses storage's transfer capacity; B scales application ingress and egress. |
| Consistency | Both need authoritative completion and metadata state. |
| Complexity | A manages bearer capabilities and post-upload validation; B manages streaming limits. |
| Cost | A reduces app bandwidth; B pays compute and transfer through the API. |
| Failure behaviour | A leaked URLs permit scoped access; B large uploads can exhaust app resources. |

**Staff-level view:** A needs storage signing and lifecycle knowledge; B needs safe streaming operations.

---

## Media Delivery

### Short vs Long Video Segments

**Scenario:** A live player must balance latency against network efficiency.

|  | A — Short media segments | B — Long media segments |
|---|---|---|
| **Wins when** | Fast live response and frequent adaptation are essential. | Efficient delivery and a stable buffer matter more than low delay. |

| Dimension | Comparison |
|---|---|
| Latency | A lowers segment accumulation delay; B increases buffering granularity. |
| Throughput | A creates more requests; B amortizes request overhead. |
| Consistency | Both need aligned boundaries and consistent manifests. |
| Complexity | A requires tighter packaging and player tuning; B is operationally simpler. |
| Cost | A can increase CDN request costs; B may waste more downloaded bytes after a switch. |
| Failure behaviour | A is sensitive to jitter with tiny buffers; B recovers quality more slowly. |

**Staff-level view:** A needs live-media expertise; B suits less demanding delivery teams.

---

## Media Processing

### Precompute Renditions vs Just-In-Time Encoding

**Scenario:** A large video library has a long tail of rarely watched titles.

|  | A — Encode a ladder before publication | B — Generate renditions on demand |
|---|---|---|
| **Wins when** | Predictable playback startup and popular content dominate. | Many assets will never be watched in every format. |

| Dimension | Comparison |
|---|---|
| Latency | A serves ready segments; B first requests may wait for compute. |
| Throughput | A moves work off playback; B must scale with viewing demand. |
| Consistency | Both need immutable versioned outputs and publication checks. |
| Complexity | A manages batch pipelines; B manages cache fills and demand deduplication. |
| Cost | A spends storage and unused encoding; B spends burst compute and orchestration. |
| Failure behaviour | A failed encoding delays publication; B cold-demand storms hit encoders. |

**Staff-level view:** B needs media compute scheduling and cold-start mitigation expertise.

---

## Global Architecture

### Active-Passive vs Active-Active Regions

**Scenario:** A SaaS platform needs to survive a region outage.

|  | A — Standby region with controlled promotion | B — Several actively serving regions |
|---|---|---|
| **Wins when** | Simpler write authority and recoverability meet the SLA. | Local service and regional capacity use justify cross-region coordination. |

| Dimension | Comparison |
|---|---|
| Latency | A remote users may travel farther; B can serve regionally. |
| Throughput | A standby may be idle; B spreads active traffic. |
| Consistency | A still needs promotion safety; B requires explicit write topology and conflict rules. |
| Complexity | A emphasizes failover rehearsal; B adds global routing and steady-state complexity. |
| Cost | A pays standby capacity; B pays replicated active capacity and network. |
| Failure behaviour | A cold failover can surprise; B global config failures affect all regions. |

**Staff-level view:** A needs disaster recovery skills; B needs ongoing multi-region operations.

---

## Operations

### Managed Service vs Self-Hosted Infrastructure

**Scenario:** A team needs a production message broker without existing broker specialists.

|  | A — Managed offering | B — Self-hosted deployment |
|---|---|---|
| **Wins when** | Time, staffing and routine availability operations dominate. | Control, unusual configuration or scale economics justify ownership. |

| Dimension | Comparison |
|---|---|
| Latency | A adds provider constraints; B permits deeper topology tuning. |
| Throughput | Both depend on configuration and workload, not ownership alone. |
| Consistency | Compare documented durability and failover guarantees directly. |
| Complexity | A delegates routine operations; B owns upgrades, backups and recovery. |
| Cost | A charges service premiums; B includes staffing and incident costs. |
| Failure behaviour | A has provider outages and limits; B exposes operator error and maintenance gaps. |

**Staff-level view:** A needs vendor and capacity literacy; B needs dedicated operational expertise.

---

## Compute

### Serverless Functions vs Long-Running Containers

**Scenario:** An API has irregular demand and some long-lived streaming connections.

|  | A — Event-driven functions | B — Long-running container services |
|---|---|---|
| **Wins when** | Short stateless requests have intermittent or unpredictable demand. | Connections, execution duration or custom runtime control dominate. |

| Dimension | Comparison |
|---|---|
| Latency | A may have cold starts; B stays warm if provisioned. |
| Throughput | A scales within platform quotas; B scales fleets with explicit capacity. |
| Consistency | Both require external durable state and repeat-safe processing. |
| Complexity | A delegates hosts; B controls runtime, lifecycle and networking. |
| Cost | A can suit low utilization; B can suit steady high utilization. |
| Failure behaviour | A hits concurrency/runtime limits; B risks underprovisioning and host failures. |

**Staff-level view:** A needs platform-limit knowledge; B needs deployment and capacity operations.

---

## Analytics

### Exact Count vs Approximate Sketch

**Scenario:** A dashboard counts distinct users across billions of events.

|  | A — Exact set or aggregation | B — Approximate cardinality sketch |
|---|---|---|
| **Wins when** | Billing, entitlements or small datasets demand exact answers. | Trend monitoring accepts a known error bound to save resources. |

| Dimension | Comparison |
|---|---|
| Latency | A scans or retains more state; B merges compact summaries. |
| Throughput | B supports large-scale incremental aggregation with bounded sketch state. |
| Consistency | A provides exact count under correct dedupe; B has statistical error. |
| Complexity | A handles full identities; B needs precision and merge semantics. |
| Cost | A uses substantial memory/storage; B trades accuracy for compactness. |
| Failure behaviour | A can exhaust resources; B misuse of error bounds misleads decisions. |

**Staff-level view:** A needs data correctness skills; B also needs statistical literacy.

---

## Scheduling

### FIFO vs Priority Scheduling

**Scenario:** Interactive tasks share workers with long batch jobs.

|  | A — First-in-first-out queue | B — Priority-aware scheduling |
|---|---|---|
| **Wins when** | Fair arrival order and similar job urgency are appropriate. | Different deadlines or service tiers require preferential capacity. |

| Dimension | Comparison |
|---|---|
| Latency | A urgent jobs wait behind backlog; B improves urgent latency. |
| Throughput | Neither adds capacity; B changes which work completes first. |
| Consistency | Both require duplicate-safe claims; priority is not an ordering guarantee. |
| Complexity | A is simple; B needs aging, reservations and anti-abuse rules. |
| Cost | B may reserve idle capacity to meet urgent targets. |
| Failure behaviour | A suffers head-of-line delay; B can starve low-priority jobs. |

**Staff-level view:** B needs workload classification and fairness/SLO ownership.

---

### Process Timer vs Durable Timer

**Scenario:** A reminder should execute after a delay even if workers restart.

|  | A — In-memory timer | B — Persisted due time or workflow timer |
|---|---|---|
| **Wins when** | The action is short-lived and losing it is harmless. | The action must survive process or host failure. |

| Dimension | Comparison |
|---|---|
| Latency | A fires locally with little overhead; B scheduling introduces dispatch delay. |
| Throughput | A consumes process memory; B scales through indexed due-time batches. |
| Consistency | A offers no durable promise; B still needs repeat-safe execution. |
| Complexity | A is easy; B needs claim, catch-up and clock policies. |
| Cost | A has tiny setup cost; B stores and scans schedule state. |
| Failure behaviour | A disappears on restart; B recovery can create a due-job burst. |

**Staff-level view:** A needs basic runtime knowledge; B needs scheduler and backlog operations.

---

## Networking

### Compress Payloads vs Send Uncompressed

**Scenario:** An API returns large repetitive JSON responses across costly networks.

|  | A — Compress payloads | B — Send raw payloads |
|---|---|---|
| **Wins when** | Payloads are large and bandwidth or mobile transfer time dominates. | Payloads are tiny or CPU is the tightest bottleneck. |

| Dimension | Comparison |
|---|---|
| Latency | A adds CPU but can reduce transfer delay; B avoids encoding work. |
| Throughput | A trades CPU throughput for network throughput. |
| Consistency | Compression changes representation, not application consistency. |
| Complexity | A needs negotiation and decompression limits; B is simpler. |
| Cost | A saves egress bytes; B saves compute. |
| Failure behaviour | A decompression bombs or shared-secret compression risks need controls; B large responses saturate links. |

**Staff-level view:** A needs profiling and safe codec configuration; B still needs payload budgets.

---
