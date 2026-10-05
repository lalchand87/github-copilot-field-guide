# Pattern Recognition — Hear the pattern in the problem

[← System Design index](README.md)

> 70 trigger phrases → building blocks (210 scenarios in the app: 3 rounds × 70 triggers).

Pattern Recognition trains you to go from a **trigger phrase** to the **building block**. In the app it runs three rounds over all 70 triggers:

1. **Direct** — which building block does this describe?
2. **10× growth** — traffic just grew 10×. Which building block still carries the design?
3. **Degraded dependency** — one regional dependency is slow and sometimes failing. Which building block do you lean on?

Cover the middle column, read the trigger, name the pattern — then defend its role in one sentence. Each pattern name links to its full write-up.

## Caching

| When you hear… | Pattern | What it does |
|---|---|---|
| Reads dominate writes · the same keys are requested repeatedly · bounded staleness is acceptable · the database is the bottleneck | [Cache Aside](02-patterns/part-1.md#cache-aside) | The application owns the cache: it looks there first, falls back to the database on a miss, and fills the cache itself. |
| Many services read the same data · fill logic keeps being duplicated · you want one place to fix a caching bug | [Read Through Cache](02-patterns/part-1.md#read-through-cache) | The cache itself fetches from the source on a miss, so every caller sees one lookup and the fill logic lives in one place. |
| Read-after-write must be correct · the writer immediately reads back · stale reads after an update are unacceptable | [Write Through Cache](02-patterns/part-1.md#write-through-cache) | Every write goes to the cache and the store together, so a value is never in the cache unless it is already durable. |
| Write volume exceeds what the store can absorb · many writes touch the same key · some loss on crash is acceptable | [Write Behind Cache](02-patterns/part-1.md#write-behind-cache) | Writes land in the cache and are flushed to the store asynchronously, trading durability for absorbing bursts the store cannot take. |
| Data changed but users still see the old value · staleness has to be bounded · one edit must appear everywhere at once | [Cache Invalidation](02-patterns/part-1.md#cache-invalidation) | Deciding when a cached copy stops being usable — by expiry, by explicit removal, or by versioning the key so the old copy is never asked for. |
| A latency spike every time the cache expires · bounded staleness is fine · the source cannot take a synchronised refill | [Stale While Revalidate](02-patterns/part-1.md#stale-while-revalidate) | Serve the expired copy immediately and refresh it in the background, so nobody waits at expiry and the source sees one refill instead of a stampede. |
| A hot key expiring takes the database down · duplicate identical queries in flight · the miss path is expensive | [Single Flight](02-patterns/part-1.md#single-flight) | Collapse concurrent requests for the same missing value into one execution and share the result, so a popular key costs one fetch rather than hundreds. |

## Data Architecture

| When you hear… | Pattern | What it does |
|---|---|---|
| The write schema is fighting the query patterns · reads and writes scale differently · one table serves ten incompatible views | [CQRS](02-patterns/part-1.md#cqrs) | Separate the model that accepts changes from the models that answer questions, so each can be shaped for its own job instead of compromising between them. |
| You need to know how it got this way · an audit trail is a requirement · past state must be reconstructible | [Event Sourcing](02-patterns/part-1.md#event-sourcing) | Store the sequence of changes rather than the current state, so history is the source of truth and any view of it can be rebuilt. |
| A search index or cache keeps drifting from the database · dual writes keep going wrong · you need change events from a system you cannot modify | [Change Data Capture](02-patterns/part-1.md#change-data-capture) | Read the database's own replication log to emit a stream of row changes, so downstream systems stay in step without the application publishing anything. |
| The same expensive aggregate is computed repeatedly · dashboard queries take seconds · joins dominate read latency | [Materialized View](02-patterns/part-1.md#materialized-view) | Precompute and store the answer to an expensive query, so reads become a lookup and the cost moves to write time. |
| A backup restored to an impossible state · an export where totals do not reconcile · a migration that copied a half-finished transaction | [Consistent Snapshot](02-patterns/part-1.md#consistent-snapshot) | Read a set of data as it existed at one instant, so a backup, export or migration never mixes rows from before and after a concurrent write. |
| A field must be added, renamed or removed · producers and consumers deploy independently · stored data outlives the code that wrote it | [Schema Evolution](02-patterns/part-1.md#schema-evolution) | Change the shape of data over time while old and new readers and writers coexist, by keeping every step compatible in both directions. |

## Transactions

| When you hear… | Pattern | What it does |
|---|---|---|
| One business operation spans several services · each owns its own database · a distributed transaction is not available or not acceptable | [Saga](02-patterns/part-1.md#saga) | Break a business transaction into local transactions with compensating actions, so a multi-service operation can be undone without a distributed lock. |
| Several systems must commit atomically · partial application is unacceptable · participants support a prepare phase | [Two Phase Commit](02-patterns/part-1.md#two-phase-commit) | A coordinator asks every participant to prepare, and only when all agree does it tell them to commit — giving atomicity across systems at the cost of blocking. |

## Messaging

| When you hear… | Pattern | What it does |
|---|---|---|
| A state change must always produce a message · dual writes are silently losing events · atomicity across database and broker is needed | [Transactional Outbox](02-patterns/part-1.md#transactional-outbox) | Write the message into the same database transaction as the state change, then publish it from that table, so the two can never disagree. |
| At-least-once delivery · handlers with side effects that must not repeat · duplicates observed downstream | [Inbox Deduplication](02-patterns/part-1.md#inbox-deduplication) | Record every message identifier the consumer has processed, in the same transaction as its effects, so a redelivered message is recognised and skipped. |
| A request triggers slow work · load arrives in bursts · producers and consumers should scale independently | [Work Queue](02-patterns/part-1.md#work-queue) | Put units of work in a durable queue and let a pool of workers pull from it, so producers never wait and capacity scales by adding consumers. |
| Several consumers need the same stream · ordering per entity matters · events must be replayable after a bug or a new consumer | [Partitioned Log](02-patterns/part-1.md#partitioned-log) | Append events to an ordered, replayable log split into partitions, so consumers read at their own pace with ordering guaranteed per key. |

## Reliability

| When you hear… | Pattern | What it does |
|---|---|---|
| Requests with side effects over an unreliable network · clients that retry · a timeout that might have succeeded | [Idempotency Key](02-patterns/part-1.md#idempotency-key) | The client sends a unique key with a request so the server can recognise a retry and return the original result instead of performing the action again. |
| Transient network and dependency failures · retry storms observed during incidents · synchronised client behaviour | [Retry With Jitter](02-patterns/part-1.md#retry-with-jitter) | Retry failed calls with exponentially growing, randomised delays, so transient faults recover without the retries themselves becoming the outage. |
| A dependency is down and every call waits for a timeout · threads exhausted by a failing service · cascading failure | [Circuit Breaker](02-patterns/part-1.md#circuit-breaker) | Track failures against a dependency and stop calling it once they cross a threshold, failing fast until a probe shows it has recovered. |
| One slow dependency takes down unrelated features · a single tenant degrades everyone · shared thread or connection pools | [Bulkhead](02-patterns/part-1.md#bulkhead) | Give each dependency or tenant its own bounded pool of resources, so one of them saturating cannot consume the capacity the others need. |
| Tail latency far worse than median · occasional slow replicas · latency matters more than a small load increase | [Hedged Requests](02-patterns/part-1.md#hedged-requests) | Send a second copy of a request to another replica once the first is slower than usual, and take whichever answer arrives first, cutting tail latency. |

## Coordination

| When you hear… | Pattern | What it does |
|---|---|---|
| Work that must happen exactly once across a fleet · a coordinator role that cannot be duplicated · a singleton that must survive its host dying | [Leader Election](02-patterns/part-2.md#leader-election) | Have a group of identical processes agree on one of them to perform singleton work, with automatic replacement when that one fails. |
| Two processes must not act on the same resource simultaneously · a shared external resource with no locking of its own · coordination across machines | [Distributed Lock](02-patterns/part-2.md#distributed-lock) | Grant exclusive access to a resource across processes using a shared store, with an expiry so a crashed holder cannot block it forever. |
| A holder can die without releasing · ownership must recover without operator action · caches or claims that must not outlive their validity | [Leases](02-patterns/part-2.md#leases) | Grant a right for a bounded period rather than indefinitely, so a holder that dies silently loses it automatically without anyone having to detect the failure. |
| Failed nodes must be removed from rotation · work must be reassigned · membership has to reflect reality automatically | [Heartbeat Failure Detection](02-patterns/part-2.md#heartbeat-failure-detection) | Have nodes send periodic signals and declare a node failed when they stop, accepting that partitions and pauses make the verdict a guess. |
| Hundreds or thousands of nodes · no central coordinator wanted · membership must survive partitions and keep working | [Gossip Membership](02-patterns/part-2.md#gossip-membership) | Each node exchanges state with a few random peers, so membership and failure information spreads through the cluster without any central registry. |

## Partitioning

| When you hear… | Pattern | What it does |
|---|---|---|
| A node pool that changes size · modulo hashing causing mass remapping · distributed caches and partitioned stores | [Consistent Hashing](02-patterns/part-2.md#consistent-hashing) | Map keys and nodes onto the same ring so adding or removing a node moves only its share of keys instead of remapping everything. |
| One database cannot hold the data or serve the writes · vertical scaling exhausted · a natural partition key exists | [Sharding](02-patterns/part-2.md#sharding) | Split data across independent databases by a chosen key, so capacity scales horizontally at the cost of cross-shard operations. |
| One partition saturated while others idle · a celebrity account or viral item · a counter every request touches | [Hot Key Mitigation](02-patterns/part-2.md#hot-key-mitigation) | Spread or absorb traffic for a single dominant key, since partitioning by hash always sends one key to exactly one place. |

## Replication

| When you hear… | Pattern | What it does |
|---|---|---|
| Replicated data that must not serve stale reads · availability required despite replica failures · tunable consistency needed per operation | [Quorum Replication](02-patterns/part-2.md#quorum-replication) | Write to a majority of replicas and read from a majority, so any read is guaranteed to see the most recent write without contacting them all. |
| Users spread across continents · a region outage must not be an outage · data residency requirements | [Geographic Replication](02-patterns/part-2.md#geographic-replication) | Keep copies of data in several regions so reads are local and a region failure is survivable, accepting that writes cannot be both fast and globally consistent. |
| Replicas drift after failures or dropped writes · eventual consistency that must actually converge · repair without a separate process | [Read Repair](02-patterns/part-2.md#read-repair) | Detect stale replicas during ordinary reads and update them, so the act of reading gradually repairs divergence at no extra cost. |
| Cold data diverging unnoticed · replicas rebuilt or restored from backup · a guarantee that divergence is bounded | [Anti Entropy](02-patterns/part-2.md#anti-entropy) | Periodically compare replicas in the background and reconcile differences, so divergence is bounded even for data nobody reads. |

## Feeds

| When you hear… | Pattern | What it does |
|---|---|---|
| Feed reads vastly outnumber writes · feed latency must be minimal · followers per author are bounded | [Fanout On Write](02-patterns/part-2.md#fanout-on-write) | Push each new item into every follower's precomputed feed at write time, so reading a feed is a single cheap lookup. |
| Authors with enormous follower counts · feeds read rarely relative to posting · write amplification unacceptable | [Fanout On Read](02-patterns/part-2.md#fanout-on-read) | Store each post once and assemble a feed by querying the accounts a viewer follows at read time, keeping writes trivial at the cost of expensive reads. |
| A follower distribution with a long tail · pure fanout failing on celebrities or on heavy followers · a production feed at scale | [Hybrid Fanout](02-patterns/part-2.md#hybrid-fanout) | Precompute feeds for ordinary accounts and merge large accounts in at read time, so neither followers nor followings can make an operation unbounded. |

## Traffic Control

| When you hear… | Pattern | What it does |
|---|---|---|
| A sustained rate limit that must still permit bursts · API quotas · protecting a backend from spikes | [Token Bucket](02-patterns/part-2.md#token-bucket) | Accumulate tokens at a fixed rate up to a cap, spending one per request, so a steady rate is enforced while short bursts are still allowed. |
| A downstream system that cannot absorb bursts · a strict output rate required · smoothing matters more than latency | [Leaky Bucket](02-patterns/part-2.md#leaky-bucket) | Queue incoming requests and release them at a constant rate, converting bursty arrivals into a perfectly smooth output stream. |
| A rate limit needed quickly · minimal state acceptable · approximate enforcement sufficient | [Fixed Window Rate Limit](02-patterns/part-2.md#fixed-window-rate-limit) | Count requests within aligned time windows and reject beyond the limit, the simplest possible limiter and the one with a known boundary flaw. |
| Fixed-window boundary bursts causing real load · limits that must be accurate at every instant · third-party limits enforced this way | [Sliding Window Rate Limit](02-patterns/part-2.md#sliding-window-rate-limit) | Evaluate the limit over a window that moves with the present moment, removing the boundary burst that fixed windows permit. |
| Queues growing without bound · memory pressure from buffered work · a producer faster than its consumer | [Backpressure](02-patterns/part-2.md#backpressure) | Signal upstream to slow down when a system cannot keep up, so demand is reduced at the source instead of accumulating in queues. |
| Demand exceeding capacity with no way to slow the source · degradation affecting all requests · protecting a service from collapse | [Load Shedding](02-patterns/part-2.md#load-shedding) | Deliberately reject a portion of traffic when overloaded, so the remainder is served properly instead of everything failing together. |

## Service Architecture

| When you hear… | Pattern | What it does |
|---|---|---|
| Many services exposed directly to clients · cross-cutting concerns duplicated in each · clients coupled to internal topology | [API Gateway](02-patterns/part-2.md#api-gateway) | Put one entry point in front of many services to handle authentication, routing, limiting and observability once instead of everywhere. |
| One API serving clients with conflicting needs · mobile making many round trips · over-fetching on constrained connections | [Backend For Frontend](02-patterns/part-2.md#backend-for-frontend) | Give each client type its own backend that shapes data specifically for it, so no single API has to compromise between very different consumers. |
| Instances that come and go · autoscaling and rolling deploys · hardcoded addresses breaking on every change | [Service Discovery](02-patterns/part-2.md#service-discovery) | Let services find each other through a registry of currently healthy instances, so addresses can change without reconfiguring callers. |
| Several languages needing the same infrastructure behaviour · libraries requiring redeploys to upgrade · a service mesh being adopted | [Sidecar](02-patterns/part-2.md#sidecar) | Run shared infrastructure concerns in a separate process alongside each service instance, so every language gets the same behaviour without a library. |
| A screen needing data from several services · clients orchestrating service calls · chatty client-server interaction | [Aggregator](02-patterns/part-2.md#aggregator) | Have one service call several others and combine their responses, so a client makes one request instead of coordinating many. |
| A query that must consult every shard · search across a partitioned index · parallel computation over distributed data | [Scatter Gather](02-patterns/part-2.md#scatter-gather) | Send the same request to many nodes in parallel and combine their partial answers, so work that spans a dataset completes in the time of one node. |

## Data Processing

| When you hear… | Pattern | What it does |
|---|---|---|
| Large volumes to process efficiently · results that can be hours old · computations that must be reproducible | [Batch Processing](02-patterns/part-2.md#batch-processing) | Process a bounded set of data as one job on a schedule, trading freshness for throughput, simplicity and the ability to reprocess. |
| Results needed in seconds · continuously updated aggregates · detection that must happen while it still matters | [Stream Processing](02-patterns/part-2.md#stream-processing) | Compute continuously over unbounded event streams, maintaining state and emitting results within seconds of the events that caused them. |

## Realtime Delivery

| When you hear… | Pattern | What it does |
|---|---|---|
| Bidirectional messaging · sub-second updates in both directions · high message rates where per-request overhead matters | [WebSocket](02-patterns/part-3.md#websocket) | Hold a persistent bidirectional connection so either side can send at any time, with no polling and no per-message request overhead. |
| Push needed through restrictive networks · infrastructure that only understands request-response · a fallback for persistent connections | [Long Polling](02-patterns/part-3.md#long-polling) | Hold a request open until data is available or a timeout expires, giving push-like latency using only ordinary HTTP requests. |
| One-directional server updates · push without the weight of WebSockets · standard HTTP infrastructure preferred | [Server Sent Events](02-patterns/part-3.md#server-sent-events) | Stream updates from server to client over one long-lived HTTP response, with reconnection and resumption handled by the browser. |

## Media Delivery

| When you hear… | Pattern | What it does |
|---|---|---|
| Users far from the origin · static assets and media dominating traffic · origin bandwidth or load becoming the constraint | [CDN](02-patterns/part-3.md#cdn) | Serve content from caches near users, so most requests never reach the origin and distance stops dominating latency. |
| Video delivery to varied connections · buffering harming the experience · mobile networks whose capacity changes constantly | [Adaptive Bitrate Streaming](02-patterns/part-3.md#adaptive-bitrate-streaming) | Encode video at several qualities, split it into short segments, and let the player switch between them as available bandwidth changes. |

## Media Storage

| When you hear… | Pattern | What it does |
|---|---|---|
| Large files outgrowing a database · media, backups and logs · storage that must scale without capacity planning | [Object Storage](02-patterns/part-3.md#object-storage) | Store immutable blobs addressed by key in a flat namespace, trading filesystem semantics for effectively unlimited, cheap, durable capacity. |
| Files large enough that a single request is fragile · uploads failing near completion · throughput limited by one connection | [Multipart Upload](02-patterns/part-3.md#multipart-upload) | Split a large upload into independent parts that can be sent in parallel and retried individually, then assemble them server-side. |
| Uploads or downloads proxied through application servers · large files consuming API capacity · clients needing storage access without credentials | [Presigned URL](02-patterns/part-3.md#presigned-url) | Issue a time-limited signed URL that grants one specific operation, so clients read or write storage directly without holding credentials. |

## Storage Internals

| When you hear… | Pattern | What it does |
|---|---|---|
| Durability required without paying random write cost · crash recovery to a consistent state · the foundation of transactional storage | [Write Ahead Log](02-patterns/part-3.md#write-ahead-log) | Append every change to a sequential log and make it durable before applying it, so a crash can be recovered by replaying what was recorded. |
| Long-running reads blocking writes · lock contention between analytics and transactions · snapshot isolation required | [MVCC](02-patterns/part-3.md#mvcc) | Keep multiple versions of each row so readers see a consistent snapshot without blocking writers, and writers never block readers. |
| Expensive lookups for keys that usually do not exist · memory too small for a full index · disk or network reads worth avoiding | [Bloom Filter](02-patterns/part-3.md#bloom-filter) | A compact probabilistic structure that answers definitely-not-present or possibly-present, letting a system skip expensive lookups for keys it does not have. |
| Write-heavy workloads · random writes limiting throughput · time-series, logs and event data | [LSM Tree](02-patterns/part-3.md#lsm-tree) | Buffer writes in memory, flush them as sorted immutable files, and merge those files in the background, turning random writes into sequential ones. |

## Search

| When you hear… | Pattern | What it does |
|---|---|---|
| Text search over many documents · scans too slow for interactive queries · relevance ranking required | [Inverted Index](02-patterns/part-3.md#inverted-index) | Map each term to the documents containing it, so text search becomes a lookup and intersection rather than a scan. |
| Find everything within a radius · nearest neighbour queries · maps, delivery and location-based matching | [Spatial Index](02-patterns/part-3.md#spatial-index) | Index locations so that proximity queries examine only nearby candidates rather than scanning everything. |

## Scheduling

| When you hear… | Pattern | What it does |
|---|---|---|
| Mixed workloads with different urgency · bulk work delaying interactive work · capacity insufficient for everything at once | [Priority Queue](02-patterns/part-3.md#priority-queue) | Order pending work by importance rather than arrival, so limited capacity is spent on what matters most first. |
| Retries with backoff · reminders and timeouts · anything that must happen later rather than now | [Delay Queue](02-patterns/part-3.md#delay-queue) | Schedule work to become available at a future time, so retries, reminders and deferred actions need no polling loop. |
| Multi-step business processes · operations spanning minutes to months · orchestration that must survive deploys | [Durable Workflow](02-patterns/part-3.md#durable-workflow) | Persist each step's completion so a multi-step process survives crashes, restarts and long waits without losing its place. |
