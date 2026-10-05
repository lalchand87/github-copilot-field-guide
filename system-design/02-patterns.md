# Pattern Gym — Learn to hear the pattern in the problem

> 70 patterns in 18 categories, each led by its trigger phrase ("When you hear…"): the problem, mental model, how it works (step-by-step request flows), technologies, trade-offs, failure modes, where it is used, how to present it in an interview, a practice drill and related patterns.

## Contents

- **Caching** (7): [Cache Aside](#cache-aside) · [Read Through Cache](#read-through-cache) · [Write Through Cache](#write-through-cache) · [Write Behind Cache](#write-behind-cache) · [Cache Invalidation](#cache-invalidation) · [Stale While Revalidate](#stale-while-revalidate) · [Single Flight](#single-flight)
- **Data Architecture** (6): [CQRS](#cqrs) · [Event Sourcing](#event-sourcing) · [Change Data Capture](#change-data-capture) · [Materialized View](#materialized-view) · [Consistent Snapshot](#consistent-snapshot) · [Schema Evolution](#schema-evolution)
- **Transactions** (2): [Saga](#saga) · [Two Phase Commit](#two-phase-commit)
- **Messaging** (4): [Transactional Outbox](#transactional-outbox) · [Inbox Deduplication](#inbox-deduplication) · [Work Queue](#work-queue) · [Partitioned Log](#partitioned-log)
- **Reliability** (5): [Idempotency Key](#idempotency-key) · [Retry With Jitter](#retry-with-jitter) · [Circuit Breaker](#circuit-breaker) · [Bulkhead](#bulkhead) · [Hedged Requests](#hedged-requests)
- **Coordination** (5): [Leader Election](#leader-election) · [Distributed Lock](#distributed-lock) · [Leases](#leases) · [Heartbeat Failure Detection](#heartbeat-failure-detection) · [Gossip Membership](#gossip-membership)
- **Partitioning** (3): [Consistent Hashing](#consistent-hashing) · [Sharding](#sharding) · [Hot Key Mitigation](#hot-key-mitigation)
- **Replication** (4): [Quorum Replication](#quorum-replication) · [Geographic Replication](#geographic-replication) · [Read Repair](#read-repair) · [Anti Entropy](#anti-entropy)
- **Feeds** (3): [Fanout On Write](#fanout-on-write) · [Fanout On Read](#fanout-on-read) · [Hybrid Fanout](#hybrid-fanout)
- **Traffic Control** (6): [Token Bucket](#token-bucket) · [Leaky Bucket](#leaky-bucket) · [Fixed Window Rate Limit](#fixed-window-rate-limit) · [Sliding Window Rate Limit](#sliding-window-rate-limit) · [Backpressure](#backpressure) · [Load Shedding](#load-shedding)
- **Service Architecture** (6): [API Gateway](#api-gateway) · [Backend For Frontend](#backend-for-frontend) · [Service Discovery](#service-discovery) · [Sidecar](#sidecar) · [Aggregator](#aggregator) · [Scatter Gather](#scatter-gather)
- **Data Processing** (2): [Batch Processing](#batch-processing) · [Stream Processing](#stream-processing)
- **Realtime Delivery** (3): [WebSocket](#websocket) · [Long Polling](#long-polling) · [Server Sent Events](#server-sent-events)
- **Media Delivery** (2): [CDN](#cdn) · [Adaptive Bitrate Streaming](#adaptive-bitrate-streaming)
- **Media Storage** (3): [Object Storage](#object-storage) · [Multipart Upload](#multipart-upload) · [Presigned URL](#presigned-url)
- **Storage Internals** (4): [Write Ahead Log](#write-ahead-log) · [MVCC](#mvcc) · [Bloom Filter](#bloom-filter) · [LSM Tree](#lsm-tree)
- **Search** (2): [Inverted Index](#inverted-index) · [Spatial Index](#spatial-index)
- **Scheduling** (3): [Priority Queue](#priority-queue) · [Delay Queue](#delay-queue) · [Durable Workflow](#durable-workflow)

## Caching

### Cache Aside

*The application owns the cache: it looks there first, falls back to the database on a miss, and fills the cache itself.*

> **When you hear…** Reads dominate writes · the same keys are requested repeatedly · bounded staleness is acceptable · the database is the bottleneck

**Flow:** `Client` → `Application` → `Cache lookup` → `Database on miss` → `Cache fill`

**The problem**

A product page is read fifty thousand times a second and changes twice a day. Every one of those reads becomes a database query that returns the same rows, so the database is sized for a load that is almost entirely redundant — and it is the hardest and most expensive tier to scale.

The reads are also not evenly distributed. A small number of items attract most of the traffic, which means a small number of rows are being fetched over and over while the rest of the table is barely touched.

> **Put the answer nearer the caller than the source of truth**  
> If the same question keeps being asked and the answer changes rarely, storing the answer somewhere cheap and fast removes almost all the work. Cache aside is the variant where the application does this itself rather than delegating it to a library or the store — which is what makes it simple to reason about and easy to get subtly wrong.

**Mental model**

The cache is a shortcut, not a component the read path depends on. The application asks the cache; if the answer is there it returns it, and if not it goes to the database, returns that, and leaves a copy behind for next time.

1. **Look aside** — Check the cache first. A hit costs a network round trip to memory instead of a query.
2. **Fall through** — A miss is not an error. Read the authority and carry on.
3. **Fill** — Write the value into the cache with a lifetime, so the next reader gets a hit.
4. **Invalidate on write** — When the underlying row changes, remove the cached copy rather than trying to update it.
5. **Expire** — A TTL bounds how wrong a forgotten invalidation can leave you.

> **The cache must never become load-bearing**  
> Because the application handles misses itself, a cache outage should degrade to slower reads rather than failed ones. That property is easy to lose: a fill that throws on a cache error, or a read path that treats a cache timeout as a request failure, turns an optional accelerator into a hard dependency — and then the cache going down takes the product down with it.

**How it works**

**Diagram: Cache aside: the three paths**

Components: Client · Application · Cache (Redis) · Database (authority)

The hit path never reaches the database. The miss path pays for the lookup **and** the query, which is why a low hit ratio is worse than no cache at all. The write path deletes rather than updates — step through it and watch what the cache is left holding.

*Read · hit*

1. Client: **A read arrives** — The same product page that fifty thousand other callers are asking for right now.
2. Client → Application: **The application owns the decision** — Nothing in the client knows a cache exists.
3. Application → Cache (Redis): **Look aside first: `cache.get(key)`** — A memory lookup over the network, not a query.
4. Cache (Redis) → Application: **Hit — the value is present** — The database is never consulted on this path. *(~1 ms)*
5. Application → Client: **Return the cached value** — At a 95% hit ratio this is what 19 out of 20 reads look like. *(~1 ms total)*

*Read · miss*

1. Client: **A read arrives** — A cold key, or one whose TTL has just expired.
2. Client → Application: **The application owns the decision**
3. Application → Cache (Redis): **Look aside first: `cache.get(key)`**
4. Cache (Redis) → Application: **Miss — the key is absent** — A miss is not an error. It is the fall-through path.
5. Application → Database (authority): **Fall through to the authority** — `db.query(key)` — the work the cache existed to avoid. *(~20 ms)*
6. Database (authority) → Application: **The rows come back** — This is the only place the true value is read.
7. Application → Cache (Redis): **Fill: `cache.set(key, value, ttl)`** — The application fills the cache itself — that is what makes this *aside* rather than read-through.
8. Application → Client: **Return the value** — A miss costs the lookup plus the query. Below roughly a 70% hit ratio, that tax outweighs the saving. *(~25 ms total)*

*Write · invalidate*

1. Client: **A write arrives** — The price on the product changes.
2. Client → Application: **The application owns both stores**
3. Application → Database (authority): **Write the authority first** — The database is the only thing that must be correct.
4. Application → Cache (Redis): ****Delete** the key — do not update it** — Two concurrent writers both updating would leave whichever reached the cache last in place, which may not be whichever reached the database last. A delete is idempotent and cannot invert ordering.
5. Application → Client: **Acknowledge the write** — The next reader takes a miss and refills from whatever the database actually holds. If this delete were lost, the TTL is the only thing that ever corrects it.

**The read and write paths**

```text
READ
  value = cache.get(key)
  if value is present:
      return value                      <- the hit path, ~1 ms
  value = db.query(key)                 <- the miss path
  cache.set(key, value, ttl)
  return value

WRITE
  db.update(row)
  cache.delete(key)                     <- delete, do not update

WHY DELETE RATHER THAN UPDATE
  two concurrent writers both update the cache
  -> whichever write reaches the cache last wins,
     which may not be whichever reached the database last
  -> the cache now disagrees with the authority, and
     nothing will correct it until the TTL expires
  deleting is idempotent and cannot invert order:
  the next reader refills from whatever the database
  actually holds.

WHY THE TTL STILL MATTERS
  every invalidation is a message that can be lost:
  a crash between the database write and the delete
  leaves a stale entry with no one to remove it
  -> the TTL is the backstop that bounds how long
     that mistake can survive
```

1. **Delete on write, never update** — An update races with other writers and can leave the cache disagreeing with the database permanently; a delete cannot invert ordering.
2. **Always set a TTL** — Invalidation is a best-effort message. The TTL is what bounds the damage when one is lost.
3. **Treat cache errors as misses** — A cache timeout should fall through to the database, not fail the request, or the accelerator becomes a dependency.
4. **Cache the shape you serve** — Caching a rendered response avoids re-doing the assembly work; caching raw rows only saves the query.
5. **Key on everything that changes the value** — A key that omits a parameter serves one caller's answer to another.
6. **Guard the miss path against stampedes** — A popular key expiring sends every concurrent reader to the database at once.

**Worked example: a product catalogue**

```text
TRAFFIC        50,000 reads/s
HIT RATIO      95%
ITEM SIZE      2 KB

DATABASE LOAD
  without cache   50,000 queries/s
  with cache      2,500 queries/s      <- 5% of reads miss
  -> the database is sized for 2,500 q/s, not 50,000

CACHE FOOTPRINT
  1,000,000 items x 2 KB = ~2 GB
  -> comfortably a single memory node, replicated

WHAT A COLD CACHE COSTS
  restart with an empty cache
  -> 100% miss for the first moments
  -> 50,000 q/s at a database provisioned for 2,500
  -> the database falls over, and it falls over
     precisely when you were recovering
  this is why cache loss is a capacity event, not a
  performance one: plan for warming, or shed load
  while the hit ratio recovers.
```

| Metric | Value | Note |
|---|---|---|
| Reads | 50,000/s | unchanged |
| DB load | 2,500/s | **95% removed** |
| Footprint | ~2 GB | 1M items |
| Cold start | 20× DB load | capacity event |

> **A cold cache is a thundering herd waiting to happen**  
> The moment the cache is empty — a restart, a flush, a failover — every read becomes a miss and the database receives the full unfiltered load, which is by definition far more than it was provisioned for. Warming the cache before taking traffic, or shedding load until the hit ratio recovers, is the difference between a slow recovery and a second outage on top of the first.

**Technologies**

| Choice | Best for | Trade |
|---|---|---|
| Redis | Rich types, persistence, clustering | More to operate than a plain cache |
| Memcached | Pure key-value at very high throughput | No persistence or data structures |
| In-process cache | Sub-microsecond hits for tiny hot sets | Per-instance; no shared invalidation |
| Two-tier (local + shared) | Cutting network round trips on the hottest keys | Two invalidation paths to keep honest |
| CDN or edge cache | Publicly cacheable responses | Only for content that is not user-specific |

The two-tier arrangement is worth understanding because it is where most of the subtle bugs live: a small in-process cache in front of a shared one removes the network hop for the hottest keys, but an invalidation now has to reach every instance's local copy rather than one shared entry, and a missed one is invisible until someone notices two servers disagreeing.

**Trade-offs**

**Cache aside against its alternatives**

| Approach | Who fills the cache | Consistency | Best for |
|---|---|---|---|
| Cache aside | The application, on a miss | Bounded by TTL and invalidation | Read-heavy, tolerant of brief staleness |
| Read through | The cache library or store | Same, but centralised | Uniform access across many services |
| Write through | The write path, synchronously | Cache always matches the last write | Read-after-write correctness |
| Write behind | The write path, asynchronously | Risk of loss before flush | Write-heavy with tolerable loss |
| No cache | — | Always correct | Low read volume, or strict freshness |

The reason cache aside is the default is that it fails soft: the application already knows how to read from the database, so a cache problem costs latency rather than correctness. Every other variant moves some responsibility into the cache layer and takes that property away with it.

> **Ask before choosing it**  
> How stale can this value be before someone is harmed? What is the read-to-write ratio? What happens the first minute after the cache is empty? If the answer to the first question is that it cannot be stale at all, the cache belongs somewhere else in the design, or the pattern is the wrong one.

**How it fails**

**How cache aside goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Database collapses on restart | Cold cache sends full read load through | Warm before taking traffic; shed load while the ratio recovers |
| Every reader hits the database at once | A hot key expired simultaneously for all callers | Single flight on the miss path; jittered TTLs |
| Stale data that never corrects | Invalidation lost between the write and the delete | TTL as a backstop; delete after the commit |
| Cache and database disagree permanently | Writers update the cache instead of deleting | Delete on write; refill from the authority |
| An old value overwrites a newer one | A slow miss fill lands after a concurrent write | Version the value or guard the fill |
| Cache outage becomes an outage | Cache errors treated as request failures | Treat any cache error as a miss |
| One caller sees another's data | Key omits a parameter that changes the value | Key on every input that affects the result |

> **The slow-fill race is the one that survives review**  
> A reader misses, queries the database, and is then delayed. Meanwhile a writer updates the row and deletes the cache entry. The delayed reader now writes its stale value into the empty cache, and it will sit there until the TTL expires — with nothing in the logs to suggest anything went wrong. Guarding the fill with a version, or filling only if the key is still absent, is what closes it.

**Where it is used**

- **Product catalogues and profile services**, where a small set of records is read constantly and changes rarely.
- **Session and entitlement lookups**, which are on every request and would otherwise dominate database load.
- **Rendered fragments** — a serialised response cached whole, so a hit skips assembly as well as the query.
- **Rate-limit counters and feature flags**, where the authority is consulted rarely and the cached copy carries the traffic.
- **Any read path where the database is the constraint** and the workload is skewed toward a hot subset.

**In the interview**

What the interviewer is listening for is whether you treat the cache as an optimisation or as part of the correctness argument.

- **State the staleness budget out loud.** Say how old a value may be before it causes harm, and make the TTL follow from that rather than picking a round number.
- **Say delete, not update, and give the reason** — concurrent writers can otherwise leave the cache permanently disagreeing with the database.
- **Volunteer the cold-start problem.** Describing what happens when the cache is empty shows you have thought past the happy path, and it is the failure that actually takes systems down.
- **Name the stampede and its fix** before being asked; single flight on the miss path is the expected answer.
- **Make the fallback explicit.** Cache errors are misses — that one sentence is what separates an accelerator from a hard dependency.

**Practice drill**

A catalogue service takes 40,000 reads per second at a 95% hit ratio. Work out the database load in steady state, then the load in the first second after the cache is flushed, and decide what the service should do during that window. Then extend it: the cache holds 1 million items at 2 KB each — size the memory, and say what you would change if the hot set were 50 million items instead.

**Go deeper**

Cache aside is the simplest caching arrangement and the one whose failure modes are most often discovered in production rather than in review.

**The application owns the miss.** That single design choice is what makes the pattern fail soft — the read path already contains a working database query, so a cache that is slow, empty or entirely absent degrades latency and nothing else. Every richer variant moves the fill into the cache layer and gives up that property, which is why cache aside remains the sensible default even where a read-through library is available.

**Deleting rather than updating is a correctness argument, not a style preference.** Two writers who both update the cache can land in the opposite order to the one in which they reached the database, leaving a cached value that no longer corresponds to any committed state and that nothing will correct until it expires. A delete cannot invert order: whoever refills reads whatever the authority actually holds at that moment.

**The TTL exists because invalidation is unreliable.** Every delete is a message that can be lost — the process can crash between committing the write and issuing the delete, the cache can be briefly unreachable, a deployment can drop the in-flight call. Without an expiry, a single lost invalidation leaves a wrong value in place indefinitely. With one, the damage is bounded by a number you chose deliberately.

**Skew is what makes caching pay, and also what makes it dangerous.** The same concentration of traffic on a few keys that produces a high hit ratio also means those few keys, when they expire together, send a synchronised burst at the database. Jittering expiry times spreads the burst, and collapsing concurrent misses for the same key into a single fill removes it — which is the single-flight pattern, and it belongs on any key hot enough to be worth caching.

**A cold cache is a capacity event.** Steady-state sizing assumes the hit ratio holds, so the database is provisioned for the misses only. The moment the cache is empty the database receives the entire read volume, frequently an order of magnitude more than it can serve, and the failure arrives exactly when the system is already recovering from whatever emptied the cache. Treating cache loss as a load-shedding scenario rather than a latency regression is what keeps that from compounding.

**The subtle race is the slow fill.** A reader misses, goes to the database, and is delayed; a writer commits and invalidates in the meantime; the delayed reader then writes its now-stale value into the empty slot. Nothing errors, nothing is logged, and the wrong value persists for a full TTL. Versioning the cached value, or filling only when the key is still absent, closes it — and knowing that this race exists is a reliable marker of someone who has operated a cache rather than only drawn one.

**Related patterns:** Cache Invalidation · Single Flight · Stale While Revalidate · Read Through Cache

---

### Read Through Cache

*The cache itself fetches from the source on a miss, so every caller sees one lookup and the fill logic lives in one place.*

> **When you hear…** Many services read the same data · fill logic keeps being duplicated · you want one place to fix a caching bug

**Flow:** `Client` → `Cache layer` → `Loader on miss` → `Source of truth` → `Cached result`

**The problem**

Six services all cache the same customer record. Each implements its own look-aside logic, and they have drifted: two use a five-minute lifetime and one uses an hour, one forgot to handle a cache error as a miss, and only one guards against concurrent fills. A bug in the caching logic has to be found and fixed six times.

The duplication is not merely tedious. Because each copy is slightly different, the same record can be stale for different lengths of time depending on which service is asked, and nobody can state a single staleness guarantee for the system.

> **Move the fill behind the cache interface**  
> If the cache knows how to load what it does not have, callers stop writing fill logic altogether — they simply ask for a key. The lifetime, the concurrency guard, the error handling and the metrics then exist once, so improving them improves every caller at the same time.

**Mental model**

The caller sees a store that always has the answer. Behind that interface the cache checks itself, and on a miss calls a loader function that knows how to produce the value, stores it, and returns it.

1. **Ask** — The caller requests a key. There is no miss branch in caller code.
2. **Resolve** — On a miss the cache invokes the registered loader for that key.
3. **Store** — The loaded value is written with the configured lifetime.
4. **Serve** — The value is returned, identically to a hit from the caller's point of view.
5. **Collapse** — Concurrent requests for the same missing key wait on one load rather than each issuing their own.

> **The uniform interface hides where the time went**  
> Because a hit and a miss look the same to the caller, a request that quietly took a database query and a network round trip is indistinguishable from one served from memory. Without per-key hit ratio and load-duration metrics exposed by the cache layer, a collapsing hit ratio shows up only as unexplained latency somewhere upstream.

**How it works**

**The loader contract**

```text
CALLER
  value = cache.get(key)          <- that is the whole API

CACHE LAYER
  get(key):
    v = store.lookup(key)
    if v present: return v
    return singleFlight(key, () => {
        v = loader(key)           <- registered per cache
        store.put(key, v, ttl)
        return v
    })

WHAT THE LAYER OWNS
  the lifetime policy
  collapsing concurrent misses for the same key
  error handling: a loader failure must not be cached
    as a value, or one bad moment is served for a
    full TTL
  metrics: hits, misses, load time, load failures

WHAT THE CALLER OWNS
  nothing but the key

NEGATIVE RESULTS NEED A DECISION
  loader returns nothing for a key that does not exist
  -> caching that absence stops repeated lookups for
     keys that will never resolve
  -> not caching it means a missing key is an
     uncached path an attacker can hammer
  -> cache the absence, with a shorter lifetime than
     a real value
```

1. **Register one loader per cache** — The loader is the only place that knows how to produce a value, which is what removes the duplication.
2. **Collapse concurrent misses** — Single flight belongs in the layer; putting it in callers reintroduces the duplication you removed.
3. **Never cache a loader failure as a value** — An error stored under the key serves one bad moment for a full lifetime.
4. **Cache negative results deliberately** — With a shorter lifetime than a real value, so a missing key is not an uncached path.
5. **Expose hit ratio and load duration** — The uniform interface hides the cost of a miss; metrics are how it becomes visible again.
6. **Keep the loader free of side effects** — It may be invoked concurrently, retried, or abandoned when a caller times out.

**Where the wins show up**

```text
SIX SERVICES, ONE RECORD TYPE

BEFORE (cache aside in each)
  6 implementations of the miss path
  3 different lifetimes in production
  1 of 6 guards against concurrent fills
  staleness guarantee: unstateable

AFTER (read through)
  1 loader, 1 lifetime, 1 concurrency guard
  staleness guarantee: one number, written down
  a caching fix ships once and applies everywhere

THE COST
  the fill now runs inside the cache layer, so a slow
  source makes cache.get() slow — and callers no
  longer have a visible miss branch in which to
  reason about that
  -> the layer needs its own timeout, and a decision
     about what to do when the loader exceeds it:
     fail, or serve the expired value it still holds
```

| Metric | Value | Note |
|---|---|---|
| Implementations | 6 → 1 | one loader |
| Lifetimes | 3 → 1 | **stateable guarantee** |
| Concurrency guards | 1 of 6 → all | in the layer |
| New risk | hidden miss cost | needs metrics |

> **A loader with no timeout makes every caller wait on the slowest source**  
> Because the miss is invisible, a source that has become slow does not present as a cache problem — it presents as every caller of that cache becoming slow at once, with no obvious cause. The layer needs its own timeout on the loader and an explicit policy for exceeding it, or the thing that was meant to protect the source becomes the mechanism that propagates its latency everywhere.

**Technologies**

| Choice | Best for | Trade |
|---|---|---|
| Caffeine or Guava loading caches | In-process read through with single flight built in | Per-instance; no shared invalidation |
| Redis with a cache library | Shared read through across services | The loader still runs in application processes |
| DataLoader-style batching | Collapsing many keys into one source query | Request-scoped; not a long-lived cache |
| Platform caching layers | Read through managed by the store itself | Less control over the fill and its timeout |
| Hand-rolled wrapper | Exactly the semantics you want | You own single flight, metrics and negative caching |

Batching deserves a mention alongside read through because they compose well: collapsing concurrent misses removes duplicate loads for the same key, while batching collapses loads for different keys into a single source query — together they turn a fan-out of individual lookups into one round trip.

**Trade-offs**

**Read through against cache aside**

| Aspect | Read through | Cache aside |
|---|---|---|
| Fill logic | Once, in the layer | In every caller |
| Staleness guarantee | One policy, stateable | Whatever each caller chose |
| Caller complexity | A single get | Explicit miss branch |
| Miss cost visibility | Hidden without metrics | Visible in caller code |
| Failing soft | Depends on the layer's policy | Natural — the caller owns the fallback |
| Flexibility per caller | Low by design | High |

The trade is uniformity against visibility. Read through is the right answer when many callers share the same data and the duplication is causing real inconsistency; cache aside stays better when one service owns the data and wants its miss path in plain sight.

> **Ask before choosing it**  
> How many callers actually share this data? If the answer is one, the indirection buys nothing. If it is several and they have already drifted apart on lifetimes, that drift is the argument for the pattern — and the first thing to fix is agreeing what the single staleness guarantee should be.

**How it fails**

**How read through goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Every caller slows at once | Loader has no timeout and the source degraded | Timeout in the layer plus an explicit exceeded-policy |
| An error is served for a full lifetime | Loader failure cached as if it were a value | Never store a failure; retry on the next request |
| Missing keys hammer the source | Negative results not cached | Cache absence with a shorter lifetime |
| Latency with no visible cause | Hit ratio collapsed but is not measured | Expose hit ratio and load duration from the layer |
| Duplicate loads for one key | Single flight omitted from the layer | Collapse concurrent misses inside the cache |
| Stale data across services | Invalidation reaches the shared cache but not local tiers | One invalidation path, or no local tier |
| Loader side effects repeated | Loader invoked concurrently or retried | Keep the loader pure and idempotent |

> **Caching a failure is worse than not caching at all**  
> If the loader throws and the layer stores that outcome under the key, a transient source blip is converted into a guaranteed period of failure for every caller of that key — and the source recovering does not end it. The rule is absolute: failures are never values. Retry on the next request, and if repeated failures need to be damped, do it with a breaker rather than by storing the error.

**Where it is used**

- **Shared reference data** — customers, products, entitlements — read by many services that would otherwise each cache it differently.
- **In-process loading caches** in application frameworks, where single flight and expiry come built in.
- **Graph and API layers** that resolve the same entity many times within one request, usually combined with batching.
- **Configuration and feature-flag clients**, where the library hides the fetch and every consumer just reads a value.
- **Platform-managed caches** that sit in front of a store and load on miss without the application participating.

**In the interview**

The signal here is whether you can say what the layer must own and why putting those responsibilities in callers is what caused the problem.

- **Lead with the duplication, not the performance.** Read through is chosen to make one staleness guarantee stateable, not to make hits faster — hits are equally fast either way.
- **Name what moves into the layer**: lifetime, single flight, error policy, negative caching, metrics. Listing those is the answer.
- **Volunteer that failures must never be cached as values**, which is the mistake that turns a blip into an outage.
- **Raise the hidden-miss problem yourself.** A uniform interface conceals where the time went, so the layer has to publish hit ratio and load duration.
- **Say when you would not use it** — a single owning service is better served by cache aside, where the miss path stays visible.

**Practice drill**

Six services each cache the same customer record with their own look-aside code, using lifetimes of five minutes, five minutes, one hour, one hour, ten minutes and none at all. Write down the staleness guarantee the system can currently offer. Then design the read-through replacement: name every responsibility that moves into the layer, decide the negative-caching lifetime, and say what the layer should do when the loader exceeds its timeout.

**Go deeper**

Read through moves the fill behind the cache interface so that the policy governing it exists once rather than once per caller.

**The problem it solves is inconsistency, not latency.** A hit is exactly as fast under cache aside; nothing about the read path improves. What improves is that the lifetime, the concurrency guard, the error handling and the negative-caching decision stop being reimplemented — and therefore stop diverging. A system with six look-aside implementations cannot state how stale its data may be; a system with one loader can, and that number can then be defended or changed deliberately.

**The layer inherits real responsibilities.** Collapsing concurrent misses belongs there, because pushing it back to callers reintroduces exactly the duplication the pattern removed. So does deciding what a missing key means: caching absence stops repeated lookups for keys that will never resolve, and leaving it uncached creates a path that bypasses the cache entirely, which is both a performance hole and something an adversary can exploit.

**Failures must never be stored as values.** This is the sharpest rule in the pattern. Caching a loader error converts a transient source problem into a guaranteed window of failure for every caller of that key, and the source recovering does not shorten it. Where repeated failures genuinely need damping, a circuit breaker in front of the loader is the right instrument, because it is time-bounded and self-healing in a way a cached error is not.

**Hiding the miss hides the cost.** Under cache aside the miss branch is visible in caller code, so a developer reasoning about latency sees the database query. Read through removes that branch, and a source that has become slow then presents as every caller of the cache slowing simultaneously with no local explanation. The layer must therefore publish hit ratio, load duration and load failures, and it must impose its own timeout on the loader — otherwise the abstraction that was meant to protect the source becomes the channel that spreads its latency.

**It composes with batching more usefully than it composes with anything else.** Single flight collapses concurrent misses for the same key; batching collapses misses for different keys into one source query. Together they turn a fan-out of individual lookups — the characteristic shape of a graph or aggregation layer — into a single round trip, which is a far larger win than caching alone provides.

**Choose it for shared data and against it for owned data.** Where one service owns a dataset and reads it, the indirection buys nothing and costs visibility; cache aside keeps the fallback in plain sight and fails soft by construction. Where several services read the same records and have already drifted on lifetimes, that drift is the evidence for the pattern, and agreeing the single staleness number is the first and most valuable part of adopting it.

**Related patterns:** Cache Aside · Single Flight · Write Through Cache · Cache Invalidation

---

### Write Through Cache

*Every write goes to the cache and the store together, so a value is never in the cache unless it is already durable.*

> **When you hear…** Read-after-write must be correct · the writer immediately reads back · stale reads after an update are unacceptable

**Flow:** `Client write` → `Cache update` → `Store write` → `Commit` → `Subsequent read hits`

**The problem**

A user edits their profile and is redirected to the profile page. Under cache aside the write invalidated the cached entry, and the redirect arrives fast enough that the refill races the commit — so the page sometimes shows the value the user just replaced, and they press save again.

The pattern is not rare. Any flow that writes and then immediately reads the same record — edit and redirect, submit and confirm, update and re-render — puts the read in exactly the window where invalidation has not settled.

> **Make the cache part of the write, not a consequence of it**  
> If the write path populates the cache before returning, the entry is correct the instant the write is acknowledged, and the read that follows is guaranteed to hit a current value. The cost is that every write now touches two systems and the write path owns both of them.

**Mental model**

The cache sits in front of the store on the write path as well as the read path. A write is not finished until both have it, so a reader can never observe a cache entry that is older than the last acknowledged write.

1. **Write to the store** — The authority commits first; nothing is acknowledged before it is durable.
2. **Write to the cache** — The same value, under the same key, as part of the same operation.
3. **Acknowledge** — Only once both have accepted it.
4. **Read** — Always a hit for a recently written key, and always the current value.
5. **Expire** — A lifetime still bounds keys nobody has written for a long time.

> **The order decides what a failure leaves behind**  
> Cache first then store means a crash in between advertises a value that was never committed — readers see data the database does not have. Store first then cache means a crash leaves the cache merely stale, which the lifetime will correct. Always commit the authority first and accept that the cache may lag; never the reverse.

**How it works**

**The write path and its failure points**

```text
WRITE
  db.update(row)                <- authority first, always
  cache.set(key, value, ttl)    <- then the copy
  return ack

WHY NOT THE OTHER ORDER
  cache.set then crash before db.update
  -> the cache advertises a value the store never
     accepted; readers see a write that did not happen
  -> no lifetime saves you from this: the value is
     wrong, not merely old
  db.update then crash before cache.set
  -> the cache holds the previous value, which the
     lifetime removes; readers see old data briefly
  one is a correctness failure, the other a freshness
  one — so the order is not a preference

WHAT IF THE CACHE WRITE FAILS
  the store already committed, so the write succeeded
  -> do NOT fail the request
  -> delete the key instead of leaving it stale, so
     the next reader refills from the authority
  -> if the delete also fails, the lifetime is what
     bounds the damage

READ PATH IS UNCHANGED
  still look aside: hit, or fall through and fill
  write through governs writes; it does not remove
  the need to handle a miss
```

1. **Commit the store before the cache** — The reverse order can advertise a value that was never durable, which no lifetime repairs.
2. **Never fail a request because the cache write failed** — The authority already committed; the write did succeed.
3. **Delete rather than leave stale on a cache error** — A deletion is self-correcting; a stale entry is not.
4. **Keep a lifetime anyway** — Write through only freshens keys that are written; everything else still ages.
5. **Write the shape you serve** — Storing the rendered value avoids re-assembly on read, which is most of the benefit.
6. **Accept the latency cost consciously** — Every write now waits on two systems, so the write path gets slower to make the read path correct.

**What it costs and what it buys**

```text
WORKLOAD        10,000 writes/s, 100,000 reads/s

WRITE LATENCY
  store only            8 ms
  store + cache         8 ms + 1 ms = 9 ms
  -> ~12% slower writes

READ CORRECTNESS
  cache aside    a read in the invalidation window can
                 serve the pre-write value
  write through  a read after the ack cannot: the key
                 was populated before the ack returned

WHERE IT STOPS PAYING
  write-heavy keys that are rarely read
  -> every write pays the cache cost and no read
     collects the benefit
  -> at a 1:1 read/write ratio you are doing twice the
     work for very little; the pattern assumes reads
     dominate, exactly like cache aside does

COMBINED WITH A COLD CACHE
  write through populates only what is written
  -> a restart still leaves reads missing
  -> it is not a substitute for warming
```

| Metric | Value | Note |
|---|---|---|
| Write cost | +1 ms | ~12% slower |
| Read-after-write | correct | **the point** |
| Break-even | reads ≫ writes | same as cache aside |
| Cold cache | unchanged | still needs warming |

> **Write through does not make the cache authoritative**  
> It guarantees that a written key is current, not that the cache holds everything or that it can be read instead of the store. Keys nobody has written recently still expire, a flush still empties it, and the read path still needs its miss branch. Treating the cache as the source of truth because writes go through it is how data gets lost when it is flushed.

**Technologies**

| Choice | Best for | Trade |
|---|---|---|
| Application-level dual write | Full control of ordering and failure handling | You own the correctness argument |
| Redis with write-through wrapper | Shared cache across services | The wrapper must get the ordering right |
| Caching layers in ORMs | Entity caches that follow the unit of work | Hidden behaviour; ordering may not be what you assume |
| Storage engines with integrated cache | The store manages both tiers | Less control; coupling to one product |
| Write behind | Absorbing write bursts | Acknowledges before durability — different guarantee |

The ORM entity cache is worth singling out: it often behaves as write through within a transaction and as something looser across processes, so a guarantee that appears to hold in a single-instance test quietly does not hold in production behind several application servers.

**Trade-offs**

**Write through against the alternatives**

| Approach | Write cost | Read-after-write | Risk on crash |
|---|---|---|---|
| Cache aside | One store write | Racy in the invalidation window | Stale entry, bounded by lifetime |
| Write through | Store plus cache | Correct | Stale entry if the cache write fails |
| Write behind | Cache only, flushed later | Correct from cache | Unflushed writes can be lost |
| No cache | One store write | Correct | None |

The decision is narrower than it looks: write through buys exactly one property — a read immediately after a write sees the write — and charges for it on every write. If the flows in question do not read back straight away, cache aside gives the same read performance for less.

> **Ask before choosing it**  
> Which flows actually read a record immediately after writing it? If that list is short, consider making only those paths write through rather than all of them. And ask what should happen when the cache write fails after a successful commit — the answer must be to proceed, or you have made a cache outage into a write outage.

**How it fails**

**How write through goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Readers see a write that never committed | Cache written before the store | Commit the authority first, always |
| Writes fail when the cache is down | Cache error treated as a write failure | Acknowledge the write; delete the key instead |
| Stale entry after a partial write | Store committed, cache write failed silently | Delete on cache error; keep a lifetime as backstop |
| Write latency doubles under cache pressure | Slow cache on the synchronous write path | Timeout the cache write and fall back to a delete |
| Cold reads still miss after a restart | Only written keys are populated | Warm separately; write through is not warming |
| Two writers leave disagreeing values | Concurrent writes racing in the cache | Version the value, or delete rather than set |
| Data lost on flush | Cache treated as authoritative because writes go through it | The store remains the source of truth |

> **A cache failure must never fail a committed write**  
> Once the store has accepted the write, the operation has succeeded from the user's point of view. Returning an error because the second, optional system did not acknowledge is both wrong and dangerous — the client will retry a write that already happened. Log it, delete the key, and return success.

**Where it is used**

- **Profile and settings editors**, where the user is redirected straight back to the record they changed.
- **Shopping baskets and checkout state**, read on the very next request after every mutation.
- **Entity caches in ORMs**, which apply write-through semantics within a unit of work.
- **Session stores**, where a write is followed immediately by reads on subsequent requests.
- **Configuration services** whose consumers poll the value straight after it is published.

**In the interview**

What distinguishes a good answer is the ordering argument and the failure policy, not the definition.

- **State the order and justify it** — store first, cache second, because the reverse can advertise a value that was never committed and no lifetime repairs that.
- **Say that a cache failure must not fail the write.** It is the clearest signal that you have thought about which system is authoritative.
- **Name the single property it buys**: read-after-write correctness. Claiming broader benefits suggests you have confused it with caching in general.
- **Point out that it still needs a lifetime and still needs warming** — write through populates only what is written.
- **Offer the narrower alternative** of applying it to the few flows that read back immediately rather than to every write.

**Practice drill**

A profile service handles 10,000 writes and 100,000 reads per second, and users are redirected to their profile immediately after saving. Decide whether to apply write through to every write or only to the flows that read back, and defend the choice with the read-to-write ratio. Then specify exactly what the service does when the store commit succeeds and the cache write times out, and say what a client should see.

**Go deeper**

Write through populates the cache as part of the write, so a key that has just been written is guaranteed current when it is read.

**It buys one property and charges on every write.** The guarantee is read-after-write correctness: a reader arriving after the acknowledgement cannot see the pre-write value, because the entry was populated before the acknowledgement returned. Everything else about caching is unchanged — hit latency, footprint, the need for a miss branch. Teams that adopt it expecting general improvement are usually disappointed, because the read path performs identically to cache aside.

**The ordering is a correctness argument.** Writing the cache before the store means a failure between the two advertises a value the authority never accepted, and readers then observe a write that did not happen — a wrong value, not an old one, which no expiry corrects. Writing the store first means a failure leaves the cache merely behind, which the lifetime resolves. These outcomes are not comparable, so the order is fixed rather than chosen.

**A cache error must not fail the request.** After the store commits, the operation has succeeded; the cache is an accelerator that did not keep up. Returning an error invites the client to retry a write that already happened, which for anything non-idempotent is considerably worse than a stale cache entry. The correct handling is to delete the key so the next reader refills from the authority, and to treat a failed delete as bounded by the lifetime.

**It does not make the cache a store.** Write through creates a tempting illusion: because every write passes through the cache, the cache appears to hold everything. It does not — keys age out, flushes empty it, and a new instance starts blank. The store remains the source of truth, the read path still needs to handle a miss, and warming is still a separate concern. Systems have lost data by forgetting this after a flush.

**The economics are the same as cache aside, with an extra write.** Both assume reads dominate; write through simply adds a cache write to every mutation. As the read-to-write ratio approaches one, the added cost stops being repaid, and the pattern becomes work performed for a guarantee nobody is exercising. Knowing the ratio for the specific keys involved is what makes the choice defensible.

**Applying it selectively is usually the better answer.** The flows that genuinely read back immediately — edit and redirect, submit and confirm — are typically a small subset of writes. Making only those paths write through gives the correctness where it is observable and leaves the rest on the cheaper path, which is a more precise instrument than turning the behaviour on globally and paying for it everywhere.

**Related patterns:** Cache Aside · Write Behind Cache · Read Through Cache · Cache Invalidation

---

### Write Behind Cache

*Writes land in the cache and are flushed to the store asynchronously, trading durability for absorbing bursts the store cannot take.*

> **When you hear…** Write volume exceeds what the store can absorb · many writes touch the same key · some loss on crash is acceptable

**Flow:** `Client write` → `Cache accepts` → `Write buffer` → `Batched flush` → `Store`

**The problem**

A view counter is incremented two hundred thousand times a second. Each increment is a database write, the rows are highly contended, and the store spends its capacity on updates whose intermediate values nobody will ever read — only the running total matters.

Provisioning the store for that write rate is possible and absurd: the same row is being rewritten hundreds of times a second, so almost all of the work is immediately superseded.

> **Absorb in memory, settle in batches**  
> If writes accumulate in the cache and are flushed periodically, many updates to the same key collapse into one store write, and bursts are smoothed into a steady flush rate. The price is explicit and unavoidable: anything not yet flushed is lost if the cache dies, so the pattern is only available where that loss is acceptable.

**Mental model**

The cache becomes the write target and the store becomes a destination it drains into. The client is acknowledged as soon as the cache has the value, and a background process moves batches to the store.

1. **Accept** — The write goes to the cache and is acknowledged immediately.
2. **Coalesce** — Repeated writes to the same key overwrite each other in the buffer rather than queueing.
3. **Batch** — A flush gathers many keys into one store operation.
4. **Flush** — On an interval, a buffer size, or both — whichever comes first.
5. **Accept loss** — Whatever is unflushed when the process dies is gone, by design.

> **The acknowledgement is a lie about durability**  
> The client is told the write succeeded when only volatile memory holds it. That is the whole trade, and it must be a deliberate product decision rather than an implementation detail — the difference between losing a few seconds of view counts and losing a few seconds of payments is the difference between a sensible optimisation and an incident.

**How it works**

**Buffer, coalesce, flush**

```text
WRITE
  buffer[key] = value        <- overwrites any pending value
  ack immediately

FLUSH  (timer or size)
  batch = take(buffer)
  store.writeBatch(batch)
  on failure: put the batch back and retry with backoff

COALESCING IS WHERE THE WIN IS
  200,000 increments/s across 1,000 hot keys
  flush every second
  -> 1,000 store writes/s, not 200,000
  -> a 200x reduction, achieved because intermediate
     values were never worth persisting

WHAT THE BUFFER MUST BOUND
  memory: an unbounded buffer turns a store outage
    into an out-of-memory crash, which loses
    everything in it
  age: a key written once and never again must still
    reach the store, so flush on time as well as size

ORDERING WITHIN A KEY
  coalescing is safe for last-write-wins values
  it is NOT safe where every change matters:
  counters must accumulate deltas, not overwrite,
  or increments are silently dropped
```

1. **Bound the buffer in size and age** — Unbounded growth converts a store outage into a crash that loses the entire buffer.
2. **Coalesce only last-write-wins values** — Counters must accumulate deltas; overwriting silently discards increments.
3. **Flush on time as well as size** — A key written once would otherwise sit in the buffer indefinitely.
4. **Retry failed batches with backoff** — A transient store failure should delay the flush, not discard it.
5. **Flush on shutdown** — A graceful stop can drain the buffer; only a crash should lose data.
6. **Make the loss window explicit** — State the flush interval as the amount of data the design is willing to lose.

**Sizing the loss window**

```text
FLUSH INTERVAL = MAXIMUM DATA LOSS

1 second    lose up to 1 s of writes
            200,000 buffered at peak
            ~1,000 store writes/s
10 seconds  lose up to 10 s
            better coalescing, 10x the exposure
on size only
            loss depends on traffic, which means the
            exposure is unbounded during a lull —
            avoid

MEMORY
  1,000 hot keys x small values   trivial
  10,000,000 distinct keys        no longer a buffer,
                                  it is a database
  -> the pattern assumes writes concentrate on a
     bounded working set

WHAT HAPPENS WHEN THE STORE IS DOWN
  buffer grows
  -> at the memory bound, either shed writes or block
  -> blocking converts an asynchronous design back
     into a synchronous one under stress, which is
     usually the right choice: it is better to slow
     down than to accept writes you cannot keep
```

| Metric | Value | Note |
|---|---|---|
| Store writes | 200k → 1k/s | **200× coalesced** |
| Loss window | = flush interval | state it explicitly |
| Buffer | bounded | or OOM loses all |
| Working set | must be bounded | not a database |

> **A store outage is where this pattern decides its own character**  
> With the store unavailable the buffer grows. Letting it grow without limit means the eventual crash loses everything it holds. Shedding writes silently means the client was told they succeeded. Blocking new writes reintroduces the synchronous behaviour the pattern removed — and is usually correct, because slowing down is preferable to acknowledging writes that will never land.

**Technologies**

| Choice | Best for | Trade |
|---|---|---|
| Redis with a flush worker | Counters and aggregates on a bounded key set | You own the buffer bounds and retry policy |
| Application write buffer | Simple in-process coalescing | Lost on every restart, not just crashes |
| Stream plus consumer | Durable buffering with replay | No longer loses data, but no longer as cheap |
| Storage engines with write buffers | LSM stores that batch internally | Durability handled by the engine's own log |
| Write through | Where loss is unacceptable | Every write pays store latency |

The stream variant is the honest upgrade path: putting writes on a durable log instead of a memory buffer keeps the coalescing and the batching while removing the loss, at the cost of running the log. When someone objects to the loss window, that is usually the design being asked for.

**Trade-offs**

**Write behind against the alternatives**

| Approach | Ack latency | Store load | Loss on crash |
|---|---|---|---|
| Write through | Store latency | One write per update | None |
| Write behind | Cache latency | Coalesced and batched | Up to one flush interval |
| Durable queue | Queue append latency | Coalesced by the consumer | None |
| Direct writes | Store latency | One write per update | None |

The row that matters is the last column. Write behind is the only option that trades durability, and it is chosen precisely when the data is cheap enough that a few seconds of it is worth a large reduction in store load. Where that is not true, the durable queue gives most of the same shape without the trade.

> **Ask before choosing it**  
> What exactly is lost if the process dies right now, and who notices? If the answer involves money, entitlements or anything a user would dispute, the pattern is wrong and a durable buffer is the design being described. If it is view counts or last-seen timestamps, it is an easy win.

**How it fails**

**How write behind goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Data lost on restart | Buffer not drained on shutdown | Flush on graceful stop; only crashes should lose |
| Out of memory during a store outage | Unbounded buffer growth | Bound the buffer; block or shed at the limit |
| Increments silently dropped | Coalescing by overwrite on a counter | Accumulate deltas rather than overwriting |
| A key never reaches the store | Flushing on size only | Flush on age as well |
| Duplicate application after a retry | Batch retried without idempotence | Idempotent writes or a per-batch key |
| Reads disagree with the store | Readers querying the store, not the cache | Read from the cache, which holds the newest value |
| Unbounded exposure during quiet periods | Loss window depends on traffic | Time-based flush makes the window a constant |

> **Reads must come from the cache, not the store**  
> While writes are buffered the cache holds newer data than the store does. A reader that queries the store directly sees values that are behind by up to a flush interval — and because the write was acknowledged, the user believes it happened. Any read path in a write-behind design has to go through the same cache, which is a coupling worth stating explicitly.

**Where it is used**

- **View, like and impression counters**, where the running total matters and individual increments do not.
- **Last-seen and presence timestamps**, overwritten constantly and read rarely.
- **Metrics and telemetry aggregation**, which batches by nature and tolerates a small loss window.
- **LSM-tree storage engines**, which buffer writes in memory and flush in batches — the same idea inside a database.
- **Rate-limit counters**, where approximate accounting is acceptable and the write volume is enormous.

**In the interview**

This pattern is a durability trade, so the answer that lands is the one that names the trade before naming the benefit.

- **Say what is lost and how much** — up to one flush interval — and make it a product decision rather than a technical detail.
- **Lead with coalescing, not batching.** The large win comes from many updates to one key collapsing into a single write; batching is secondary.
- **Describe the store-outage behaviour.** Bounding the buffer and blocking at the limit is the mature answer; unbounded growth is the common wrong one.
- **Note that counters cannot be coalesced by overwriting** — deltas must accumulate, or increments vanish.
- **Offer the durable-log upgrade** when the interviewer pushes on loss; it keeps the shape and removes the trade.

**Practice drill**

A counter service takes 200,000 increments per second spread across 1,000 hot keys, flushing every second. Work out the store write rate and the coalescing factor, then state precisely what is lost if the process is killed at an arbitrary moment. Now the store goes down for ten minutes: decide what the service does as the buffer fills, and justify choosing between blocking writers and shedding writes.

**Go deeper**

Write behind acknowledges a write once the cache holds it and moves data to the store asynchronously, trading durability for throughput.

**Coalescing is the real mechanism.** Batching many keys into one store operation helps, but the large reduction comes from repeated writes to the same key overwriting each other in the buffer so that only the final value is persisted. Where traffic concentrates on a small hot set — counters, timestamps, presence — that collapses two orders of magnitude of writes into one, because the intermediate values were never worth storing.

**The acknowledgement is not a durability claim, and that must be deliberate.** The client is told the write succeeded while only volatile memory holds it. For view counts this is obviously fine; for anything a user could dispute it is obviously not. Because the code looks identical either way, the decision has to be made explicitly at design time rather than discovered when a process dies.

**Coalescing is unsafe for anything that is not last-write-wins.** Overwriting a pending value is correct for a timestamp or a status, and silently destructive for a counter: two increments buffered as overwrites become one. Counters must accumulate deltas in the buffer and apply them additively at the store, which is a different operation and an easy one to get wrong when the pattern is introduced by analogy.

**The buffer needs two bounds, not one.** A size bound prevents a store outage from growing the buffer until the process dies and loses everything in it. An age bound ensures a key written once and never again still reaches the store, rather than sitting in memory until something evicts it. Flushing on size alone also makes the loss window depend on traffic, which means exposure grows during quiet periods — the opposite of what anyone intends.

**The store-outage policy defines the design's character.** When the destination is unavailable the buffer fills, and there are only three options: grow without limit and eventually lose everything, drop writes that were already acknowledged, or stop accepting new writes. The third reintroduces synchronous behaviour under stress and is usually right, because degrading to slow is better than degrading to dishonest. Choosing it consciously is what separates a considered implementation from one that has simply never been tested against a failed store.

**Reads have to follow the writes.** While data is buffered the cache is ahead of the store, so any reader consulting the store directly sees values older than the acknowledged write. That makes the cache a required participant in the read path too, which is a coupling the pattern quietly introduces and which becomes visible only when a reporting query or a second service reads the store and disagrees with the application.

**Related patterns:** Write Through Cache · Cache Aside · Work Queue · Batch Processing

---

### Cache Invalidation

*Deciding when a cached copy stops being usable — by expiry, by explicit removal, or by versioning the key so the old copy is never asked for.*

> **When you hear…** Data changed but users still see the old value · staleness has to be bounded · one edit must appear everywhere at once

**Flow:** `Source changes` → `Invalidation signal` → `Entry removed` → `Next read misses` → `Fresh fill`

**The problem**

A price is corrected in the database and the old price is still being served twenty minutes later from three different caches, a CDN, and the browsers of everyone who loaded the page before the change. The correction happened in one place; the stale copies exist in many.

The difficulty is not removing one entry. It is knowing every place a copy came to rest, and reaching all of them — some of which, like a browser cache, cannot be reached at all.

> **You cannot delete what you cannot reach, so bound it instead**  
> Invalidation is a best-effort message to every holder of a copy, and messages get lost. The reliable instrument is not the message but the lifetime: an expiry bounds how wrong a missed invalidation can leave you, without needing to reach anything. The strongest technique removes the problem entirely by never reusing a key whose content has changed.

**Mental model**

There are three ways to stop serving a stale value, and they differ in what they require of you: waiting for it to expire, telling everyone to drop it, or changing the name so nobody asks for the old one.

1. **Expire** — A lifetime on the entry. Requires nothing of the writer; bounds staleness but never eliminates it.
2. **Evict** — An explicit delete on change. Immediate where it lands, but it is a message that can be lost.
3. **Version the key** — Change the key when the content changes. Nothing to invalidate, because the old key is simply never requested again.
4. **Tag and purge** — Group entries so one signal removes a set — the answer when one change affects many keys.
5. **Accept and bound** — Where a holder cannot be reached, the lifetime you gave it is the only control you have.

> **An invalidation you cannot verify is a hope, not a guarantee**  
> Browser caches, CDN edges you do not control, and in-process caches on instances that are mid-restart all hold copies you cannot reliably reach. Designs that assume a purge lands everywhere are correct almost all the time and wrong in ways that surface as one customer seeing something nobody else can reproduce.

**How it works**

**The three strategies, and when each applies**

```text
1  EXPIRY ONLY
   set(key, value, ttl)
   + nothing to do on write
   + no message to lose
   - every value can be stale for up to the ttl
   -> use when staleness is tolerable and writes are
      frequent enough that eviction would not help

2  EXPLICIT EVICTION
   on write: db.commit(); cache.delete(key)
   + fresh almost immediately
   - a lost delete leaves a stale entry
   - you must know every key the change affects
   -> always pair with a ttl as the backstop

3  KEY VERSIONING  (the strongest)
   key = name + ":" + version
   version changes when the content changes
   + nothing to invalidate: the old key is orphaned
   + safe across caches you cannot reach, including
     browsers
   - old entries linger until they expire, so you pay
     in memory rather than in staleness
   -> content-hashed asset names are this pattern

WHICH KEYS DOES ONE CHANGE AFFECT
   a product edit invalidates:
     product:123
     category-listing:electronics
     search-results containing it
     the rendered homepage fragment
   -> this fan-out is what makes eviction hard, and
      what tagging exists to solve
```

1. **Always set a lifetime, whatever else you do** — It is the only mechanism that needs no message to arrive, and therefore the only one that bounds an unreachable copy.
2. **Delete after the commit, never before** — Deleting first lets a concurrent reader refill the old value from a database that has not changed yet.
3. **Version the key where the consumer is unreachable** — Browsers and third-party caches cannot be purged; a new name makes that irrelevant.
4. **Tag entries when one change affects many** — Purging by tag is the difference between invalidating a product and enumerating every page that mentions it.
5. **Jitter expiry times** — Entries created together otherwise expire together and produce a synchronised refill.
6. **Never build correctness on prompt purging** — Propagation is slow, partial, and sometimes rate-limited.

**The delete-before-commit race**

```text
WRONG ORDER
  cache.delete(key)
  ... another reader misses here ...
  ... and refills from the OLD database value ...
  db.commit(newValue)
  -> the cache now holds the pre-write value with a
     full lifetime ahead of it, and nothing will
     correct it

RIGHT ORDER
  db.commit(newValue)
  cache.delete(key)
  -> any refill between the two reads the new value
  -> worst case the delete is lost and the ttl
     bounds it

THE RESIDUAL RACE
  even in the right order, a reader that missed
  BEFORE the commit can write its stale value AFTER
  the delete
  -> guard the fill: write only if still absent, or
     attach a version and refuse to overwrite a newer
     one
  -> this is the same slow-fill race cache aside has,
     and it is why versioning is the strongest option
```

| Metric | Value | Note |
|---|---|---|
| Expiry | no message | bounds everything |
| Eviction | fast, lossy | needs a TTL backstop |
| Versioning | **nothing to purge** | costs memory |
| Tagging | one signal, many keys | for fan-out |

> **Purging everything is not a fix, it is a second outage**  
> When a stale value is discovered the instinct is to flush the cache. That empties every entry at once, sends the full read load at a store provisioned for a small fraction of it, and takes the system down on top of the problem being investigated. Purge by key or by tag; treat a full flush as an emergency action with a known cost.

**Technologies**

| Mechanism | Reaches | Trade |
|---|---|---|
| TTL on the entry | Every holder, eventually | Staleness up to the lifetime |
| Explicit delete | Caches you can address | Lost messages leave stale copies |
| Surrogate keys or cache tags | Grouped entries in one call | Requires CDN or cache support |
| Content-hashed keys | Everything, including browsers | Old entries linger in memory |
| Pub/sub invalidation | Many local caches at once | At-most-once delivery; needs a TTL anyway |
| Full purge | Everything | Cold cache; treat as an incident action |

Pub/sub invalidation is how multi-tier caches keep local copies honest: the writer publishes a key change and every instance drops its local entry. It is genuinely useful and genuinely unreliable — a subscriber that was restarting missed the message — so it shortens the stale window rather than closing it, and the lifetime is still what bounds the worst case.

**Trade-offs**

**Choosing an invalidation strategy**

| Strategy | Freshness | Effort on write | Reaches unreachable caches |
|---|---|---|---|
| Long TTL only | Poor | None | Yes, eventually |
| Short TTL only | Good | None | Yes, eventually |
| Delete on write plus TTL | Very good | Must know affected keys | No, TTL covers them |
| Tag and purge | Very good for fan-out | Tagging at write time | Depends on the cache |
| Key versioning | Perfect | Version propagation | Yes, by construction |

The reason versioning wins where it is available is that it converts invalidation from a message-delivery problem into a naming problem, and naming is reliable. Where a version cannot be threaded to every reader, the practical answer is delete-plus-lifetime: fast where it lands, bounded where it does not.

> **Ask before choosing it**  
> How long may this value be wrong before someone is harmed — and separately, can every holder of a copy be reached? Those two answers pick the strategy between them. A value that must never be wrong and lives in a browser cache is telling you to version the key, not to purge harder.

**How it fails**

**How invalidation goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Stale value that never corrects | Delete lost and no lifetime set | Always set a TTL alongside eviction |
| Old value refilled with a full lifetime | Cache deleted before the commit | Commit first, then delete |
| One reader keeps seeing old data | An unreachable cache — browser or edge | Version the key rather than purging |
| Some pages update and others do not | Not every affected key was invalidated | Tag related entries and purge the tag |
| Store collapses after a fix | Full flush emptied the cache | Purge by key or tag; treat flush as an incident action |
| Synchronised load spikes | Entries created together expire together | Jitter the lifetimes |
| Stale value returns after being purged | Slow fill from a reader that missed before the write | Guard the fill by version or fill-if-absent |

> **The fan-out is where eviction quietly fails**  
> A single product edit can invalidate the product entry, the category listing, several search result pages and a rendered homepage fragment. Deleting the obvious key and missing the derived ones produces the worst symptom in caching: the detail page is correct while the listing that links to it is not, so the system appears to contradict itself. Tagging at write time is what makes the derived set removable in one operation.

**Where it is used**

- **Content-hashed asset filenames** in web builds — key versioning in its purest form, which is why those files can be cached for a year.
- **CDN surrogate keys**, tagging responses so one purge removes every page that embedded a changed record.
- **Pub/sub invalidation** across application instances holding local caches, shortening the window between a write and every node dropping its copy.
- **Short TTLs with stale-while-revalidate** on read-heavy API responses, where bounded staleness is cheaper than precise eviction.
- **ETag and conditional requests**, which let a holder revalidate rather than be told to discard.

**In the interview**

The expected move is to stop treating invalidation as a delete and start treating it as a reachability problem.

- **Name the three strategies** — expire, evict, version — and say which problem each solves rather than describing only deletion.
- **State the ordering rule and its reason**: commit first, then delete, because the reverse lets a reader refill the pre-write value with a fresh lifetime.
- **Raise the fan-out.** Listing the derived keys one edit affects shows you have invalidated something real rather than a single row.
- **Say that a TTL is always present**, because it is the only control over holders you cannot address.
- **Reach for versioning when asked about browsers or CDNs** — it is the answer that makes unreachable caches a non-issue.

**Practice drill**

A product edit changes a price. List every cached artefact that must stop serving the old value — application cache, rendered fragments, category listings, search results, CDN responses, and already-loaded browser copies. For each, say which strategy applies and why, then state the worst-case staleness the design guarantees and which holder sets that bound.

**Go deeper**

Invalidation is the problem of stopping every holder of a copy from serving it, and its difficulty comes entirely from the word every.

**There are three instruments and they solve different problems.** A lifetime bounds staleness without requiring anything of the writer or any message to arrive, which makes it the only mechanism that works against holders you cannot address. An explicit delete is fast where it lands but is a message, and messages are lost to crashes, restarts and brief network failures. Versioning the key sidesteps delivery altogether: if changed content gets a new name, the old entry is never requested again and needs no removal.

**Ordering is a correctness rule.** Deleting before the commit opens a window in which a concurrent reader misses, reads the unchanged database, and refills the cache with the pre-write value carrying a full lifetime — a stale entry created by the act of invalidating. Committing first means any refill in the gap reads the new value, and the worst case is a lost delete that the lifetime eventually resolves.

**The residual race survives correct ordering.** A reader that missed before the commit can still write its stale value after the delete, because its fill is in flight across the whole operation. Guarding the fill — writing only if the key is still absent, or attaching a version and refusing to overwrite something newer — is what closes it, and the fact that versioning removes this race by construction is a large part of why it is the strongest option.

**Fan-out is what makes eviction hard in practice.** One row changing can invalidate a detail entry, several listing pages, cached search results and a rendered fragment on an unrelated page. Removing the obvious key and missing the derived ones produces a system that visibly contradicts itself, which is worse for trust than uniform staleness. Tagging entries at write time so a single purge removes the whole derived set is the mechanism that scales here, and it has to be designed in rather than retrofitted.

**Unreachable holders set the real guarantee.** Once a response is in a browser cache or an edge you do not control, no purge reaches it, and the strongest statement the system can make about freshness is the lifetime it handed out. That is why long browser lifetimes are dangerous and long CDN lifetimes are not: one is revocable and the other is not. Separating the two — a short client lifetime with a long edge lifetime — gives both protection for the origin and the ability to correct a mistake.

**Flushing is an incident action with a known cost.** Discovering a stale value invites emptying the cache, which removes every entry at once and directs the full read load at a store sized for a small fraction of it. The cure then causes a larger outage than the disease, at a moment when attention is already elsewhere. Purging by key or tag should be the reflex, and a full flush should be treated as something with a capacity consequence that has been thought about beforehand.

**Related patterns:** Cache Aside · Stale While Revalidate · CDN · Single Flight

---

### Stale While Revalidate

*Serve the expired copy immediately and refresh it in the background, so nobody waits at expiry and the source sees one refill instead of a stampede.*

> **When you hear…** A latency spike every time the cache expires · bounded staleness is fine · the source cannot take a synchronised refill

**Flow:** `Request` → `Expired entry` → `Serve stale now` → `Background refresh` → `Entry replaced`

**The problem**

A cached API response has a one-minute lifetime. Fifty-nine seconds of every minute are served from memory in two milliseconds; at the sixtieth, the unlucky requests wait four hundred milliseconds for the source. The p99 is therefore dominated entirely by the moment of expiry.

Worse, they all wait together. Every caller that arrives while the entry is missing goes to the source, so expiry produces both a latency cliff and a synchronised burst on the thing the cache was protecting.

> **Expiry should start a refresh, not create a gap**  
> Nothing requires the cache to be empty while it is being refreshed. Serving the slightly old value immediately and fetching the new one behind the request removes the latency cliff for everyone, and lets a single refresh replace the entry that all those callers would otherwise have fetched individually.

**Mental model**

An entry has two ages rather than one: the point at which it stops being fresh, and the later point at which it stops being usable at all. Between them it is served immediately while a refresh runs.

1. **Fresh** — Inside the lifetime. Served directly, nothing else happens.
2. **Stale but usable** — Past the lifetime, inside the stale window. Served immediately, and a refresh is started.
3. **Refreshing** — One refresh runs; concurrent requests continue to be served the old value.
4. **Replaced** — The new value lands and subsequent requests are fresh again.
5. **Expired** — Past the stale window. Now a request must wait, exactly as before.

> **Staleness is now two numbers and people quote the wrong one**  
> The freshness lifetime is not the guarantee any more; the guarantee is lifetime plus stale window, because a value may legitimately be served for that long. Designs that advertise the first number and configure the second generously are promising freshness they do not deliver.

**How it works**

**The two windows**

```text
Cache-Control: max-age=60, stale-while-revalidate=600

age 0-60 s      fresh      serve, do nothing
age 60-660 s    stale      serve immediately AND refresh
age > 660 s     expired    must fetch; the caller waits

WHAT THE CALLER EXPERIENCES
  without swr    one request per minute waits 400 ms
                 and the rest wait behind it
  with swr       no request ever waits, until the
                 entry falls out of the stale window

WHAT THE SOURCE EXPERIENCES
  without swr    a burst of concurrent misses at each
                 expiry
  with swr       one refresh per interval per key,
                 provided refreshes are collapsed

THE REFRESH MUST BE SINGLE-FLIGHTED
  otherwise every request during the stale window
  starts its own refresh, and the stampede returns —
  now with the added insult of being invisible,
  because the callers were all served promptly
```

1. **Collapse concurrent refreshes** — Without single flight the stale window starts one refresh per request and the stampede returns unseen.
2. **Bound the stale window deliberately** — It is the real staleness guarantee, so choose it against what the data is for.
3. **Handle refresh failure by continuing to serve** — A failed refresh should extend the stale period, not evict a usable value.
4. **Do not apply it where staleness is unsafe** — Balances, entitlements and inventory at checkout must not be served from an expired copy.
5. **Refresh out of the request path** — The point is that the caller does not wait; a refresh awaited inline gives none of the benefit.
6. **Measure how often you serve stale** — A high ratio means the lifetime is too short or refreshes are failing.

**What the numbers do**

```text
1,000 req/s on one key, source takes 400 ms
max-age 60 s

WITHOUT SWR
  at each expiry ~400 concurrent requests miss
  -> 400 source calls in one burst, every 60 s
  -> those callers see 400 ms instead of 2 ms
  p99 is set entirely by the expiry moment

WITH SWR AND SINGLE FLIGHT
  at each expiry 1 refresh runs
  -> 1 source call every 60 s
  -> every caller still sees 2 ms
  p99 is the cache latency

SOURCE LOAD REDUCTION  400x at the expiry moment
LATENCY                cliff removed entirely

WHAT IT COSTS
  values up to 60 s old are served as normal, and up
  to 660 s old if refreshes keep failing
  -> the second number is the one to defend
```

| Metric | Value | Note |
|---|---|---|
| p99 | 400 ms → 2 ms | **cliff removed** |
| Source burst | 400 → 1 | single flight |
| Real staleness | max-age + window | quote this |
| On failure | keep serving | do not evict |

> **Without single flight this pattern hides a stampede rather than removing one**  
> During the stale window every arriving request is a candidate to start a refresh. If refreshes are not collapsed, a thousand requests per second start a thousand refreshes — the same burst as before, except that callers are served promptly from the stale copy, so nothing in the latency metrics reveals what is happening to the source.

**Technologies**

| Where it lives | Best for | Trade |
|---|---|---|
| HTTP Cache-Control directive | CDN and browser caching of public responses | Support varies by client and edge |
| CDN configuration | Origin protection at scale | Per-provider semantics differ |
| Application cache wrapper | Internal service-to-service calls | You implement single flight and the failure policy |
| Loading caches with refresh-after-write | In-process caches in application frameworks | Per-instance; refresh happens per node |
| Plain short TTL | When staleness must be tight | Reintroduces the latency cliff |

The in-process variant has a subtlety worth knowing: refresh-after-write in a per-instance cache means every instance refreshes independently, so a fleet of fifty nodes produces fifty refreshes per interval rather than one. That is still far better than fifty bursts, but it is not the single refresh the shared-cache version achieves.

**Trade-offs**

**Stale-while-revalidate against the alternatives**

| Approach | Latency at expiry | Source load | Staleness |
|---|---|---|---|
| Short TTL only | Cliff on every expiry | Burst per expiry | Bounded by TTL |
| Long TTL only | Rare cliff | Rare burst | Poor |
| Stale-while-revalidate | None | One refresh per interval | TTL plus stale window |
| Background refresh ahead of expiry | None | One refresh per interval | Bounded by TTL |
| No cache | Always slow | Full | None |

Proactively refreshing just before expiry achieves the same latency profile with tighter staleness, but only works for keys you know are hot — it wastes work on everything else. Stale-while-revalidate is demand-driven, which is why it generalises to the whole key space.

> **Ask before choosing it**  
> What is the largest age at which this value is still safe to serve? That number is the stale window, and it should be set from the answer rather than copied from an example. Then ask what should happen if the source is down for an hour — continuing to serve an hour-old value is often right, and occasionally unacceptable.

**How it fails**

**How it goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Source still sees a burst | Refreshes not collapsed | Single flight around the refresh |
| Very old data served | Stale window set generously and never revisited | Choose the window from the data's tolerance |
| Value evicted when the source failed | Refresh failure treated as invalidation | Keep serving; retry later |
| No benefit observed | Refresh awaited in the request path | Refresh must be out of band |
| Latency cliff returns | Entry aged past the stale window under low traffic | Longer window, or proactive refresh for hot keys |
| Freshness overstated to stakeholders | Quoting max-age rather than max-age plus window | State the real bound |
| Fleet-wide refresh storm | Per-instance caches each refreshing | Shared cache, or jitter the refresh |

> **A failing source plus a generous window serves very old data silently**  
> If refreshes keep failing, the pattern keeps serving the last good value for the whole stale window without erroring. That is usually the desired behaviour — it is a form of graceful degradation — but it means a broken source can be invisible for as long as the window allows. Alerting on refresh failures and on the proportion of responses served stale is what keeps it from being silent.

**Where it is used**

- **CDN-fronted API responses**, where the directive removes the origin burst at expiry without extra infrastructure.
- **Dashboards and read-heavy aggregates**, where a value a minute old is indistinguishable from a current one.
- **Configuration and feature-flag distribution**, which must never block a request on a control-plane fetch.
- **Pricing and catalogue endpoints**, where the freshness requirement is bounded and the read volume is high.
- **Rendered page fragments**, refreshed behind the request so navigation stays instant.

**In the interview**

The distinguishing detail is single flight; without it the pattern is usually described as a win when it has merely hidden the problem.

- **Describe the two windows explicitly** and say that the real staleness guarantee is their sum, not the freshness lifetime.
- **Volunteer that refreshes must be collapsed.** Saying this unprompted is the strongest signal in the answer.
- **Explain what happens when the refresh fails** — keep serving, extend, retry — and note that this is graceful degradation rather than an error path.
- **Point at the p99.** Framing the benefit as removing the latency cliff rather than as faster reads shows you know where the cost actually was.
- **Say where you would not use it**: anything a user acts on where being wrong costs them money.

**Practice drill**

An endpoint takes 1,000 requests per second on a single key, the source takes 400 ms, and the freshness lifetime is 60 seconds. Work out how many source calls happen at each expiry with and without stale-while-revalidate, and what each does to p99. Then choose a stale window and defend it, and say precisely what the system does — and what it alerts on — if the source is unavailable for the whole window.

**Go deeper**

Stale-while-revalidate removes the gap between an entry expiring and its replacement arriving, by continuing to serve the old value while the new one is fetched.

**The cost it removes is concentrated, not spread.** In a plainly cached endpoint almost every request is fast and the few that coincide with expiry are slow, which means the tail latency is set entirely by a moment that occurs once per lifetime. Serving stale during the refresh flattens that: no caller waits, and the distribution loses its cliff. Framing the benefit as tail latency rather than average latency is what makes the pattern's value visible, because the average barely moves.

**Single flight is not optional here.** Every request arriving during the stale window is a candidate to trigger a refresh, so without collapsing them the pattern converts one burst of misses into a continuous stream of redundant refreshes. What makes this particularly worth stating is that the symptom is invisible from the caller's side — everyone is served quickly from the stale copy — so the damage appears only at the source, in a system that looks healthy everywhere else.

**The guarantee is the sum of two numbers.** A value may legitimately be served for the freshness lifetime plus the stale window, and the second is frequently set generously because it costs nothing in normal operation. Quoting the freshness lifetime as the staleness bound is therefore a common and consequential overstatement, particularly when someone else is relying on that figure to decide whether a downstream behaviour is safe.

**Refresh failure should extend, not evict.** If the source is unavailable, discarding a usable value to replace it with nothing serves the reader worse than continuing with the old one. Treating a failed refresh as a reason to keep serving turns the pattern into a degradation mechanism that carries a system through a source outage — which is genuinely valuable, and also means a broken source can remain invisible for the length of the window. Alerting on refresh failure rate and on the proportion of responses served stale is what closes that gap.

**Demand-driven refresh is why it generalises.** Refreshing proactively before expiry achieves the same latency profile with tighter staleness, but it requires knowing which keys are worth refreshing, and spends work on keys nobody asks for. Stale-while-revalidate refreshes exactly what is being requested, which makes it applicable across a whole key space without any knowledge of which parts are hot.

**Per-instance caches dilute the collapsing.** In a shared cache one refresh serves the fleet. In an in-process cache each node refreshes independently, so fifty instances produce fifty refreshes per interval — far better than fifty bursts, but not the single call the shared arrangement gives. Knowing which of the two you have is what makes the source-load estimate correct rather than optimistic.

**Related patterns:** Cache Aside · Single Flight · Cache Invalidation · CDN

---

### Single Flight

*Collapse concurrent requests for the same missing value into one execution and share the result, so a popular key costs one fetch rather than hundreds.*

> **When you hear…** A hot key expiring takes the database down · duplicate identical queries in flight · the miss path is expensive

**Flow:** `Concurrent misses` → `First caller executes` → `Others wait on it` → `One source call` → `Result shared`

**The problem**

A cached homepage fragment expires. In the two hundred milliseconds it takes to regenerate, four hundred requests arrive, all find the key missing, and all start generating it. The source receives four hundred identical expensive queries for a result that four hundred callers are about to discard all but one copy of.

The cache did its job right up to the moment it mattered most. Popularity is exactly what makes a key worth caching, and exactly what makes its expiry dangerous.

> **Concurrent identical work is work you can simply not do**  
> If four hundred callers want the same value and none of them has it, only the first needs to fetch it — the rest can wait for that fetch and share its result. Nothing is lost, because they were all going to receive identical answers anyway, and the source sees one request instead of four hundred.

**Mental model**

A small registry of in-flight operations keyed by what is being fetched. The first caller creates an entry and does the work; everyone else finds the entry and waits on it.

1. **Arrive** — A caller needs a value that is not cached.
2. **Claim or join** — It either becomes the executor for that key, or finds an existing attempt and waits.
3. **Execute once** — Only the executor touches the source.
4. **Share** — The result — or the error — is delivered to every waiter.
5. **Release** — The entry is removed so the next miss starts fresh.

> **The waiters inherit the executor's fate, including its latency**  
> Everyone waiting is now coupled to one operation. If it is slow, they are all slow; if it hangs without a timeout, they all hang. Collapsing requests concentrates risk as well as work, which is why the executor needs a deadline and the waiters need to be released when it expires.

**How it works**

**The registry**

```text
inFlight = map of key -> pending result

get(key):
  existing = inFlight[key]
  if existing: return await existing       <- join
  promise = run(key)                       <- claim
  inFlight[key] = promise
  try:    return await promise
  finally: delete inFlight[key]            <- always release

WHY THE RELEASE MUST BE IN A FINALLY
  an executor that throws and never clears the entry
  leaves every future caller waiting on a promise
  that will never settle again
  -> the key becomes permanently unservable, which is
     far worse than the stampede it was preventing

SCOPE
  per process   collapses within one instance
                50 instances -> up to 50 source calls
  shared lock   collapses across the fleet -> 1 call
                but now a lock, its ttl, and what to
                do when the holder dies

THE DEFAULT IS PER PROCESS
  going from 400 calls to 50 removes the danger;
  going from 50 to 1 adds a distributed lock and its
  failure modes for a much smaller gain
```

1. **Release the entry in a finally block** — An executor that dies holding the slot makes the key permanently unservable — worse than the original problem.
2. **Give the executor a deadline** — Everyone waiting shares its latency, so an unbounded fetch hangs every one of them.
3. **Share failures as well as results** — Waiters must be told the fetch failed, not silently left to time out.
4. **Do not cache the failure** — The next request should be free to try again immediately.
5. **Prefer per-process scope first** — It removes almost all of the burst without introducing a distributed lock.
6. **Key on exactly what identifies the work** — Too coarse and callers receive someone else's result; too fine and nothing collapses.

**What collapsing actually saves**

```text
HOT KEY, 2,000 req/s, source takes 200 ms
expiry every 60 s

WITHOUT SINGLE FLIGHT
  misses during the regeneration window:
    2,000 req/s x 0.2 s = 400 concurrent misses
  -> 400 identical source calls, every 60 s
  -> the source must be sized for the burst, not the
     steady state

WITH SINGLE FLIGHT (per process, 8 instances)
  each instance collapses its own share
  -> 8 source calls instead of 400
  -> 50x reduction with no coordination

WITH A SHARED LOCK
  -> 1 source call
  -> 8x better than per-process, at the cost of a
     lock, a lease ttl, and a stampede if the holder
     dies without releasing

DIMINISHING RETURNS
  the first 50x is free; the last 8x costs a
  distributed system
```

| Metric | Value | Note |
|---|---|---|
| Concurrent misses | 400 | per expiry |
| Per-process | 8 calls | **50× cheaper, free** |
| Shared lock | 1 call | adds a lock |
| Release | finally | or key is dead |

> **Collapsing too coarsely serves one caller's answer to another**  
> The key must identify the work precisely. Collapsing on the endpoint rather than the endpoint plus its parameters means the first caller's result is handed to everyone, regardless of what they asked for. The failure is silent, plausible-looking, and extremely hard to reproduce, because it only appears under concurrency.

**Technologies**

| Where it lives | Best for | Trade |
|---|---|---|
| Language primitive (Go singleflight, Java loading cache) | In-process collapsing with no extra infrastructure | Per instance only |
| Promise or future registry | Any async runtime, a few lines of code | You own the release and timeout |
| DataLoader-style batching | Collapsing different keys into one source query | Request-scoped rather than long-lived |
| Distributed lock | One fetch across the whole fleet | Lock lifetime, holder death, added failure modes |
| Stale-while-revalidate | Avoiding the empty window altogether | Serves slightly old data |

Stale-while-revalidate belongs on this list because it addresses the same stampede from the other direction: rather than collapsing the callers who find an empty key, it arranges for the key never to be empty. Where staleness is acceptable, combining the two is stronger than either — no gap, and one refresh.

**Trade-offs**

**Ways to stop a stampede**

| Approach | Source calls at expiry | Cost | Serves stale |
|---|---|---|---|
| Nothing | One per concurrent miss | None | No |
| Per-process single flight | One per instance | Negligible | No |
| Fleet-wide lock | One | A distributed lock | No |
| Stale-while-revalidate | One per interval | Two-window config | Yes |
| Jittered expiry | Spread out, not reduced | Negligible | No |
| Proactive refresh | One, before expiry | Must know hot keys | No |

Jittering is worth pairing with any of these: it does not reduce the number of fetches but it stops many different keys expiring at the same instant, which is a separate and equally real source of synchronised load.

> **Ask before choosing it**  
> How many instances are there, and is the remaining per-instance burst actually a problem? If eight concurrent fetches are harmless, per-process collapsing is the whole answer and a distributed lock is over-engineering. Ask too whether the value could simply be served stale — that removes the empty window rather than managing it.

**How it fails**

**How single flight goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| A key becomes permanently unservable | Entry not released after the executor threw | Release in a finally block, always |
| Every caller hangs together | Executor has no timeout | Deadline on the executor; release waiters when it expires |
| Waiters get the wrong data | Key too coarse to identify the work | Include every parameter that changes the result |
| Stampede persists across the fleet | Per-process scope with many instances | Accept it, or add a shared lock deliberately |
| A failure is served repeatedly | Error cached under the key | Share the error with current waiters; do not cache it |
| Burst returns when the lock holder dies | Distributed lock with no lease or handover | Lease with expiry; next caller takes over |
| Latency worse than without it | Waiters queued behind a slow executor with no cap | Bound the wait; let late arrivals fail fast |

> **The unreleased entry is the failure that turns a mitigation into an outage**  
> If the executor throws and the registry entry survives, every subsequent caller joins a result that will never arrive. The key stops being servable entirely, and because the mechanism was introduced to protect that exact key, it is invariably one of the most important in the system. The release has to be unconditional.

**Where it is used**

- **Cache fill paths on hot keys**, which is the canonical use and the reason the pattern exists.
- **Expensive aggregations** — a dashboard query that several users trigger simultaneously.
- **Token and credential refresh**, where concurrent expiry would otherwise produce a burst of identical auth calls.
- **Graph and API layers**, usually combined with batching so different keys also collapse into one source query.
- **Configuration fetches on startup**, where an entire fleet restarting would otherwise hit the control plane together.

**In the interview**

This is a small pattern, so the depth shows in the failure handling rather than the mechanism.

- **Say the release is unconditional** and explain what an orphaned entry does — it is the detail that separates having used this from having read about it.
- **Raise the timeout.** Waiters inherit the executor's latency, so an unbounded fetch hangs all of them at once.
- **Distinguish per-process from fleet-wide** and argue that per-process is usually sufficient; reaching straight for a distributed lock is a red flag.
- **Mention key granularity.** Collapsing too coarsely serves one caller's answer to another, and it only shows up under concurrency.
- **Offer stale-while-revalidate as the complementary fix** — it removes the empty window instead of managing the crowd at it.

**Practice drill**

A key serving 2,000 requests per second expires every 60 seconds, and regenerating it takes 200 milliseconds. Work out how many concurrent misses occur at expiry, then how many source calls remain with per-process single flight across 8 instances, and then with a fleet-wide lock. Decide which you would ship and justify it. Finally, describe exactly what happens if the executor throws halfway through, and what your implementation does about it.

**Go deeper**

Single flight collapses concurrent identical work into one execution whose result is shared, which turns a key's popularity from a liability at expiry back into an asset.

**The waste it removes is pure.** Every caller in the window wants the same value and will receive the same bytes; the only reason they each fetch it is that none of them knows the others exist. A small registry of in-flight work supplies exactly that knowledge, and the reduction is proportional to how popular the key is — which means the mechanism is most effective precisely where the danger is greatest.

**Releasing the entry is the critical detail.** An executor that fails while holding the slot leaves every later caller waiting on a result that will never arrive, and the key becomes permanently unservable. Because this pattern is applied to the hottest keys in a system, the consequence is that the most important value in the cache stops being fetchable at all — a far worse outcome than the stampede being prevented. The release therefore belongs in a finally, not on the success path.

**Collapsing concentrates latency as well as work.** Once four hundred callers are waiting on one fetch, that fetch's duration is every caller's duration and its failure is every caller's failure. A deadline on the executor and a bound on how long waiters will wait are what keep a slow source from producing a synchronised hang, and they are frequently omitted because the happy path works without them.

**Per-process scope is usually the right amount.** Collapsing within an instance takes a burst of several hundred down to one call per instance — a reduction of one or two orders of magnitude for a few lines of code and no coordination. Collapsing across the fleet takes that remainder to one, which is a much smaller improvement bought with a distributed lock, a lease, and a new failure mode when the holder dies. Starting per-process and only escalating with evidence is the proportionate path.

**Key granularity is a correctness concern.** The registry key must identify the work exactly: too coarse and callers asking different questions are served the first answer, too fine and nothing collapses. The coarse failure is the dangerous one because it produces plausible wrong data, appears only under concurrency, and is close to impossible to reproduce from a bug report.

**It composes with the patterns around it rather than competing.** Stale-while-revalidate prevents the empty window that creates the crowd; jittered expiry stops many keys becoming empty at the same instant; batching collapses different keys into one source query where single flight collapses identical ones. A cache that is genuinely hardened against stampedes usually has three of these working together, and knowing which one addresses which part of the problem is what makes the combination deliberate rather than accumulated.

**Related patterns:** Cache Aside · Stale While Revalidate · Cache Invalidation · Read Through Cache

---

## Data Architecture

### CQRS

*Separate the model that accepts changes from the models that answer questions, so each can be shaped for its own job instead of compromising between them.*

> **When you hear…** The write schema is fighting the query patterns · reads and writes scale differently · one table serves ten incompatible views

**Flow:** `Command` → `Write model` → `Event or sync` → `Read models` → `Query`

**The problem**

An order table is normalised for correctness: writes touch one row, constraints hold, nothing is duplicated. Then the product needs an order history view, a seller dashboard, a fulfilment queue and a monthly report, and each of those is a five-table join with different filters and sort orders. The indexes required to make them fast slow every write down, and no single shape satisfies all of them.

The pressure is structural, not a tuning problem. A schema optimised for accepting a change correctly is rarely the schema optimised for answering a question quickly, and adding indexes until both work means paying write cost for read convenience on every single insert.

> **Stop making one model serve two opposing jobs**  
> Commands need invariants, constraints and a small write surface. Queries need denormalised, pre-joined shapes matched to how they are actually asked. Once you accept that these are different requirements, keeping one model for writes and deriving separate models for reads stops being duplication and starts being specialisation.

**Mental model**

One side accepts commands and enforces rules. The other side holds read models built for specific questions. A propagation mechanism keeps the second in step with the first, and it does so after the fact.

1. **Command** — An intent to change something. Validated against the write model's invariants.
2. **Write model** — Normalised, constrained, the source of truth. Small and correct.
3. **Propagate** — Events, change data capture or a sync job carry the change outward.
4. **Read model** — A shape built for one query. Denormalised, indexed for its access pattern, disposable.
5. **Query** — Answered from a read model with no joins and no contention with writers.

> **The read side is behind, and the interface has to admit it**  
> Propagation takes time. A user who submits a change and immediately reads it back can see the previous state, which looks like the write was lost. This is the defining cost of CQRS and it is a product problem, not only a technical one — the flows that read back have to be designed knowing the answer may be stale.

**How it works**

**The two sides and what connects them**

```text
COMMAND SIDE
  validate against invariants
  write to the normalised model
  emit an event describing what changed
  acknowledge

PROPAGATION  (pick one)
  domain events     explicit, intentional, needs an outbox
                    so publishing cannot be lost
  change data capture
                    read the database log; nothing to
                    add to the write path
  periodic sync     simplest, coarsest, highest lag

READ SIDE
  consume the change
  update one or more read models
  each shaped for a single query

THE READ MODEL IS DISPOSABLE
  it can always be rebuilt from the write model or
  from the event history
  -> that is what makes adding a new view cheap:
     build the projection, replay, start serving
  -> and what makes a projection bug recoverable:
     fix the code, drop the model, rebuild

WHAT MUST NOT HAPPEN
  a command reading from a read model to decide
  whether it is allowed
  -> the read model is stale, so the invariant is
     checked against data that may already be wrong
  -> invariants are enforced on the write side only
```

1. **Enforce every invariant on the write side** — Checking a rule against a stale read model means enforcing it against data that has already changed.
2. **Make read models disposable and rebuildable** — That property is what makes new views cheap and projection bugs recoverable.
3. **Publish changes through an outbox** — A change committed but not published leaves the read side permanently behind with nothing to detect it.
4. **Shape each read model for one question** — A read model serving several queries is drifting back toward the compromise you left.
5. **Surface the lag** — Measure write-to-visible time; it is the number that tells you whether the design is working.
6. **Do not apply it everywhere** — Most of a system is simple enough that one model is correct and cheaper.

**Where the cost and benefit land**

```text
BEFORE
  one orders table, normalised
  6 query shapes, each a multi-table join
  14 indexes added to make them acceptable
  -> every insert maintains 14 indexes
  -> reports still slow, writes now slower

AFTER
  write model: orders, 2 indexes, fast inserts
  read models:
    order_history_by_customer
    seller_dashboard_rows
    fulfilment_queue
    monthly_rollup
  each denormalised, single-table, exactly indexed

  write cost   14 index updates -> 2
  read cost    joins -> single-key lookups
  new view     add a projection and replay, no schema
               change on the write side

WHAT IT COSTS
  four more stores to operate and monitor
  propagation lag between write and visibility
  projection code that can be wrong, silently
  no cross-model transaction: the two sides are
    eventually consistent by construction
```

| Metric | Value | Note |
|---|---|---|
| Write indexes | 14 → 2 | **faster writes** |
| Read queries | joins → lookups | shaped per question |
| New view | projection + replay | no write-side change |
| Cost | lag + 4 stores | eventual consistency |

> **Most systems that adopt CQRS did not need it**  
> The pattern earns its complexity when reads and writes genuinely have incompatible shapes or wildly different scale. A CRUD application with a reporting page needs a replica and a couple of indexes, not a second model and a propagation pipeline. Adopting it by default means paying for eventual consistency, extra stores and projection code in a system whose queries a single well-indexed table would have answered.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Write store | Relational database | Constraints and transactions are the point |
| Propagation | Outbox plus broker, or change data capture | CDC avoids touching the write path |
| Read store | Denormalised tables, a document store, a search index | Chosen per query shape |
| Rebuild | Replay from events or re-derive from the write model | The property that makes projections safe |
| Simpler alternative | Read replica with its own indexes | Often the right answer |

The read-replica row deserves emphasis because it is the honest competitor: it gives read scaling and some index freedom for almost no design cost, and it keeps one model. CQRS becomes worth it when the required read shapes cannot be expressed as indexes over the write schema at all.

**Trade-offs**

**CQRS against the alternatives**

| Approach | Read flexibility | Consistency | Operational cost |
|---|---|---|---|
| Single model | Limited by the write schema | Strong | Lowest |
| Read replica | Index freedom, same shape | Replication lag | Low |
| Materialised views in the database | Pre-joined shapes | Refresh lag | Moderate |
| CQRS | Any shape, per query | Eventually consistent | High |
| CQRS with event sourcing | Any shape, plus history | Eventually consistent | Highest |

The table is deliberately ordered by escalating cost, because the right move is to climb it only as far as the problem requires. Each step buys read flexibility and charges in consistency and operations, and stopping at the first row that solves the actual problem is almost always correct.

> **Ask before choosing it**  
> Can the required queries be served by indexes over the existing schema, or by a replica? If yes, the pattern is not needed. If no, can the product tolerate a user not seeing their own change immediately — and if not, which specific flows need a read-your-writes path around the read model?

**How it fails**

**How CQRS goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Users think their change was lost | Read model behind and the interface does not show it | Read-your-writes for those flows; surface pending state |
| Read model permanently out of date | Change committed but the event was never published | Transactional outbox |
| Invariant violated | Rule checked against a stale read model | Enforce invariants on the write side only |
| Projection silently wrong | Bug in the projection with no comparison against source | Reconcile periodically; rebuild from source |
| Duplicate rows in a read model | Events redelivered and the projection is not idempotent | Idempotent projections keyed by event id |
| Rebuild takes days | No snapshot and a very long history | Snapshot the model; rebuild forward from there |
| Complexity with no benefit | Adopted by default rather than for a real constraint | Collapse back to one model and a replica |

> **Enforcing an invariant against a read model is the failure that corrupts data**  
> If a command checks whether something is allowed by consulting a read model, it is checking against a view that may already be behind. Two commands can then both be told they are permitted when only one is — the classic double-spend shape. Invariants belong exclusively to the write side, where the current state and its constraints actually live.

**Where it is used**

- **Order and fulfilment platforms**, where the write model protects invariants and many parties need very different views.
- **Trading and ledger systems**, which pair a strict write path with denormalised reporting models.
- **Content platforms** that serve one authored record through feeds, search indexes and rendered pages.
- **Analytics and reporting layers**, effectively CQRS with a batch propagation mechanism.
- **Systems with extreme read-to-write asymmetry**, where the read side is scaled independently of the write side.

**In the interview**

The strongest answer treats CQRS as an expensive tool and spends most of its words on when not to reach for it.

- **Start by ruling it out.** Say what a replica and better indexes would solve, and adopt the pattern only for what they cannot.
- **Name the defining cost**: the read side is eventually consistent, and the flows that read back immediately are a product problem.
- **Insist invariants live on the write side** and explain the double-spend shape that results from checking them against a projection.
- **Mention the outbox** without being asked — a change committed but not published is the failure that silently desynchronises the two sides.
- **Say that read models are disposable.** Being able to drop and rebuild one is what makes both new views and projection bugs cheap.

**Practice drill**

An orders table currently carries 14 indexes to serve six reporting queries, and inserts have become slow. Decide first whether a read replica with its own indexes solves it, and say what would have to be true for that to fail. Assuming it does fail, design the CQRS split: name the read models, choose the propagation mechanism and justify it, state the expected write-to-visible lag, and identify which user flows need a read-your-writes path.

**Go deeper**

CQRS separates the model that accepts change from the models that answer questions, because those two jobs pull a schema in opposite directions.

**The tension it resolves is real and structural.** A write model wants to be normalised and constrained so that a change touches one place and invariants are cheap to enforce. A read model wants to be denormalised and pre-joined so that a question is one lookup. Serving both from one schema means adding indexes until reads are tolerable, which makes every write pay for query convenience, and still leaves complex queries doing joins under load.

**Eventual consistency is the price, and it is paid in the product.** Propagation takes time, so a user who changes something and immediately looks at it can see the old value — which reads as the change having failed. No amount of engineering removes this; it can only be worked around for specific flows, by reading from the write model after a write, by showing pending state, or by holding the change client-side until it appears. Those decisions belong in the design, not in a later bug report.

**Invariants must never be checked against a read model.** This is the sharpest rule in the pattern. A projection is behind by construction, so a rule evaluated against it is evaluated against data that may already have changed — and two concurrent commands can both be told they are permitted when only one should be. Every constraint therefore lives on the write side, where the current state does, and the read side is strictly for answering questions nobody acts on transactionally.

**Disposability is the property worth protecting.** A read model that can be dropped and rebuilt from the write model or the event history makes two expensive things cheap: adding a new view becomes writing a projection and replaying, with no change to the write schema, and a projection bug becomes fix-and-rebuild rather than a data repair. Designs that let read models accumulate state which cannot be re-derived lose this, and with it most of the pattern's flexibility.

**Publication has to be transactional with the write.** If a change commits and the event announcing it is lost, the read side is permanently behind with nothing to detect the gap — the two stores simply disagree forever. Writing the event to an outbox inside the same transaction and publishing from there is what closes it, and change data capture achieves the same by deriving events from the database log rather than from application code.

**Most adoptions are premature.** The pattern is frequently applied because it is well known rather than because a constraint demanded it, and the result is a CRUD application paying for two stores, a pipeline, projection code and eventual consistency to serve queries that an indexed table answered adequately. The disciplined sequence is to exhaust indexing, then a replica, then database-level materialised views, and to reach CQRS only when the required read shapes genuinely cannot be expressed over the write schema.

**Related patterns:** Event Sourcing · Materialized View · Change Data Capture · Transactional Outbox

---

### Event Sourcing

*Store the sequence of changes rather than the current state, so history is the source of truth and any view of it can be rebuilt.*

> **When you hear…** You need to know how it got this way · an audit trail is a requirement · past state must be reconstructible

**Flow:** `Command` → `Event appended` → `Immutable log` → `State rebuilt` → `Snapshots`

**The problem**

An account balance is wrong. The row says one figure, the customer says another, and the only way to find out what happened is to read application logs that were never designed to answer the question. Every update overwrote its predecessor, so the information needed to explain the current value was destroyed as it was produced.

The same gap appears whenever someone asks what a record looked like last Tuesday, why a status changed, or who made a decision. A table of current state answers none of those, because it deliberately keeps only the latest answer.

> **Keep the changes and derive the state, rather than keeping the state and losing the changes**  
> If every change is appended as an immutable fact and current state is computed by replaying them, nothing is ever destroyed. Explaining a value becomes reading the facts that produced it, past state becomes replay up to a point in time, and a new view becomes a new way of folding the same history.

**Mental model**

The log is the database. State is a cached fold over it — useful, but always re-derivable and never authoritative.

1. **Command** — A request to change something, validated against current state.
2. **Event** — An immutable statement that something happened. Past tense, never modified, never deleted.
3. **Append** — The event joins the stream for that entity. This is the only write.
4. **Fold** — Replaying events in order produces current state.
5. **Snapshot** — A stored fold at a point in the stream, so replay does not start from the beginning.

> **Events are facts and facts cannot be edited**  
> A wrong event is corrected by appending a compensating event, never by changing or deleting the original — the whole value of the log is that it is an unaltered record. Teams that reach into the store to fix a bad event destroy the property they adopted the pattern for, and every projection rebuilt afterwards silently disagrees with anything derived earlier.

**How it works**

**Append, fold, snapshot**

```text
WRITE
  load current state for the entity
    (from a snapshot plus later events)
  validate the command against it
  append event(s)
  -> the append is the commit

READ CURRENT STATE
  state = fold(events, from: snapshot)

SNAPSHOTS
  every N events, store the folded state with the
  stream position it represents
  -> replay starts there instead of at event 1
  -> a snapshot is a cache: wrong snapshots are
     discarded and re-derived, never repaired

CONCURRENCY
  append with an expected version:
    appendIfVersionIs(stream, expectedVersion, event)
  -> two concurrent commands cannot both succeed on
     stale state; the loser retries against the new
     state
  -> this is optimistic concurrency, and it is the
     mechanism that makes invariants safe

EVENT SCHEMA WILL CHANGE
  events are stored forever, so the code that reads
  them must handle every version ever written
  -> upcast old events on read, or version the type
  -> you cannot migrate the past: it already happened
```

1. **Append with an expected version** — Optimistic concurrency is what stops two commands both acting on stale state.
2. **Correct with compensating events** — Editing history destroys the property the pattern exists to provide.
3. **Treat snapshots as a cache** — They must be discardable and re-derivable, never a second source of truth.
4. **Version events from the first day** — Stored events outlive the code that wrote them, and old versions are read forever.
5. **Keep events about the domain, not the database** — An event named for what happened survives refactoring; one named for a table does not.
6. **Plan the rebuild time** — How long a full replay takes determines how recoverable a projection bug is.

**What replay costs and what snapshots fix**

```text
ONE ACCOUNT, 5 YEARS, 40,000 EVENTS

NO SNAPSHOTS
  every read folds 40,000 events
  at ~2 us per event -> ~80 ms per load
  -> unusable on a hot path

SNAPSHOT EVERY 500 EVENTS
  load snapshot + up to 500 events
  -> ~1 ms
  -> snapshots are not an optimisation, they are what
     makes the pattern viable at all

FULL PROJECTION REBUILD
  100 million events across all streams
  at 50,000 events/s -> ~33 minutes
  -> that number is your recovery time for a bad
     projection, so measure it before you need it

STORAGE
  events accumulate and are never deleted
  -> growth is linear in activity, forever
  -> retention is a legal question, not a technical
     one, because deleting an event changes history
```

| Metric | Value | Note |
|---|---|---|
| Fold, no snapshot | 80 ms | unusable |
| With snapshots | ~1 ms | **makes it viable** |
| Full rebuild | ~33 min | your recovery time |
| Storage | grows forever | retention is legal |

> **The right to erasure collides with an immutable log**  
> Regulations that require deleting personal data are awkward against a store whose defining property is that nothing is deleted. The usual resolution is to keep personal data outside the events and reference it by key, so erasing the referenced record renders the events non-identifying without altering them. Deciding this after the log exists is considerably harder than deciding it before.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Event store | Purpose-built stores, or an append-only table | A table with a version column is often enough |
| Streaming log | Kafka-style partitioned log | Good transport; retention and per-entity replay need care |
| Snapshots | Serialised state in the same store | A cache, never a second authority |
| Projections | Any read store, one per query shape | This is CQRS on the read side |
| Simpler alternative | Audit table alongside normal state | Gives history without changing the write model |

The audit-table row is the one most teams should consider first: keeping current state normally and writing an append-only history alongside it delivers the traceability people actually asked for, without making replay the primary read path or event versioning a permanent obligation.

**Trade-offs**

**Event sourcing against the alternatives**

| Approach | History | Read cost | Complexity |
|---|---|---|---|
| Current state only | None | Direct | Lowest |
| State plus audit table | Complete, secondary | Direct | Low |
| Temporal tables | Point-in-time state | Direct | Moderate |
| Event sourcing | Complete, primary | Fold or projection | High |
| Event sourcing with CQRS | Complete, primary | Projection lookups | Highest |

The distinction that matters is whether history is primary or secondary. An audit table gives you the record without making every read a fold and every schema change a versioning problem; event sourcing makes history the truth, which is more powerful and considerably more demanding.

> **Ask before choosing it**  
> Is the requirement to explain how state was reached, or merely to have a record of changes? The first argues for event sourcing; the second is satisfied by an audit log at a fraction of the cost. Then ask how long a full rebuild takes and whether personal data will ever need erasing, because both answers are much cheaper to act on before the log exists.

**How it fails**

**How event sourcing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Reads become slow as entities age | Folding long streams with no snapshots | Snapshot at intervals; fold forward from there |
| Two commands both act on stale state | Appending without an expected version | Optimistic concurrency on append |
| Old events can no longer be read | Event schema changed with no versioning | Version from day one; upcast on read |
| History quietly altered | A bad event edited or deleted in place | Compensating events only |
| Projection rebuild takes too long to be useful | No snapshots and a very large log | Measure rebuild time; snapshot projections too |
| Personal data cannot be erased | Identifying data embedded in immutable events | Reference it by key; erase the referenced record |
| Complexity with no payoff | Adopted where an audit table was the requirement | Keep state primary, history secondary |

> **Events named after database operations age badly**  
> An event called something like row updated carries no meaning and cannot be interpreted years later without the code that wrote it. Events named for what happened in the domain — an order was dispatched, a limit was raised — remain readable and remain correct across refactors. Because these records are permanent, this naming decision is one of the few in software that genuinely cannot be undone.

**Where it is used**

- **Ledgers and accounting systems**, where the sequence of entries is the record and balances are derived.
- **Order lifecycles**, where every transition matters and the path to the current status is itself the useful information.
- **Collaborative editing**, where the operation sequence is the document and state is a fold over it.
- **Regulated domains** that must reconstruct what was known at a point in time.
- **Version control systems**, which are event sourcing over a filesystem with snapshots and replay.

**In the interview**

Interviewers are usually probing whether you know the operational obligations rather than the concept.

- **Distinguish the requirement.** Wanting a record of changes is an audit table; wanting state to be derived from history is event sourcing, and only the second justifies the cost.
- **Bring up snapshots immediately.** Without them, replay makes reads slower as entities age, so they are a viability requirement rather than an optimisation.
- **Mention optimistic concurrency on append** — the expected-version check is what makes invariants hold under concurrent commands.
- **Say events are corrected by compensation**, never edited, and explain why in-place edits destroy the pattern's only real guarantee.
- **Raise event versioning and erasure unprompted.** Both are permanent obligations that are cheap to plan for and very expensive to retrofit.

**Practice drill**

An account accumulates roughly 8,000 events a year and must be readable on a hot path. Work out the fold cost after five years with no snapshots, choose a snapshot interval and show the resulting read cost. Then size a full projection rebuild across 100 million events and state what that number means operationally. Finally, decide where you would put the customer's name and address so that an erasure request can be honoured without altering history.

**Go deeper**

Event sourcing stores the sequence of changes as the source of truth and derives current state from it, which preserves the information that overwriting destroys.

**It answers questions that current-state storage cannot.** How a value was reached, what a record looked like at a past moment, which decision produced a status — none of these survive an update that overwrites its predecessor. Where those questions are genuinely part of the requirement rather than a nice-to-have, keeping the changes and deriving the state is the only arrangement that answers them reliably, because the answer is the data rather than a reconstruction from logs.

**Snapshots are a viability requirement.** Folding a stream on every read means reads get slower as entities accumulate history, so a long-lived account eventually becomes unusable on a hot path. Storing a periodic fold with its stream position bounds the work to the events since that point. Crucially a snapshot must remain a cache — discardable and re-derivable — because the moment it is treated as authoritative the log stops being the truth and the pattern's guarantee evaporates.

**Optimistic concurrency on append is what keeps invariants safe.** A command loads state, validates against it, and appends; without an expected-version check two concurrent commands can both validate against the same stale state and both succeed, producing exactly the overdraft or double-allocation the invariant existed to prevent. Appending only if the stream is still at the version that was read makes the loser retry against current state, which is both correct and cheap.

**Stored events outlive the code that wrote them, which makes two decisions permanent.** Schema versioning has to exist from the first event, because old versions will be read for as long as the system lives and the past cannot be migrated — it already happened. And naming has to describe the domain rather than the storage operation, because an event named for a table change is uninterpretable once that table is gone, whereas one named for what occurred remains meaningful indefinitely.

**History being immutable collides with the right to erasure.** A log whose defining property is that nothing is removed sits badly with an obligation to delete personal data on request. The workable resolution is to keep identifying data outside the events and refer to it by key, so that erasing the referenced record leaves the events intact but non-identifying. This is straightforward to design in and extremely awkward to retrofit once a large log exists, which makes it one of the first questions rather than a later one.

**The requirement is usually weaker than the pattern.** Most teams asking for event sourcing want traceability — a dependable record of what changed and when — which an append-only audit table alongside conventional state provides at a small fraction of the cost and with none of the versioning, snapshotting or rebuild obligations. Event sourcing earns its complexity specifically when state must be derived from history rather than merely accompanied by it, and separating those two requirements early is the most valuable thing to do in the conversation.

**Related patterns:** CQRS · Materialized View · Consistent Snapshot · Transactional Outbox

---

### Change Data Capture

*Read the database's own replication log to emit a stream of row changes, so downstream systems stay in step without the application publishing anything.*

> **When you hear…** A search index or cache keeps drifting from the database · dual writes keep going wrong · you need change events from a system you cannot modify

**Flow:** `Database write` → `Commit log` → `CDC connector` → `Change stream` → `Downstream consumers`

**The problem**

An order service writes to its database and then publishes an event so the search index, the cache and the analytics warehouse can follow. Sometimes the database commit succeeds and the publish fails, and the three downstream systems are silently behind forever. Sometimes the publish succeeds and the transaction rolls back, and they are ahead of a change that never happened.

Adding retries does not close it. There is no way to make two separate systems — a database and a broker — commit atomically from application code, so every dual write has a window in which exactly one of them succeeded.

> **The database already keeps a perfect, ordered record of every change**  
> Replication logs exist so that replicas can follow a primary exactly. Reading that log gives a stream of committed changes in commit order, derived from the same artefact that guarantees durability — so it cannot disagree with the database, and the application does not have to publish anything at all.

**Mental model**

A connector pretends to be a replica. The database streams it the same log it would send to any follower, and the connector turns each entry into a change event for consumers that are not databases.

1. **Commit** — The application writes normally. It knows nothing about CDC.
2. **Log** — The write lands in the transaction log, which is how durability works already.
3. **Capture** — A connector reads the log from a recorded position, as a replica would.
4. **Emit** — Each committed row change becomes an event with before and after images.
5. **Checkpoint** — The connector stores its position so a restart resumes rather than replays everything.

> **CDC gives you rows, not intent**  
> The log records that a column changed from one value to another. It does not record why — whether a status moved because an order was cancelled or because a refund completed. Consumers that need the business meaning have to reconstruct it from column diffs, which is brittle and breaks whenever the schema is refactored. Where intent matters, domain events published through an outbox carry it and CDC does not.

**How it works**

**How capture actually works**

```text
DATABASE                    CONNECTOR

commit txn  ---> WAL/binlog
                      |
                      +---> read from stored position
                            decode row changes
                            emit {before, after, op, ts}
                            checkpoint position

WHAT YOU GET PER EVENT
  op        insert | update | delete
  before    the row as it was (update/delete)
  after     the row as it is  (insert/update)
  position  log sequence number, for ordering
  commit ts when the transaction committed

ORDERING GUARANTEE
  events arrive in COMMIT order, per log
  -> a change to A that happened before a change to B
     is emitted before it
  -> partitioning downstream by primary key preserves
     per-row order, which is what consumers need

THE SNAPSHOT PROBLEM
  the log only holds recent history
  -> a new consumer cannot replay from the beginning
  -> so CDC starts with a consistent snapshot of the
     table, then switches to the log at exactly the
     position the snapshot was taken
  -> getting that handover wrong is how rows are
     duplicated or missed at startup
```

1. **Take a consistent snapshot before streaming** — The log does not reach back to the beginning of the table; the handover position must match the snapshot exactly.
2. **Partition downstream by primary key** — Per-row ordering is what consumers depend on, and it survives parallelism only if the key routes the event.
3. **Make consumers idempotent** — Delivery is at-least-once, and a connector restart replays from the last checkpoint.
4. **Watch replication slot lag** — An unconsumed slot stops the database reclaiming log space and can fill the disk.
5. **Treat schema changes as events too** — A column rename reaches consumers as a structural change they must tolerate.
6. **Do not use it where intent matters** — Row diffs cannot express why something changed; domain events can.

**Dual write against CDC**

```text
DUAL WRITE  (the thing being replaced)
  db.commit(order)
  broker.publish(orderChanged)     <- can fail
  -> commit succeeded, publish failed:
     downstream permanently behind, no error anywhere
  -> publish succeeded, commit rolled back:
     downstream has an event for a change that
     never happened
  no amount of retrying makes two systems atomic

CDC
  db.commit(order)                 <- the only write
  ...connector reads the committed log...
  -> the event exists if and only if the change
     committed
  -> ordering comes free, from the log
  -> the application has no publishing code at all

WHAT CDC DOES NOT FIX
  the consumer can still fail to apply an event
  -> at-least-once delivery + idempotent apply
  -> and lag is real: downstream is behind by the
     capture and processing time
```

| Metric | Value | Note |
|---|---|---|
| Dual write | two systems | no atomicity |
| CDC | one write | **event iff committed** |
| Ordering | commit order | free from the log |
| Delivery | at-least-once | idempotent consumers |

> **An unconsumed replication slot can take the database down**  
> While a slot exists the database must retain every log segment the consumer has not acknowledged. A connector that stops — crashed, misconfigured, paused for a deployment — makes the log grow without bound, and a full disk on the primary is a far worse outcome than a stale search index. Slot lag needs an alert with a threshold well below the disk headroom, and a policy for dropping a slot that is never coming back.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Postgres | Logical replication slots, wal2json, pgoutput | Slot retention is the operational risk |
| MySQL | Binlog in row format | Statement format is unusable for CDC |
| Connector | Debezium and similar | Handles snapshot, resume and schema history |
| Transport | Kafka-style partitioned log | Key by primary key to preserve per-row order |
| Managed | Cloud database change streams | Less to run; less control over snapshot behaviour |
| Alternative | Transactional outbox | Carries intent; requires application changes |

The outbox row is the genuine alternative rather than a lesser option. It also guarantees the event exists if and only if the change committed, because the event is written in the same transaction — but it expresses domain intent instead of column diffs, at the cost of the application having to participate.

**Trade-offs**

**CDC against the alternatives**

| Approach | Atomic with the write | Carries intent | Touches the application |
|---|---|---|---|
| Dual write | No | Yes | Yes |
| Transactional outbox | Yes | Yes | Yes |
| Change data capture | Yes | No — row diffs only | No |
| Periodic diff or full reload | Yes, coarsely | No | No |
| Triggers writing a queue table | Yes | Partially | Schema only |

The choice usually collapses to one question: can you change the application? If yes, an outbox gives atomicity and meaning together. If not — a legacy system, a vendor product, a database you do not own — CDC is the only mechanism that gives atomicity without touching the writer.

> **Ask before choosing it**  
> Do consumers need to know what happened or only that a row changed? And who will notice when the connector stops — because a stopped connector is both a silently stale downstream and, through slot retention, a growing risk to the primary database.

**How it fails**

**How CDC goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Primary runs out of disk | Replication slot retaining log for a dead consumer | Alert on slot lag; drop abandoned slots |
| Rows missing or duplicated at startup | Snapshot and log handover position mismatched | Use a connector that does the handover atomically |
| Events applied out of order | Downstream partitioned by something other than the key | Partition by primary key |
| Consumer breaks after a migration | Schema change reached it as an unexpected structure | Treat schema changes as events; version consumers |
| Duplicate side effects downstream | At-least-once delivery with non-idempotent apply | Idempotent consumers keyed by primary key and position |
| Downstream cannot tell why a row changed | Row diffs carry no intent | Outbox with domain events, or derive intent carefully |
| Deletes never propagate | Soft deletes look like ordinary updates | Agree the convention explicitly with consumers |

> **The snapshot-to-log handover is where silent data loss lives**  
> A new consumer needs the full current table and then every change since. If the snapshot is taken at one moment and streaming begins at a later log position, everything in between is lost — permanently, and with no error. If streaming begins earlier, rows are replayed, which idempotent consumers survive. Getting this boundary exactly right is the main reason to use a mature connector rather than writing one.

**Where it is used**

- **Search index maintenance**, keeping an index in step with a relational source without the application publishing anything.
- **Cache invalidation from the database**, so a write through any path — including manual ones — invalidates correctly.
- **Data warehouse and lakehouse ingestion**, replacing nightly full reloads with a continuous change stream.
- **Migrating off a legacy system** by streaming its changes into a new one while both run.
- **Microservice data replication**, giving one service a read-only local copy of another's data.

**In the interview**

The expected framing is that CDC is the answer to dual writes, and the expected depth is knowing what it cannot give you.

- **Lead with the dual-write problem.** Two systems cannot be committed atomically from application code, which is the reason CDC exists rather than a performance argument.
- **Say it gives rows, not intent.** Volunteering that limitation and naming the outbox as the alternative is the distinguishing move.
- **Raise replication slot retention** — an abandoned slot filling the primary's disk is the operational failure that actually happens.
- **Mention the snapshot handover** as the place new consumers silently lose data.
- **State that delivery is at-least-once**, so consumers must be idempotent regardless of how clean the stream looks.

**Practice drill**

A service writes orders to Postgres and then publishes events so a search index and a cache can follow; both drift out of step roughly weekly. Explain precisely why retries cannot fix this. Then design the CDC replacement: describe the snapshot-to-log handover, choose the downstream partitioning key and justify it, and specify the two alerts you would add before turning it on.

**Go deeper**

Change data capture derives a stream of committed changes from the database's own replication log, which removes the need for the application to publish anything.

**It solves an impossibility, not an inefficiency.** A dual write asks two independent systems to commit together, which cannot be made atomic from application code — every implementation has a window where exactly one succeeded, leaving downstream either permanently behind or holding an event for a change that was rolled back. Because the replication log is written as part of making the transaction durable, an event derived from it exists if and only if the change committed. That is a guarantee retries cannot manufacture.

**Ordering arrives for free and must be preserved deliberately.** The log is in commit order, so the stream is correctly sequenced when it leaves the connector. Downstream parallelism can destroy that: two events for the same row processed by different workers can be applied in the wrong order. Partitioning by primary key is what keeps per-row ordering intact while still allowing throughput, and it is the single most important downstream configuration choice.

**Rows are not intent, and no amount of processing recovers it.** The log records that a status column changed from one value to another; it does not record whether that was a cancellation, an expiry or a correction. Consumers needing business meaning must infer it from column diffs, which is fragile and breaks under ordinary refactoring. Where meaning matters, a transactional outbox provides the same atomicity while carrying a domain event — the trade being that the application must participate, which is exactly what CDC is chosen to avoid.

**The snapshot handover is the subtle correctness problem.** A new consumer needs the table as it currently stands plus every change since, and the boundary between those two has to be exact. Beginning the stream after the snapshot position loses everything in the gap, silently and permanently; beginning before it replays rows, which idempotent consumers absorb. This asymmetry is why mature connectors take the snapshot and record the log position as one operation, and why writing your own is rarely worth it.

**Replication slots make a stopped consumer a database problem.** While a slot exists the primary must retain every log segment it has not acknowledged, so a connector that crashes, is paused for a deployment, or is simply forgotten causes unbounded log growth on the primary. A full disk on the write path is dramatically worse than the stale index the connector was maintaining, which makes slot lag one of the few downstream metrics that needs an alert on the database side.

**It is the right tool specifically when you cannot change the writer.** Legacy systems, vendor applications, databases owned by another team — these are where CDC is not merely convenient but the only mechanism that delivers atomicity without modifying the application. Where the application is yours and can be changed, the outbox is usually the better engineering choice, because intent expressed once at the source is more durable than intent reconstructed forever at every consumer.

**Related patterns:** Transactional Outbox · Materialized View · CQRS · Partitioned Log

---

### Materialized View

*Precompute and store the answer to an expensive query, so reads become a lookup and the cost moves to write time.*

> **When you hear…** The same expensive aggregate is computed repeatedly · dashboard queries take seconds · joins dominate read latency

**Flow:** `Base tables` → `View definition` → `Precomputed result` → `Refresh on change` → `Fast read`

**The problem**

A seller dashboard shows revenue by product by week. The query joins orders to items to products, filters by seller, groups and sums — eight seconds over a large table, and it runs every time anyone opens the page. The underlying numbers change a few hundred times an hour, but the query runs thousands of times an hour.

Indexes help the filter and do nothing for the aggregation. The work is real: rows genuinely have to be scanned and summed, and doing it on demand means doing it far more often than the data actually changes.

> **Compute it when it changes, not when it is asked for**  
> If a result is read far more often than its inputs change, computing it once per change and storing it converts thousands of expensive reads into thousands of cheap lookups. The cost does not disappear — it moves to write time, where there is much less of it.

**Mental model**

A view definition describes a derived result. Unlike an ordinary view it is stored rather than computed on access, so it needs a refresh strategy — which is where all the design decisions live.

1. **Define** — The query whose result is worth keeping.
2. **Materialise** — Run it once and store the output as a table.
3. **Index** — Index the stored result for how it is actually read.
4. **Refresh** — Recompute on a schedule, on change, or incrementally.
5. **Serve** — Reads hit the stored table, never the underlying joins.

> **A materialised view is stale by definition, and the refresh strategy sets by how much**  
> Between a change to the base data and the next refresh, the view is wrong. That is acceptable for a revenue dashboard and unacceptable for an available-inventory check at checkout. The staleness window is a product decision disguised as a refresh interval, and it should be chosen from what the number is used for.

**How it works**

**Three refresh strategies**

```text
1  FULL REFRESH ON A SCHEDULE
   recompute the whole view periodically
   + simplest; nothing to reason about
   - cost is proportional to the whole dataset
   - staleness up to one interval
   -> fine for nightly reporting, poor for anything
      read continuously

2  INCREMENTAL REFRESH
   apply only the rows that changed
   + cost proportional to change volume, not size
   - the view definition must be expressible as a
     delta: sums and counts are, medians and distinct
     counts are much harder
   -> the difference between a view that refreshes in
      50 ms and one that takes 8 minutes

3  EVENT-DRIVEN PROJECTION
   a consumer updates the view as changes arrive
   + near-real-time; no polling
   - you own correctness, idempotence and replay
   -> this is CQRS read-model territory

WHICH AGGREGATES ARE INCREMENTAL
  sum, count, min on insert      easy
  max/min on delete              needs a rescan
  average                        keep sum and count
  distinct count                 needs a sketch or
                                 a full recompute
  -> check this before promising incremental refresh
```

1. **Choose the refresh strategy from the read pattern** — A nightly report and a live dashboard need different answers, and the interval is the staleness guarantee.
2. **Verify the aggregate is incrementally maintainable** — Sums and counts compose; distinct counts and medians generally do not.
3. **Index the view for its queries** — A stored result that is still scanned has moved the cost without removing it.
4. **Refresh without blocking readers** — A refresh that locks the view makes reads fail during exactly the period it was meant to accelerate.
5. **Publish the view's age** — Consumers need to know how old the number is, especially on a dashboard someone acts on.
6. **Keep the base query runnable** — It is how you verify the view and rebuild it when a bug is found.

**Where the cost goes**

```text
DASHBOARD QUERY, 8 s, 5,000 reads/hour
base data changes 300 times/hour

ON DEMAND
  5,000 x 8 s = 11 compute-hours per hour
  -> impossible; the database is the product now

FULL REFRESH EVERY 5 MINUTES
  12 refreshes/hour x 8 s = 96 s of compute
  reads: single-table lookup, ~5 ms
  staleness: up to 5 minutes
  -> 400x less compute, and reads are 1,600x faster

INCREMENTAL
  300 deltas/hour, each a few ms
  staleness: seconds
  -> cheaper still, but only because sum and count
     are incrementally maintainable

THE SHAPE OF THE WIN
  cost now scales with CHANGE volume, not READ volume
  -> the more read-heavy the workload, the better it
     looks, which is the same economics as caching
```

| Metric | Value | Note |
|---|---|---|
| On demand | 11 compute-h/h | impossible |
| 5-min refresh | 96 s/h | **400× less** |
| Read latency | 8 s → 5 ms | 1,600× faster |
| Scales with | changes | not reads |

> **A refresh that locks the view is an outage on a schedule**  
> If refreshing takes an exclusive lock, every reader blocks for its duration — so a view introduced to make a dashboard fast makes it unavailable for eight seconds every five minutes. Concurrent refresh, or building into a new table and swapping it in, keeps readers served throughout and is the difference between a useful optimisation and a recurring incident.

**Technologies**

| Option | Refresh | Note |
|---|---|---|
| Postgres materialized views | Full, or concurrent full | No built-in incremental maintenance |
| Summary tables maintained by triggers | Incremental | You own correctness; triggers add write cost |
| Stream processing into a store | Continuous | Near-real-time; you own replay and idempotence |
| Data warehouse materialisations | Scheduled, often incremental | Built for exactly this shape |
| Rollup tables written by the application | On write | Simple and explicit; couples writers to the view |
| Plain index plus query | None | Try this first — often enough |

The last row matters more than it looks. A great many materialised views exist because a query was slow before anyone checked whether the right composite index would have made it fast, and an index costs far less to maintain and never goes stale.

**Trade-offs**

**Materialised view against the alternatives**

| Approach | Read cost | Freshness | Maintenance |
|---|---|---|---|
| Query on demand | Full cost every read | Perfect | None |
| Index tuning | Reduced | Perfect | Index upkeep on write |
| Materialised view, scheduled | Lookup | Up to one interval | Refresh cost |
| Materialised view, incremental | Lookup | Near-real-time | Delta logic must exist |
| Event-driven projection | Lookup | Near-real-time | You own the whole pipeline |

Reading the table top to bottom is also the order to try things in. Each row buys read speed and charges in freshness or maintenance, and the first row that makes the query acceptable is the right place to stop.

> **Ask before choosing it**  
> Has an appropriate index been tried, and did it fail for a reason you can state? Then: how stale may this number be before someone makes a wrong decision from it, and is the aggregate one that can be maintained incrementally at all?

**How it fails**

**How materialised views go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Readers blocked during refresh | Refresh takes an exclusive lock | Concurrent refresh, or build and swap |
| Numbers quietly wrong | Refresh failed and nothing alerted | Alert on refresh failure and on view age |
| Refresh takes longer than the interval | Full refresh over a growing dataset | Move to incremental, or lengthen the interval |
| Incremental view drifts from reality | Delta logic wrong for the aggregate | Periodic full rebuild and comparison |
| Decisions made on stale figures | View age not visible to the reader | Show the as-of time next to the number |
| View is still slow | Stored result not indexed for its queries | Index the view as you would a table |
| Complexity that an index would have removed | Adopted before tuning the query | Try the index first |

> **A silently failed refresh is worse than a slow query**  
> If a refresh job dies, the view keeps serving instantly — with numbers frozen at whatever moment it last succeeded. Nothing errors, the dashboard looks healthy, and people make decisions from figures that stopped moving days ago. Alerting on refresh failure and displaying the view's as-of time are what prevent a performance optimisation from becoming a correctness incident.

**Where it is used**

- **Reporting dashboards and analytics rollups**, the canonical use and the one where staleness is obviously acceptable.
- **Leaderboards and rankings**, precomputed because the ordering is expensive and read constantly.
- **Denormalised read models in CQRS**, which are materialised views maintained by events rather than by the database.
- **Search index documents**, a materialisation of a join across several tables into one flattened record.
- **Warehouse summary tables**, where incremental materialisation is a first-class feature of the platform.

**In the interview**

The strongest answer treats it as a cost-shifting decision and is explicit about what is being traded away.

- **Say the cost moves to write time** and that the pattern pays precisely in proportion to how read-heavy the workload is — the same economics as caching.
- **Rule out an index first.** Reaching for materialisation before tuning the query is a common and expensive habit.
- **Name the aggregates that cannot be incrementally maintained** — distinct counts and medians — because that determines which refresh strategy is even available.
- **Raise the locking refresh** as the failure that turns the optimisation into scheduled downtime.
- **Volunteer that a failed refresh is silent** and that the view's age should be visible wherever the number is used.

**Practice drill**

A dashboard query takes 8 seconds, runs 5,000 times an hour, and its base data changes 300 times an hour. Compute the compute cost on demand and under a five-minute full refresh, and state the staleness each gives. Then decide whether the aggregate can be maintained incrementally, say what you would do differently if it included a distinct count, and list the two alerts and one piece of UI you would add before shipping it.

**Go deeper**

A materialised view stores the result of an expensive query so that reading it is a lookup, shifting the computation from read time to write time.

**The economics are the economics of caching.** Work is done once per change rather than once per read, so the benefit scales with the read-to-write ratio. A result read thousands of times an hour and changed hundreds of times an hour is an obvious candidate; one read and written at similar rates is not, because the same computation happens either way and the added machinery buys nothing but staleness.

**The refresh strategy is the whole design.** A full refresh is simple and costs in proportion to the dataset, which means it stops being viable as the data grows rather than as the workload changes. Incremental refresh costs in proportion to the change volume and is dramatically cheaper — but only for aggregates that can be expressed as deltas. Sums and counts compose naturally; maxima break on deletion, averages need sum and count kept separately, and distinct counts generally require either a sketch or a full recomputation. Checking which case applies before promising near-real-time freshness avoids a commitment that cannot be met.

**Locking during refresh converts an optimisation into scheduled downtime.** A refresh that takes an exclusive lock blocks every reader for its duration, so a view introduced to make a page fast makes it unavailable periodically instead. Building the new result into a separate table and swapping it in, or using a concurrent refresh where the database supports it, keeps readers served throughout — and this detail is frequently discovered only after the first production refresh.

**A failed refresh is invisible in exactly the wrong way.** Unlike a slow query, which announces itself, a dead refresh job leaves the view serving instantly with numbers frozen at its last success. Dashboards stay responsive, no errors appear, and people continue making decisions from figures that stopped moving. Alerting on refresh failure, and rendering the view's as-of timestamp next to the figures it produced, are what stop a performance optimisation from turning into a correctness incident nobody noticed.

**It is frequently reached for too early.** A slow aggregate query often has a straightforward explanation — a missing composite index, a filter that cannot use the index it has, a join order the planner got wrong — and fixing that costs far less than maintaining a derived table and never introduces staleness. The disciplined sequence is to understand why the query is slow, try the index, and only then decide that the work is irreducible and should be done in advance.

**At the far end it stops being a database feature and becomes an architecture.** A view maintained continuously by a stream consumer rather than by a scheduled job is a CQRS read model: near-real-time, arbitrarily shaped, and entirely your responsibility for correctness, idempotence and rebuild. The progression from index to scheduled view to incremental view to event-driven projection is a ladder of increasing power and increasing ownership, and knowing which rung the problem actually requires is more valuable than knowing how to build the top one.

**Related patterns:** CQRS · Change Data Capture · Cache Aside · Batch Processing

---

### Consistent Snapshot

*Read a set of data as it existed at one instant, so a backup, export or migration never mixes rows from before and after a concurrent write.*

> **When you hear…** A backup restored to an impossible state · an export where totals do not reconcile · a migration that copied a half-finished transaction

**Flow:** `Snapshot point` → `Version pinned` → `Concurrent writes continue` → `Consistent read` → `Released`

**The problem**

A nightly export reads the orders table and then the payments table. Between the two reads a customer pays, so the export contains a payment for an order whose status still says awaiting payment. Nothing failed, no error was raised, and the downstream reconciliation reports a discrepancy that does not exist in the database.

The same tear appears in backups taken table by table, in migrations that copy one entity at a time, and in any report that issues several queries and assumes they saw the same world.

> **Pin the version, then read at leisure**  
> A store that keeps multiple versions of each row can serve a reader the versions that were current at one chosen instant, while writers carry on creating newer ones. The reader then sees a coherent world regardless of how long its scan takes, without blocking anyone.

**Mental model**

A snapshot is a timestamp, not a copy. The reader is told which version of the world to look at, and every row it reads is resolved as of that moment.

1. **Choose an instant** — Usually the transaction's start, recorded as a version or sequence number.
2. **Pin it** — The store agrees not to discard the versions that instant depends on.
3. **Read** — Every lookup resolves to the newest version at or before the pinned instant.
4. **Write freely** — Concurrent writers create new versions; the reader never sees them.
5. **Release** — The reader finishes and the old versions become reclaimable.

> **A long-held snapshot stops the database reclaiming space**  
> While a snapshot is pinned, every version it might need must be retained. A reader that sits open for hours therefore prevents cleanup of rows that have been updated thousands of times since, and the table bloats. The snapshot costs nothing to take and can cost a great deal to hold.

**How it works**

**How a snapshot resolves a read**

```text
EACH ROW VERSION CARRIES
  value, created_by_txn, deleted_by_txn

SNAPSHOT = a transaction id plus the set of
           transactions still in flight at that moment

A VERSION IS VISIBLE IF
  created_by  committed before the snapshot, AND
  deleted_by  is empty, or committed after it
-> so the reader sees exactly the committed state at
   one instant, with no half-applied transactions

TAKING ONE
  BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ
    select from orders   ...
    select from payments ...     <- same instant
  COMMIT

-> both queries resolve against the same snapshot, so
   a payment committed between them is invisible to
   both, and the export reconciles

WHY NOT JUST LOCK
  locking the tables gives consistency by stopping
  writes
  -> correct, and unavailable: an export that blocks
     writes for twenty minutes is an outage
  -> snapshots give the same read consistency with no
     impact on writers at all
```

1. **Wrap multi-query reads in one transaction** — Two queries outside a transaction see two different instants, which is the whole bug.
2. **Use repeatable read or stronger** — Read committed resolves each statement separately, so it does not give a stable snapshot.
3. **Keep snapshots short** — Retention of old versions is the cost, and it grows with both duration and write rate.
4. **Export from a replica where possible** — Bloat and retention then affect a follower rather than the primary.
5. **Record the snapshot point** — Anything resuming or reconciling later needs to know which instant it saw.
6. **Do not assume a snapshot means a lock** — Writers continue; the reader simply cannot see them.

**What a torn read looks like, and what it costs to avoid**

```text
TORN EXPORT
  t0  read orders    -> order 42 status = awaiting
  t1  customer pays; both tables updated
  t2  read payments  -> payment for order 42 present
  result: a payment against an unpaid order
  -> reconciliation fails on data that was never
     inconsistent

SNAPSHOT EXPORT
  t0  snapshot taken
  t0  read orders    -> as of t0
  t1  customer pays  -> invisible to this reader
  t0  read payments  -> as of t0
  result: coherent; the payment simply appears in
  tomorrow's export

COST OF HOLDING IT
  write rate 2,000 updates/s
  snapshot held 30 minutes
  -> ~3.6 million superseded versions must be retained
  -> table and index bloat, slower scans, eventual
     aggressive cleanup
  SHORTER IS CHEAPER, and a replica is cheaper still
```

| Metric | Value | Note |
|---|---|---|
| Snapshot cost | free to take | costly to hold |
| Writers | unblocked | **no locking** |
| 30-min hold | ~3.6M versions | at 2,000 w/s |
| Best practice | read from a replica | isolates bloat |

> **Read committed is not a snapshot**  
> Under the common default isolation level each statement sees the state at the moment that statement began, so two queries in the same transaction can still observe different worlds. Multi-query consistency requires repeatable read or stronger, and assuming the default provides it is one of the quieter sources of inconsistent exports.

**Technologies**

| Mechanism | Where | Note |
|---|---|---|
| MVCC snapshot isolation | Postgres, Oracle, MySQL InnoDB | Repeatable read or snapshot isolation |
| Filesystem or volume snapshot | Storage layer | Consistent on disk; the database may need a flush first |
| Backup tools with a snapshot start | pg_dump and equivalents | Take a snapshot then stream from it |
| Log position plus base copy | CDC initial load | The snapshot-to-log handover problem |
| Table locks | Any store | Correct and hostile to writers |
| Read replica at a pinned point | Replicated setups | Isolates the retention cost from the primary |

The CDC row is worth connecting: an initial load is exactly this problem, which is why a connector takes a consistent snapshot and records the log position it corresponds to. Every consistency argument in change capture rests on that handover being atomic.

**Trade-offs**

**Ways to read a coherent set of data**

| Approach | Blocks writers | Consistency | Cost |
|---|---|---|---|
| Separate queries, no transaction | No | None — torn reads | None |
| Snapshot isolation | No | One instant | Version retention |
| Table locks | Yes | One instant | Writers blocked |
| Copy to a replica and read there | No | One instant, on the replica | Replication lag plus a replica |
| Accept inconsistency and reconcile later | No | None | Reconciliation complexity |

The second and fourth rows are the practical choices. Snapshot isolation on the primary is simplest; reading from a replica gives the same guarantee while moving the retention cost off the write path, which is why large exports almost always run against a follower.

> **Ask before choosing it**  
> How long will the read take, and what is the write rate underneath it? Those two numbers multiply into the retention cost. If the answer is hours against a busy table, the question is not how to take the snapshot but where to take it — and the answer is a replica.

**How it fails**

**How snapshots go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Export internally inconsistent | Queries issued outside one transaction | Wrap the whole read in a single transaction |
| Still inconsistent inside a transaction | Read committed isolation | Repeatable read or snapshot isolation |
| Tables bloat and scans slow | Long-held snapshot blocking cleanup | Shorten the read; run it on a replica |
| Snapshot too old to serve | Required versions already reclaimed | Bound how long a reader may run |
| Backup restores to an impossible state | Tables dumped independently | Snapshot-based backup tooling |
| Migration copies a half-applied transaction | Row-by-row copy with no pinned instant | Snapshot the source, record the position |
| Replica export missing recent data | Replication lag not accounted for | Record the lag; state the as-of time |

> **A long snapshot on a busy primary degrades everything, not just itself**  
> Retention of superseded versions is a whole-table effect: every reader scanning that table now walks more versions, indexes grow, and cleanup falls further behind. A single well-meaning export left open for hours can therefore slow unrelated queries across the system, which makes it a failure that is diagnosed far from its cause.

**Where it is used**

- **Database backups**, which take a snapshot first so the dump represents one coherent instant.
- **CDC initial loads**, where the snapshot and the log position must correspond exactly.
- **Nightly exports and reconciliations**, the case where torn reads produce discrepancies that do not exist.
- **Online migrations**, copying a table while writes continue and then catching up from a recorded position.
- **Analytical queries over live data**, which need a stable world for the duration of a long scan.

**In the interview**

The signal is whether you distinguish consistency from locking, and whether you know what holding a snapshot costs.

- **Describe the torn read concretely** — a payment against an order that still says unpaid — because the abstract version does not convey why it matters.
- **Say that snapshots do not block writers.** Contrasting with table locks shows you know why this mechanism exists rather than the obvious one.
- **Name the retention cost** and connect duration and write rate; it is the detail that separates theory from operating one.
- **Point out that read committed is not enough**, which is the mistake that produces inconsistent exports from code that looks correct.
- **Suggest the replica** for anything long-running — it is the practical answer and it shows you have thought about blast radius.

**Practice drill**

A nightly export reads orders and then payments as two separate queries against a primary taking 2,000 writes per second, and reconciliation fails intermittently. Explain the mechanism producing the discrepancy. Then fix it three ways — one transaction at the right isolation level, a replica, and a snapshot-based backup tool — and for each state the consistency guarantee and where the retention cost lands. Finally, estimate the versions retained if the export takes 30 minutes.

**Go deeper**

A consistent snapshot lets a reader see the committed state of a dataset at a single instant while writers continue unimpeded.

**The failure it prevents is silent and looks like corruption.** Reading two related tables at two different moments can produce a combination that never existed — a payment recorded against an order still marked unpaid — and nothing errors. Downstream, that surfaces as a reconciliation discrepancy, and the investigation naturally starts by looking for a bug in the data rather than in the reading of it. Recognising torn reads as the cause is often the hardest part.

**Consistency here does not require blocking.** The obvious way to read a coherent set is to stop writes, and it works and is unacceptable: an export that locks tables for twenty minutes is an outage. Multi-version storage resolves each read against a pinned instant, so the reader sees a stable world while writers carry on creating versions it simply cannot see. That decoupling is the entire value, and it is why snapshot isolation rather than locking underpins every serious backup and migration tool.

**The default isolation level does not provide it.** Under read committed, each statement resolves independently, so two queries within one transaction can still observe different states. Code that wraps its queries in a transaction therefore looks correct and remains vulnerable, which makes this a particularly durable bug: the structure suggests the guarantee, and only the isolation level delivers it.

**Taking a snapshot is free; holding one is not.** Every version a pinned snapshot might need must be retained, so the cost is the product of how long the reader runs and how fast the underlying data changes. At a few thousand writes a second, a reader open for half an hour obliges the store to keep millions of superseded versions, and the consequence is not confined to that reader — tables and indexes bloat, other scans walk more versions, and cleanup falls behind. A single long export can therefore slow queries that have nothing to do with it.

**Which is why long reads belong on a replica.** Running the export against a follower gives the same instant-consistent view while placing the retention burden somewhere that is not serving writes. The trade is replication lag: the snapshot is coherent but slightly behind, which is almost always acceptable for exports and reporting provided the as-of time is recorded alongside the output.

**It is the foundation under several other patterns.** A change-capture connector's initial load is exactly this problem, and its correctness depends on the snapshot and the log position corresponding to the same instant. An online migration is the same shape again: copy at a pinned point, then replay everything since. Recognising consistent snapshotting as the shared primitive makes those handovers easier to reason about, because the question in each case reduces to whether the base copy and the resume point describe the same moment.

**Related patterns:** Change Data Capture · MVCC · Event Sourcing · Schema Evolution

---

### Schema Evolution

*Change the shape of data over time while old and new readers and writers coexist, by keeping every step compatible in both directions.*

> **When you hear…** A field must be added, renamed or removed · producers and consumers deploy independently · stored data outlives the code that wrote it

**Flow:** `Current schema` → `Additive change` → `Dual read and write` → `Backfill` → `Old shape retired`

**The problem**

A field is renamed in a message format. The producer ships on Tuesday, the three consumer teams ship over the following fortnight, and for two weeks messages are written in a shape half the readers cannot parse. Some fail loudly, one silently reads a missing field as zero, and the error is discovered in a finance report a month later.

The same shape appears in databases, in stored events, and in any API with clients you do not deploy. Wherever data outlives the code that produced it, a change to its shape has to work for both the old and the new interpreter at once.

> **There is no moment when everything changes at once**  
> Deployments roll, clients cache, messages sit in queues, and stored rows persist for years. Every schema change therefore has a window in which both shapes exist, so the only safe changes are ones where each version can read what the other writes. Designing for that window is the whole discipline.

**Mental model**

Two compatibility directions, and a change is only safe if it holds in whichever ones apply.

1. **Backward compatible** — New code can read old data. Required whenever stored data outlives a deploy.
2. **Forward compatible** — Old code can read new data. Required whenever writers upgrade before readers.
3. **Full compatibility** — Both. Required when producers and consumers deploy independently.
4. **Expand** — Add the new shape; write both. Nothing breaks because nothing was removed.
5. **Contract** — Remove the old shape, but only once nothing reads or writes it.

> **A rename is a delete plus an add, and deletes are the dangerous half**  
> Renaming a field in one step removes something readers depend on at the same moment it introduces something they do not know about. Split into add, dual-write, migrate readers, then remove, every intermediate state is safe — and the sequence takes three deployments precisely because there is no instant at which all participants change together.

**How it works**

**Which changes are safe, and in which direction**

```text
SAFE BOTH WAYS
  add an optional field with a default
  add a new message type or endpoint version
  widen a type where the old range still fits
  add a value to an enum consumers treat as open

BACKWARD ONLY  (new code reads old data)
  remove a field no current writer sets
  tighten a constraint already satisfied by all rows

FORWARD ONLY  (old code reads new data)
  add a field old readers ignore

UNSAFE IN ONE STEP
  rename a field            = remove + add
  change a type narrowly    = old values may not fit
  make an optional field required
  remove an enum value consumers switch on
  change the meaning of an existing field  <- worst,
    because nothing detects it: both sides parse
    successfully and disagree about what it means

THE SEQUENCE FOR AN UNSAFE CHANGE
  1  add the new shape alongside the old
  2  write both
  3  migrate readers to the new shape
  4  stop writing the old shape
  5  remove it, once nothing reads it
```

1. **Make every step additive or removal-only** — A step that adds and removes together cannot be compatible in both directions.
2. **Never repurpose an existing field** — Both sides parse it successfully and interpret it differently, so nothing detects the error.
3. **Treat defaults as part of the contract** — A missing field read as zero is indistinguishable from a real zero, which is how silent corruption starts.
4. **Version the payload, not just the endpoint** — Stored records are read by code written years later; the record must say what it is.
5. **Check compatibility automatically** — A registry that rejects an incompatible change catches it before deployment rather than in production.
6. **Retire the old shape deliberately** — Contraction is the step that gets forgotten, leaving both shapes maintained forever.

**The rename, done safely**

```text
GOAL  customer_name -> full_name

RELEASE 1   EXPAND
  add full_name, nullable
  write BOTH fields
  read customer_name
  -> old readers unaffected; new readers see both

BACKFILL     (no deploy)
  copy customer_name into full_name for existing rows
  in batches, monitored

RELEASE 2   SWITCH READS
  read full_name
  still write both
  -> rollback to release 1 still works, because
     customer_name is still being written

RELEASE 3   STOP DUAL WRITE
  write full_name only
  -> only once release 2 is everywhere and stable

LATER       CONTRACT
  drop customer_name
  -> irreversible; do it when no deployable version
     and no consumer reads it

WHY DUAL WRITE CONTINUES THROUGH RELEASE 2
  the previous release reads the old field, so it must
  keep receiving values or a rollback reads stale data
```

| Metric | Value | Note |
|---|---|---|
| Rename | 3 releases + drop | **no safe one-step** |
| Each step | additive or removal | never both |
| Rollback | preserved | until contraction |
| Forgotten step | contraction | both shapes forever |

> **Repurposing a field is the change no system can detect**  
> If a field that meant one thing starts meaning another, every parser still succeeds — the types match, nothing is missing, no validation fires. Old and new code simply disagree about what the value represents, and the disagreement surfaces as wrong numbers rather than as errors. Adding a new field and deprecating the old one is always cheaper than the investigation that follows.

**Technologies**

| Format or store | Compatibility mechanism | Note |
|---|---|---|
| Avro with a schema registry | Reader and writer schemas, enforced rules | Compatibility checked before publishing |
| Protocol Buffers | Field numbers; unknown fields preserved | Never reuse a retired field number |
| JSON | Convention only | Nothing enforces anything; discipline required |
| Relational databases | Expand-contract migrations | DDL is the change; rollback is not automatic |
| Event stores | Versioned events, upcasting on read | The past cannot be migrated |
| HTTP APIs | Versioned media types or paths | Clients you do not control set the retirement pace |

Protocol Buffers deserve the specific warning about field numbers: reusing a number that was previously assigned to a different field means old data decodes into the new field with the old meaning, producing exactly the silent misinterpretation that repurposing causes. Retired numbers are reserved permanently for that reason.

**Trade-offs**

**Approaches to changing a shape**

| Approach | Downtime | Rollback | Effort |
|---|---|---|---|
| Change in one step | Errors during the window | Impossible | Minimal |
| Expand and contract | None | Preserved until contraction | Three releases |
| Version the whole payload | None | Preserved | Two shapes maintained |
| Coordinated deploy of everything | Planned outage | All-or-nothing | High coordination |
| Never remove anything | None | Preserved | Permanent accumulation |

The last row is where a lot of systems end up by accident rather than by choice: the expand phase happens, the benefit arrives, and the contraction is never scheduled. That is not free — it leaves two shapes to be written, tested and reasoned about indefinitely.

> **Ask before choosing it**  
> Which direction of compatibility does this change actually need? If writers and readers deploy together and nothing is stored, the constraint is far weaker than if events persist for years. And who are the readers you cannot deploy — because those set the pace at which anything can be retired.

**How it fails**

**How schema evolution goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Consumers break during a rollout | Rename shipped in one step | Expand, dual-write, migrate, contract |
| Wrong numbers with no errors | An existing field repurposed | Add a new field; never change a meaning |
| Missing data read as zero | Default indistinguishable from a real value | Use nullable types or an explicit presence flag |
| Rollback impossible | Old field stopped being written too early | Dual-write until the previous release is retired |
| Old records unreadable | Event schema changed with no version | Version from the start; upcast on read |
| Both shapes maintained forever | Contraction never scheduled | Track it as work at the time of expansion |
| Old data decodes into the wrong field | A retired field number reused | Reserve retired numbers permanently |

> **Defaults hide missing data**  
> Reading an absent field as its zero value means a record that never carried the field is indistinguishable from one that carried a genuine zero. Aggregations then silently include rows that should have been excluded, and the discrepancy is small enough to look like a rounding issue. Where absence is meaningful, it has to be representable — a nullable type or an explicit flag — rather than collapsed into a default.

**Where it is used**

- **Event-driven platforms with a schema registry**, which reject an incompatible change at publish time rather than in production.
- **Expand-contract database migrations**, the standard approach to changing a column without downtime.
- **Protocol Buffers in service meshes**, where unknown fields are preserved so intermediaries can forward what they do not understand.
- **Event stores**, where records outlive every version of the code and upcasting on read is permanent work.
- **Public APIs**, whose retirement timelines are set by clients the provider cannot deploy.

**In the interview**

The expected answer is a sequence, not a definition — and the sharpest detail is why a rename cannot be done in one step.

- **Name the two directions** and say which the situation requires; treating compatibility as one thing is the common imprecision.
- **Walk the rename sequence** — expand, dual-write, switch reads, stop writing, drop — and explain why dual-writing continues past the read switch.
- **Call out repurposing a field as undetectable.** It is the failure with no error, and volunteering it demonstrates real experience.
- **Raise defaults hiding absence**, particularly for numeric fields where a missing value and a real zero become the same thing.
- **Mention that contraction gets forgotten**, and that tracking it at expansion time is what stops two shapes being maintained forever.

**Practice drill**

A message field must be renamed, with one producer team and three consumer teams deploying independently over a fortnight. Write the release sequence, stating at each step what the old and new code can read and whether a rollback is possible. Then identify which step is irreversible and what must be verified before it. Finally, explain what goes wrong if instead of renaming, the team reuses the existing field for the new meaning.

**Go deeper**

Schema evolution is the practice of changing the shape of data while every participant that reads or writes it continues to work, because there is never a moment when all of them change together.

**Compatibility has two directions and they are required by different circumstances.** Backward compatibility — new code reading old data — is needed wherever stored data outlives a deployment, which covers databases and event stores absolutely. Forward compatibility — old code reading new data — is needed wherever writers can upgrade before readers, which covers messaging and any API with independent clients. Systems with both properties need full compatibility, and knowing which constraint actually applies prevents both over-engineering and the wrong kind of confidence.

**A rename is unsafe because it is two changes at once.** Removing a field readers depend on and introducing one they do not know about in the same step guarantees a window where someone is broken. Splitting it into expansion, dual writing, migrating readers and eventual contraction makes each intermediate state compatible with its neighbours, which is exactly why it takes three deployments — one per participant transition rather than one per logical change.

**Dual writing must outlast the read switch.** Once readers move to the new field, the previous release still reads the old one, so stopping the old write immediately means a rollback reads a field that has gone stale. Continuing to write both until the read-switch release is stable everywhere and no longer a rollback target is what preserves the ability to go back, which is the primary mitigation for any bad deployment.

**Repurposing a field is the one change nothing catches.** Every other unsafe change produces a parse failure, a missing value or a type error somewhere. Changing what an existing field means produces none of those: both sides decode successfully and interpret the value differently, so the system reports wrong numbers while appearing entirely healthy. This is why formats that assign stable field numbers reserve retired ones permanently — reusing a number reintroduces exactly this failure at the wire level.

**Defaults quietly destroy the distinction between absent and zero.** Reading a missing numeric field as zero means a record written before the field existed is indistinguishable from one carrying a real measurement of zero, and aggregations then include rows that should have been excluded. The error is proportionally small, looks like noise, and is very hard to trace back to a schema change made months earlier. Where absence carries meaning it must remain representable rather than being collapsed into a value.

**Contraction is the step that does not happen.** Expansion delivers the benefit, the system works, and removing the old shape is never urgent — so both shapes persist, both are written, both are tested, and every future change has to accommodate both. Treating the removal as tracked work created at the moment of expansion, with a condition attached rather than a date, is the only reliable way to keep a schema from accumulating every intermediate state it has ever passed through.

**Related patterns:** Event Sourcing · Change Data Capture · Consistent Snapshot · Partitioned Log

---

## Transactions

### Saga

*Break a business transaction into local transactions with compensating actions, so a multi-service operation can be undone without a distributed lock.*

> **When you hear…** One business operation spans several services · each owns its own database · a distributed transaction is not available or not acceptable

**Flow:** `Step 1 commits` → `Step 2 commits` → `Step 3 fails` → `Compensate 2` → `Compensate 1`

**The problem**

Placing an order reserves inventory, charges a card and creates a shipment, and each lives in a different service with its own database. If the charge succeeds and the shipment fails, the customer has paid for something that will never arrive; if inventory is reserved and the charge fails, stock is held for an order that does not exist.

The obvious answer — one transaction across all three — is not available. They are separate databases, and even where a coordinator could span them, holding locks across three services for the duration of a card authorisation couples their availability together in a way nobody wants.

> **Give up atomicity and buy back correctness with compensation**  
> If each step commits locally and every step has a defined way to be undone, a failure part-way through is handled by running the undos in reverse rather than by rolling back a transaction that never existed. The operation is never atomic — intermediate states are visible — but it always reaches a consistent end state.

**Mental model**

A sequence of local transactions, each with a compensating transaction that semantically undoes it. Forward until something fails, then backward from there.

1. **Local step** — One service, one database, one ordinary transaction that commits.
2. **Record progress** — The saga's position is durable, or a crash loses track of what must be undone.
3. **Fail** — A step cannot complete, and forward progress stops.
4. **Compensate** — Run the undo for every completed step, in reverse order.
5. **End state** — Either everything happened, or everything that happened has been undone.

> **Compensation is semantic, not a rollback**  
> You cannot un-charge a card; you issue a refund, which is a different transaction with its own record. You cannot un-send an email. Compensation restores business meaning rather than previous state, which means some steps leave permanent traces and some cannot be compensated at all — and those must be ordered last.

**How it works**

**Orchestration and choreography**

```text
ORCHESTRATED  (a coordinator drives)
  orchestrator:
    reserve inventory   -> ok
    charge card         -> ok
    create shipment     -> FAILS
    refund card                 <- compensate
    release inventory           <- compensate
  + the whole flow is in one place and readable
  + progress is easy to persist and resume
  - the orchestrator is a component to run

CHOREOGRAPHED  (services react to events)
  inventory reserved -> payment listens, charges
  payment succeeded  -> shipping listens, creates
  shipping failed    -> payment listens, refunds
                     -> inventory listens, releases
  + no central component
  - the flow exists nowhere; it is emergent
  - debugging means reconstructing it from logs

WHICH TO CHOOSE
  more than about three steps, or any branching
  -> orchestration, because an emergent flow stops
     being understandable
  two or three steps with no branching
  -> choreography is lighter and fine

ORDER THE STEPS DELIBERATELY
  put the hardest-to-compensate step LAST
  -> charge the card after inventory is secured, not
     before, so the common failure needs no refund
```

1. **Order steps so the least reversible is last** — Every step before it can then fail cheaply, and the expensive compensation runs least often.
2. **Persist saga state before each step** — A crash mid-saga must be recoverable, which means knowing what has already committed.
3. **Make every step and compensation idempotent** — Retries are certain, and a compensation applied twice must not refund twice.
4. **Define a compensation for every step, at design time** — A step discovered late to be uncompensatable forces a redesign of the ordering.
5. **Decide what happens when compensation fails** — A failed undo needs a retry policy and, eventually, a human.
6. **Expose the intermediate state honestly** — The operation is not atomic; the interface should say pending rather than implying completion.

**What the customer sees, and the failure that matters**

```text
ORDER SAGA
  1  reserve inventory        compensate: release
  2  charge card              compensate: refund
  3  create shipment          compensate: cancel
  4  send confirmation email  compensate: NONE

STEP 3 FAILS
  refund the card
  release the inventory
  order ends as failed
  -> customer sees a reservation, a charge, and a
     refund; the money moved twice for nothing, which
     is visible on their statement

WHY THE EMAIL IS LAST
  it cannot be compensated, so it must only happen
  once everything reversible has already succeeded

THE HARD FAILURE
  step 3 fails, AND the refund also fails
  -> the saga cannot complete forward or backward
  -> retry with backoff, then escalate
  -> this state must be visible and finite, not an
     item stuck silently in a table forever
```

| Metric | Value | Note |
|---|---|---|
| Atomicity | given up | intermediate states visible |
| End state | always consistent | **forward or fully undone** |
| Ordering rule | least reversible last | minimises refunds |
| Stuck saga | needs escalation | not silent |

> **Sagas are not isolated, and that is visible to users**  
> Between the first commit and the last, other readers see a partly applied operation: inventory reserved for an order that does not exist yet, a charge with no shipment. No amount of engineering hides this, because the intermediate states are genuinely committed. The interface has to represent them — pending, processing, reserved — rather than pretending the operation is instantaneous.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Orchestrator | Durable workflow engines | Persistence, retries and resume come built in |
| Choreography | Event broker with per-service handlers | No central component; no central view either |
| State persistence | A saga table in the orchestrator's database | Must be written before the step it describes |
| Idempotency | Idempotency keys on each step | Retries are certain, so this is not optional |
| Alternative | Two-phase commit | Atomic, but couples availability across participants |
| Alternative | Redesign to a single service | If the steps really belong together, that is the fix |

The last row is worth taking seriously. A saga is a response to a service boundary; if three steps must always succeed or fail together and are always invoked together, the boundary may simply be in the wrong place, and merging them removes the problem rather than managing it.

**Trade-offs**

**Saga against the alternatives**

| Approach | Atomic | Isolated | Couples availability |
|---|---|---|---|
| Single local transaction | Yes | Yes | n/a — one service |
| Two-phase commit | Yes | Yes | Yes, strongly |
| Orchestrated saga | No | No | No |
| Choreographed saga | No | No | No |
| Do nothing and reconcile later | No | No | No |

The saga rows trade both atomicity and isolation to avoid coupling availability, which is usually the right exchange across service boundaries — a participant being down should delay an operation rather than block every other participant's transactions.

> **Ask before choosing it**  
> Which step cannot be compensated, and can it be moved last? Then: what does the user see between the first and last step, and does the interface represent that honestly? A saga whose intermediate states are invisible to the product is a saga that will surprise someone.

**How it fails**

**How sagas go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Money taken with nothing delivered | Compensation failed and was not retried | Retry with backoff; escalate after a bound |
| Double refund | Compensation not idempotent | Idempotency keys on every compensating action |
| Saga stuck forever | Crash with no persisted progress | Write saga state before each step |
| Inventory held indefinitely | Compensation never ran after a lost message | Timeout the saga; compensate on expiry |
| Nobody can explain the flow | Choreography across many services | Orchestrate once it exceeds a few steps |
| User told an order succeeded, then it reversed | Interface implied atomicity | Represent pending states explicitly |
| An uncompensatable step ran too early | Ordering not chosen deliberately | Least reversible step last |

> **A saga that can neither complete nor compensate is the state that needs a person**  
> When the forward step fails and its undo also fails, the operation is stranded with real effects half-applied — money moved, stock held. Retrying indefinitely hides it; giving up silently loses it. The design needs a bounded retry, then a visible failed state with enough recorded context for someone to resolve it by hand, because at that point no automatic answer exists.

**Where it is used**

- **Order fulfilment across inventory, payment and shipping**, the canonical example and the one the pattern was named for.
- **Travel booking**, where a flight, a hotel and a car are reserved independently and any one can fail.
- **Account provisioning**, creating records across several systems with cleanup when a later step fails.
- **Durable workflow engines**, which are orchestrated sagas with persistence and retries as a platform feature.
- **Money movement between institutions**, where compensation is a reversing entry rather than a deletion.

**In the interview**

The distinguishing content is ordering and the stuck state, not the definition of compensation.

- **Say compensation is semantic, not a rollback** — a refund rather than an un-charge — and note that some steps cannot be compensated at all.
- **Give the ordering rule**: the least reversible step goes last, so the common failures cost nothing to undo.
- **Volunteer that sagas are not isolated** and that the interface must represent intermediate states rather than implying atomicity.
- **Describe the doubly-failed case** — forward fails and compensation fails — and say it needs bounded retries then a human, because that is the state most answers omit.
- **Choose orchestration past a few steps** and justify it by debuggability rather than preference.

**Practice drill**

An order saga reserves inventory, charges a card, creates a shipment and sends a confirmation email. Assign a compensating action to each step and identify which has none. Choose the ordering that minimises compensation cost and justify it. Then trace what happens when the shipment step fails and the refund also fails: say exactly what state the system is in, what it retries, when it stops, and what a person needs to see in order to resolve it.

**Go deeper**

A saga replaces a distributed transaction with a sequence of local transactions, each paired with an action that semantically undoes it.

**It trades atomicity and isolation for decoupled availability.** Each step commits independently, so intermediate states are genuinely visible to everyone else — stock reserved for an order that does not exist yet, a charge with no shipment behind it. In exchange, no participant holds locks on behalf of another and one service being slow or down delays an operation rather than blocking transactions elsewhere. Across service boundaries that exchange is almost always correct, but it is an exchange, and pretending otherwise leads to interfaces that imply an atomicity the system does not have.

**Compensation restores meaning, not state.** A charge is undone by a refund, which is a new transaction leaving its own permanent record; an email cannot be undone at all. This makes step ordering a design decision rather than an implementation detail: placing the least reversible action last means the failures that actually happen are cheap to unwind, while placing it first guarantees that every subsequent failure produces a visible, customer-facing reversal.

**Durable progress is what makes recovery possible.** A saga that crashes mid-flight must know which steps committed, because compensation depends on it — and that knowledge cannot live only in memory or in the sequence of messages. Writing saga state before each step, so the record is never behind reality, is the mechanism that turns an interrupted saga into a resumable one rather than a stranded one.

**Idempotency is mandatory on both directions.** Every step and every compensation will be retried, because the alternative to retrying is losing the operation. A compensation applied twice that refunds twice is a worse outcome than the failure it was handling, so each action needs a key that lets the receiver recognise a repeat. Systems that add retries without adding idempotency convert transient failures into financial ones.

**The stranded saga is the case most designs omit.** When a forward step fails and its compensation also fails, the operation can move neither way and real effects are half-applied. Infinite retry hides it; abandonment loses it. What is needed is a bounded retry policy followed by an explicit failed state carrying enough context for a person to resolve it, because at that point there is genuinely no correct automatic action — only a choice someone has to make.

**Choreography stops scaling before it stops working.** Two or three steps reacting to one another's events is light and perfectly clear. Beyond that, the flow exists in no single place: understanding it means reconstructing it from handlers scattered across services, and changing it means reasoning about emergent behaviour. Orchestration reintroduces a component to run and repays it immediately in readability, resumability and the ability to answer where a given saga currently is — which is the question operations will ask first.

**Related patterns:** Two Phase Commit · Idempotency Key · Durable Workflow · Transactional Outbox

---

### Two Phase Commit

*A coordinator asks every participant to prepare, and only when all agree does it tell them to commit — giving atomicity across systems at the cost of blocking.*

> **When you hear…** Several systems must commit atomically · partial application is unacceptable · participants support a prepare phase

**Flow:** `Coordinator` → `Prepare request` → `All vote yes` → `Commit request` → `All commit`

**The problem**

A transfer must debit one database and credit another, and there is no acceptable outcome in which one happens without the other. Committing them in sequence leaves a window where the money has left one account and not arrived at the other, and a crash inside that window loses it.

Compensation is not always an adequate answer. In a ledger, a reversing entry is visible and auditable; in some regulatory and integration contexts, an intermediate state that was briefly visible is itself the problem.

> **Separate deciding from doing**  
> If every participant can be asked to guarantee it is able to commit — durably reserving everything it needs — without actually committing, then the coordinator learns the outcome before any of it is visible. Once all have promised, telling them to proceed cannot fail for lack of resources, so the whole thing commits or none of it does.

**Mental model**

Two rounds. The first asks for a promise and gets a vote; the second converts the promises into commits. The decision point is the moment the coordinator records the outcome.

1. **Prepare** — Each participant does everything except commit, durably, and votes yes or no.
2. **Promise** — A yes means the participant guarantees it can commit later, whatever happens in between.
3. **Decide** — The coordinator records commit if every vote was yes, abort otherwise. This write is the point of no return.
4. **Commit or abort** — Participants are told the decision and apply it.
5. **Acknowledge** — The coordinator can forget the transaction once everyone has confirmed.

> **A prepared participant is holding locks and cannot decide for itself**  
> Between voting yes and hearing the outcome, a participant must retain everything needed to commit — locks included — and it may not unilaterally resolve. If the coordinator dies in that window, the participant blocks indefinitely, holding resources that other transactions need. This is the pattern's defining weakness and it is not fixable within the protocol.

**How it works**

**The protocol and where it blocks**

```text
PHASE 1  PREPARE
  coordinator -> all: prepare
  each participant:
    do the work, write it durably, hold locks
    reply YES (I promise I can commit) or NO
  -> a YES is binding: the participant may no longer
     abort on its own

DECISION
  all YES -> coordinator durably records COMMIT
  any NO  -> coordinator durably records ABORT
  -> this write is the point of no return; the
     outcome now exists independently of any
     participant

PHASE 2  COMMIT
  coordinator -> all: commit (or abort)
  participants apply and acknowledge
  -> retried until every participant acknowledges,
     because the decision is already final

THE BLOCKING WINDOW
  participant voted YES
  coordinator crashes before sending phase 2
  -> the participant cannot commit (no decision) and
     cannot abort (it promised)
  -> it blocks, holding locks, until the coordinator
     returns
  -> other transactions needing those rows queue
     behind it

WHAT MAKES RECOVERY POSSIBLE
  the coordinator's decision log must be durable
  before phase 2 begins; on restart it replays the
  decision to anyone still waiting
```

1. **Make the coordinator's decision durable before phase 2** — Recovery depends entirely on the outcome surviving the coordinator's crash.
2. **Keep the prepare window short** — Locks are held for its duration, so a slow participant blocks everyone touching the same rows.
3. **Give participants a heuristic timeout policy** — Blocking forever is sometimes worse than guessing, but a heuristic decision can break atomicity and must be logged loudly.
4. **Run the coordinator with failover** — A single-instance coordinator makes its availability the availability of every transaction.
5. **Do not span slow or external systems** — Holding locks across a third-party call converts their latency into your contention.
6. **Prefer a saga across service boundaries** — The coupling 2PC creates is usually the thing service boundaries exist to avoid.

**Availability arithmetic**

```text
THREE PARTICIPANTS, each 99.9% available
coordinator 99.9%

A TRANSACTION SUCCEEDS ONLY IF ALL FOUR ARE UP
  0.999^4 = 99.6%
  -> ~3 hours of unavailability per month, caused
     purely by composition

AND WORSE: a participant failure during the prepare
window blocks rows rather than just failing the
transaction
  -> the impact is not confined to that transaction

COMPARE A SAGA
  each step needs only its own service
  a participant being down delays that step
  other participants keep serving unrelated work
  -> availability is not multiplied

WHY 2PC STILL EXISTS
  inside one database across shards, where the
  participants are trusted, fast and co-located, the
  blocking window is milliseconds and the atomicity
  is genuinely free of the coupling that hurts across
  services
```

| Metric | Value | Note |
|---|---|---|
| Atomicity | genuine | all or nothing |
| Availability | multiplied | **99.6% from 99.9%** |
| Blocking | locks held | coordinator is critical |
| Best fit | inside one system | not across services |

> **Heuristic decisions silently break the guarantee**  
> Some implementations let a participant that has waited too long decide for itself. That converts an indefinite block into a possible violation of atomicity — the participant may commit while others abort. It is sometimes the lesser evil operationally, but it must be recorded prominently, because the system has stopped providing the property it was chosen for and nothing else will reveal that.

**Technologies**

| Context | Support | Note |
|---|---|---|
| XA transactions | Databases and message brokers | Standard, widely implemented, widely avoided |
| Distributed databases | Internal 2PC across shards | Where it genuinely works well |
| Transaction managers | Application servers, JTA | Coordinator with recovery built in |
| Consensus-backed coordinators | Raft-replicated decision log | Removes the single-coordinator failure |
| Saga | Application-level compensation | No atomicity; no coupling either |
| Outbox | Atomic within one database | Solves the common case without a coordinator |

The outbox row deserves emphasis because it removes the most frequent reason people reach for 2PC: committing a database change and a message together. Writing the message into the same database in the same transaction achieves that atomically with no coordinator and no blocking, which is why it has largely displaced XA for that use.

**Trade-offs**

**Two-phase commit against the alternatives**

| Approach | Atomic | Blocking | Availability |
|---|---|---|---|
| Two-phase commit | Yes | Yes, on coordinator failure | Product of all participants |
| Three-phase commit | Yes, in theory | Reduced | Still coupled; rarely used |
| Consensus-backed coordinator | Yes | Much reduced | Coordinator survives failures |
| Saga | No | No | Independent per participant |
| Outbox | Yes, within one database | No | Single system |

The practical landscape is narrower than the table suggests: inside a single distributed database 2PC is a reasonable internal mechanism, and across independently operated services it has largely lost to sagas and outboxes because the availability coupling is precisely what those boundaries exist to prevent.

> **Ask before choosing it**  
> Is the atomicity requirement real, or would a visible compensating action be acceptable? And are the participants inside one trust and latency boundary? If the answer to the second is no, the blocking window will be long enough that contention, not correctness, becomes the problem.

**How it fails**

**How two-phase commit goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Participants block indefinitely | Coordinator died after prepare | Durable decision log; coordinator failover |
| Atomicity violated | A participant made a heuristic decision | Log loudly; reconcile; prefer blocking where possible |
| Widespread contention | Long prepare window holding locks | Keep participants fast and co-located |
| Availability worse than any component | Every participant must be up | Reconsider; sagas do not multiply availability |
| Recovery impossible after restart | Decision not durable before phase 2 | Write the outcome before sending commit |
| Transaction manager is a single point of failure | One coordinator instance | Replicate the coordinator's log |
| Third-party latency becomes internal contention | External system inside the transaction | Keep external calls outside; use a saga |

> **The coordinator's decision log is the whole protocol**  
> Everything about recovery depends on the outcome being durable before any participant is told. If the coordinator crashes after deciding but before recording, it cannot tell prepared participants what to do on restart, and they block with no resolution available. That single write — before phase two begins — is what separates a recoverable protocol from a permanent stall.

**Where it is used**

- **Distributed databases committing across shards**, where participants are internal, fast and co-located.
- **XA transactions across a database and a message broker**, historically common and now largely replaced by the outbox.
- **Application server transaction managers**, which provide a coordinator with recovery for enterprise integrations.
- **Financial settlement between systems** where a visible intermediate state is unacceptable.
- **Consensus-backed coordinators** in modern distributed stores, where replicating the decision log removes the classic failure.

**In the interview**

The expected answer names the blocking window unprompted and explains why sagas displaced this pattern across services.

- **Describe the prepare promise precisely** — a yes is binding and the participant can no longer abort alone — because that is what creates the block.
- **Identify the exact failure**: coordinator dies after prepare, participants hold locks with no way to resolve.
- **Do the availability arithmetic.** Multiplying participant availability is the argument against it across services, and it is concrete.
- **Say the decision must be durable before phase two**, since recovery depends entirely on that write.
- **Contrast with the outbox** for the database-plus-message case; it is the specific scenario people used to reach for XA to solve.

**Practice drill**

A transfer must debit one database and credit another atomically. Design it with two-phase commit and state the availability of the composed operation given 99.9% per participant plus a coordinator. Identify the exact window in which a coordinator crash leaves participants blocked and what they hold during it. Then redesign the same transfer as a saga, say what a user could observe that they could not under 2PC, and argue which you would ship.

**Go deeper**

Two-phase commit obtains atomicity across independent systems by separating the decision to commit from the act of committing.

**The prepare vote is a binding promise, and that is what makes it work.** A participant voting yes has done everything except commit, durably, and has given up the right to abort on its own. Because every participant has made that guarantee before the coordinator decides, the commit round cannot fail for want of resources — the outcome is already assured. This is the mechanism that delivers genuine atomicity rather than eventual convergence.

**The same promise is what makes it block.** Between voting and learning the outcome, a participant must retain everything needed to commit, including locks, and may not resolve unilaterally. If the coordinator fails in that window the participant waits indefinitely, holding resources that unrelated transactions need. The contention therefore spreads beyond the stalled transaction to anything touching the same rows, which is why the failure is disproportionate to its apparent scope.

**Availability composes multiplicatively, and that is the argument against it across services.** Every participant and the coordinator must be up for a transaction to succeed, so four components at three nines give roughly 99.6% — hours of monthly unavailability produced purely by composition, before anything actually breaks. A saga needs only the participant executing the current step, so one service being down delays an operation instead of failing every transaction that involves it.

**Durability of the decision is the entire recovery story.** The coordinator must record the outcome before telling anyone, because a crash after deciding but before recording leaves it unable to inform prepared participants on restart — and they cannot work it out themselves. Everything else in the protocol is retryable; that one write is not, and it is the reason the coordinator's log is treated as the critical artefact rather than any participant's state.

**Heuristic decisions trade correctness for liveness, quietly.** Allowing a long-blocked participant to decide for itself converts an indefinite stall into a possible atomicity violation, since it may commit while others abort. Operationally this is sometimes preferable to holding locks forever, but it means the system has stopped providing the guarantee it was chosen for, and nothing except explicit and prominent logging will reveal that a heuristic was taken.

**It survives where the coupling is not a cost.** Inside a single distributed database, participants are internal, co-located and fast, the prepare window is milliseconds, and the coordinator can be replicated by consensus — so atomicity across shards is genuinely worth having. Across independently operated services the coupling it creates is exactly what the boundaries exist to prevent, which is why sagas took that ground, and why the outbox took the specific case of committing a database change alongside a message without needing a coordinator at all.

**Related patterns:** Saga · Transactional Outbox · Idempotency Key · Leader Election

---

## Messaging

### Transactional Outbox

*Write the message into the same database transaction as the state change, then publish it from that table, so the two can never disagree.*

> **When you hear…** A state change must always produce a message · dual writes are silently losing events · atomicity across database and broker is needed

**Flow:** `Business write` → `Outbox row` → `One transaction` → `Relay reads table` → `Publish to broker`

**The problem**

An order is saved to the database and then published to a broker. Between those two operations the process can crash, the broker can be unreachable, or the publish can time out after the broker has actually accepted it. The database now says the order exists and the rest of the system never hears about it, or hears about it twice.

Reversing the order does not help. Publishing first and saving second means a consumer can act on an order that was never committed. There is no ordering of two independent writes that makes them atomic.

> **Make the message part of the state change**  
> A database can commit two rows atomically without any difficulty. If the message to be sent is written as a row in the same transaction as the business data, then either both exist or neither does, and the only remaining problem — getting that row to the broker — is a delivery problem with retries rather than a consistency problem with no solution.

**Mental model**

A table of messages waiting to be sent, written transactionally with the data that caused them, and drained by a separate process that can retry as often as it needs to.

1. **Commit** — The business change and the outbox row commit together, or neither does.
2. **Relay** — A separate process reads unsent rows from the outbox.
3. **Publish** — It sends them to the broker, in order, retrying on failure.
4. **Mark** — It records the row as sent — after the broker acknowledges, never before.
5. **Clean** — Old rows are pruned so the table does not grow without bound.

> **The outbox gives at-least-once, not exactly-once**  
> The relay can publish successfully and then crash before marking the row sent, so the message goes out again on restart. This is unavoidable: marking and publishing are themselves two systems. The guarantee is that no message is lost, which means every consumer must be prepared to see duplicates.

**How it works**

**The write path and the relay**

```text
APPLICATION WRITE
  BEGIN
    INSERT INTO orders (...)
    INSERT INTO outbox (id, topic, payload, created_at)
  COMMIT
  -> atomic; no broker involved on the request path

RELAY  (polling)
  loop:
    rows = SELECT * FROM outbox
           WHERE sent_at IS NULL
           ORDER BY id
           LIMIT 100
           FOR UPDATE SKIP LOCKED
    for row in rows:
      publish(row)
      UPDATE outbox SET sent_at = now() WHERE id = row.id

RELAY  (change data capture)
  tail the database log for inserts into outbox
  -> no polling, lower latency, no query load
  -> needs CDC infrastructure

WHY SKIP LOCKED
  multiple relay instances can run safely; each takes
  rows the others are not holding, so there is no
  leader election and no single point of failure

ORDER MATTERS
  publish, THEN mark sent
  -> crash between them republishes (at-least-once)
  mark sent, THEN publish
  -> crash between them LOSES the message
```

1. **Insert the outbox row in the same transaction** — This is the entire point; a separate transaction reintroduces the dual write.
2. **Publish before marking sent** — The reverse order converts a duplicate into a loss, which is far worse.
3. **Use SKIP LOCKED so relays can run in parallel** — Otherwise the relay is a singleton and its failure stops all messaging.
4. **Preserve per-key ordering if consumers depend on it** — Parallel relays reorder; partition the drain by key when order matters.
5. **Prune sent rows on a schedule** — An unbounded outbox eventually degrades the database it shares.
6. **Monitor outbox lag and depth** — A stalled relay is invisible from the application side — the writes keep succeeding.

**What the outbox costs and what it replaces**

```text
WITHOUT OUTBOX  (dual write)
  save order            -> ok
  publish OrderCreated  -> times out
  -> order exists, nobody knows
  -> loss rate scales with broker incidents

WITH OUTBOX
  save order + outbox row -> ok (one transaction)
  broker down for 10 minutes
  -> rows accumulate, relay retries
  -> broker returns, backlog drains
  -> zero loss, delayed delivery

COST
  + one table, one background process
  + write amplification: 2 rows per event
  + added latency: poll interval (or ~ms with CDC)
  + duplicates that consumers must absorb

THE LATENCY KNOB
  poll every 1 s   -> simple, 0-1 s added latency
  poll every 50 ms -> more query load
  CDC              -> single-digit ms, more moving parts

WHAT IT DOES NOT SOLVE
  the consumer still needs deduplication; the outbox
  guarantees delivery, not uniqueness
```

| Metric | Value | Note |
|---|---|---|
| Guarantee | at-least-once | no loss |
| Atomicity | one transaction | **database only** |
| Added latency | poll interval | ms with CDC |
| Consumer | must dedupe | duplicates certain |

> **Parallel relays reorder messages**  
> SKIP LOCKED lets several relay instances drain the table concurrently, which is what makes the relay highly available — but it also means two messages for the same aggregate can reach the broker out of order. Where consumers depend on ordering, the drain must be partitioned so that all rows for a given key are handled by one worker, which trades some of that parallelism back.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Outbox table | Any relational database | Must be the same database as the business data |
| Polling relay | A background worker with SKIP LOCKED | Simplest; latency equals poll interval |
| CDC relay | Debezium, native logical replication | Low latency, no query load, more infrastructure |
| Broker | Kafka, RabbitMQ, SQS, any | The outbox is broker-agnostic |
| Consumer side | Inbox deduplication | The necessary counterpart |
| Alternative | Two-phase commit across database and broker | Atomic, but couples their availability |

The last row is the historical context: XA transactions across a database and a broker were the standard answer to this problem and were abandoned because the coupling and blocking were worse than the duplicates. The outbox achieves the same no-loss property using only a local transaction.

**Trade-offs**

**Ways to publish a message alongside a write**

| Approach | Can lose | Can duplicate | Cost |
|---|---|---|---|
| Dual write, database first | Yes | Yes | None |
| Dual write, broker first | Publishes phantoms instead | Yes | None |
| Transactional outbox, polling | No | Yes | Table plus worker |
| Transactional outbox, CDC | No | Yes | CDC infrastructure |
| Two-phase commit | No | No | Coupled availability, blocking |
| Event sourcing | No | Yes | Different data model entirely |

Event sourcing sits at the bottom because it dissolves the problem rather than solving it: when the event log is the system of record, there is no second write to keep in step. That is a much larger commitment than adding a table, but for systems that are event-driven throughout it removes the outbox entirely.

> **Ask before choosing it**  
> Does every state change of this kind have to produce a message, or is occasional loss tolerable? If a missed analytics event costs nothing, a dual write is fine and the outbox is overhead. The pattern earns its cost where the message drives money, inventory or another system's state.

**How it fails**

**How the outbox goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Messages lost despite the outbox | Row marked sent before publishing | Publish first, mark second, always |
| Dual write reintroduced | Outbox insert in its own transaction | Same transaction as the business write |
| Database slows down over weeks | Sent rows never pruned | Scheduled deletion or partitioned table |
| Consumers act on events twice | No deduplication downstream | Inbox pattern with idempotency keys |
| Messages arrive out of order | Parallel relays draining freely | Partition the drain by aggregate key |
| Silent messaging outage | Relay crashed; writes still succeed | Alert on outbox depth and oldest unsent row |
| Relay is a single point of failure | One instance without SKIP LOCKED | Multiple relays taking disjoint rows |

> **A stalled relay is invisible from the application's point of view**  
> Every write still succeeds, every API call returns 200, and nothing in the request path indicates a problem — while the outbox silently fills and no downstream system hears anything. The only signals are the depth of the unsent set and the age of its oldest row, and both must be alerted on, because by the time a consumer notices missing events the backlog may be hours deep.

**Where it is used**

- **Order and payment services** publishing domain events that other services must not miss.
- **Microservice integration generally**, where it has become the default answer to the dual-write problem.
- **CDC pipelines** that tail an outbox table specifically, rather than the business tables, to control the event schema.
- **Audit and compliance feeds**, where loss is unacceptable and duplication is merely untidy.
- **Saga orchestration**, where each step's message must be emitted exactly when its local transaction commits.

**In the interview**

The point to land is that dual writes have no correct ordering, and that the outbox converts consistency into delivery.

- **Show that neither ordering of a dual write works** — database first loses events, broker first publishes phantoms — before offering the pattern.
- **Say publish then mark**, and explain the failure if reversed: a crash between them loses the message rather than duplicating it.
- **State the guarantee as at-least-once** and immediately pair it with consumer-side deduplication.
- **Mention pruning and monitoring.** An outbox nobody alerts on is a silent messaging outage waiting to happen.
- **Contrast with XA** to show why this pattern replaced it — same no-loss property, no coupled availability.

**Practice drill**

A service saves an order and publishes an OrderCreated event. Show why both orderings of the dual write are unsafe, giving the specific failure for each. Design the outbox version: the schema, the transaction, and the relay loop. Say precisely why the relay publishes before marking sent, what happens when it crashes between those two steps, and what the consumer must therefore do. Then list the two metrics you would alert on and the threshold for each.

**Go deeper**

The transactional outbox makes a message part of the state change that caused it, so the two cannot diverge.

**The problem it solves has no ordering-based solution.** Writing to a database and publishing to a broker are two independent operations, and any crash between them leaves the system inconsistent: database first loses events, broker first announces things that never happened. Recognising that no sequencing fixes this — that it requires making one of the writes part of the other — is the insight the pattern encodes.

**It converts a consistency problem into a delivery problem, which is a much easier one.** Once the message is committed alongside the data, getting it to the broker can be retried indefinitely without risk, because the intent to send is durable. A broker outage becomes accumulated backlog and delayed delivery rather than permanent loss, and recovery is automatic when the broker returns.

**Publish before marking sent is the ordering that matters, and it is the one people get wrong.** Marking first and publishing second turns a crash in between into a lost message, which is precisely the failure the pattern exists to prevent. Publishing first turns the same crash into a duplicate, which is recoverable downstream. The pattern deliberately chooses the recoverable failure, and that choice is what makes at-least-once its ceiling.

**At-least-once forces a counterpart on the consumer.** Because duplicates are certain rather than hypothetical, every consumer needs deduplication — an inbox table, an idempotency key, or naturally idempotent handling. Teams that deploy the outbox and stop there have moved the failure downstream: no events are lost, but some are applied twice, and double-charging a card is not an improvement over losing an analytics event.

**Operationally, its danger is silence.** A failed relay does not fail any request; writes succeed, responses are normal, and only the growing set of unsent rows reveals that the rest of the system has stopped hearing anything. Depth and oldest-unsent-age are the two signals, and without alerts on them the first indication is a consumer noticing absence, which typically happens hours later.

**Its cost is small and its scope is narrow, which is why it displaced the alternatives.** One table, one background worker, some write amplification, and a latency floor set by the poll interval or by CDC. In exchange it provides the same no-loss property as a distributed transaction across database and broker, without coupling their availability or holding locks across a network call — and that comparison, more than any abstract argument, is why it is now the default.

**Related patterns:** Inbox Deduplication · Change Data Capture · Idempotency Key · Saga

---

### Inbox Deduplication

*Record every message identifier the consumer has processed, in the same transaction as its effects, so a redelivered message is recognised and skipped.*

> **When you hear…** At-least-once delivery · handlers with side effects that must not repeat · duplicates observed downstream

**Flow:** `Message arrives` → `Check inbox` → `Seen means skip` → `Apply and record` → `One transaction`

**The problem**

A payment service consumes a ChargeRequested message, calls the card processor, and acknowledges. If the acknowledgement is lost, the broker redelivers and the card is charged again. The consumer behaved correctly both times; the message simply arrived twice.

Every practical messaging system delivers at least once, because the alternative — acknowledging before processing — loses messages. Duplicates are therefore not an anomaly to be prevented upstream but a property to be absorbed downstream.

> **Remember what you have already done, atomically with doing it**  
> If the consumer records the message identifier in the same transaction as the effect it applies, then a redelivery finds the identifier already present and skips. The record and the effect can never disagree, because they commit together — which is the same trick as the outbox, applied at the other end of the pipe.

**Mental model**

A table of processed message identifiers. Every handler checks it before acting and writes to it as part of acting, in one transaction.

1. **Receive** — A message arrives with a stable identifier.
2. **Check** — If the identifier is already in the inbox, acknowledge and stop.
3. **Apply and record** — The business effect and the inbox row commit in one transaction.
4. **Acknowledge** — The broker is told only after the commit succeeds.
5. **Expire** — Old identifiers are pruned beyond the window in which redelivery is possible.

> **The identifier must come from the producer, not from the broker**  
> Broker-assigned message identifiers change on redelivery in some systems, and change entirely if a message is republished from an outbox after a relay crash. The deduplication key has to be something the producer generates and repeats — an event identifier or an idempotency key carried in the payload — or the inbox will faithfully record two different identifiers for the same logical event.

**How it works**

**The handler**

```text
on message m:
  BEGIN
    INSERT INTO inbox (message_id, processed_at)
    VALUES (m.id, now())
    ON CONFLICT DO NOTHING
    -- if no row was inserted, we have seen it
    if rows_affected == 0:
      ROLLBACK
      ack(m)            <- already done; just ack
      return

    apply_business_effect(m)
  COMMIT
  ack(m)

WHY THE INSERT COMES FIRST
  the unique constraint does the mutual exclusion;
  two concurrent deliveries of the same message race
  on the index and exactly one wins

WHY ONE TRANSACTION
  record then apply, separately
    -> crash between them: recorded but never applied,
       and the redelivery will SKIP it. Lost.
  apply then record, separately
    -> crash between them: applied twice on redelivery.
  together
    -> neither

EXTERNAL EFFECTS DO NOT JOIN THE TRANSACTION
  charging a card cannot be rolled back
  -> pass the message id to the processor as its
     idempotency key, so the duplicate is absorbed
     there too
```

1. **Use a producer-generated identifier** — Broker identifiers can change between deliveries and defeat the whole mechanism.
2. **Put the inbox insert and the effect in one transaction** — Split them and every crash window either loses or duplicates.
3. **Rely on a unique constraint, not a read-then-write** — Concurrent deliveries race; the index resolves it, application logic does not.
4. **Acknowledge only after commit** — Acknowledging first converts at-least-once into at-most-once.
5. **Prune beyond the redelivery window** — Retention must exceed the broker's maximum retry horizon, not just a convenient number.
6. **For external side effects, pass the identifier through** — A card charge cannot be rolled back; it must be deduplicated at the processor.

**Sizing the inbox and choosing retention**

```text
1,000 messages/s, retention 7 days
  1,000 x 86,400 x 7 = 604,800,000 rows
  id (16 B) + timestamp (8 B) + index overhead
  -> tens of GB, dominated by the unique index

REDUCING IT
  retention 24 h -> 86.4 M rows, a few GB
  but ONLY if the broker cannot redeliver after 24 h

HOW TO CHOOSE RETENTION
  it must exceed the longest possible gap between
  two deliveries of the same message:
    broker retry policy
    + dead letter replay window
    + manual reprocessing you actually do
  -> if operators replay a week-old dead letter queue,
     24 h retention will let those duplicates through

PARTITION BY DAY
  drop yesterday's partition instead of deleting rows
  -> pruning stops competing with the write path
```

| Metric | Value | Note |
|---|---|---|
| Guarantee | effect once | **per identifier** |
| Mechanism | unique index | handles races |
| Retention | beyond replay | including manual |
| External calls | pass the key on | cannot roll back |

> **Retention shorter than your replay habits reintroduces duplicates**  
> The inbox only recognises what it still remembers. A team that keeps identifiers for 24 hours and then replays a dead letter queue from last week will process every one of those messages again, with no warning — the inbox will happily insert identifiers it has forgotten. Retention is not a storage decision; it is determined by the longest redelivery gap the system can actually produce, manual operations included.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Inbox table | Relational table with a unique index | Must share the transaction with the effect |
| Key source | Producer event id, idempotency key | Never a broker-assigned delivery id |
| Pruning | Time-partitioned table | Dropping a partition beats deleting rows |
| External idempotency | Payment processor idempotency keys | The only way to dedupe an unrollbackable call |
| Alternative | Naturally idempotent handlers | Set a value rather than increment it |
| Alternative | Broker exactly-once semantics | Real but scoped to that broker's boundaries |

Naturally idempotent handlers deserve first consideration: a handler that sets a status to shipped rather than appending a shipment event needs no inbox at all. Redesigning the effect to be repeatable is cheaper than remembering every message, and it is available more often than people assume.

**Trade-offs**

**Ways to absorb duplicates**

| Approach | Storage | Covers external calls | Cost |
|---|---|---|---|
| Inbox table | Grows with throughput | Only if the key is passed on | A table plus pruning |
| Naturally idempotent handler | None | Sometimes | Requires a compatible effect |
| Unique constraint on business data | None extra | No | Only where a natural key exists |
| Broker exactly-once | Managed | No | Scoped to the broker |
| Ignore duplicates | None | No | Only where repeating is harmless |

The fourth row is worth stating carefully in conversation: broker-level exactly-once is genuine within the broker's own transactional boundary, and stops being a guarantee the moment the handler calls anything outside it — which is nearly always.

> **Ask before choosing it**  
> What actually happens if this message is processed twice? A duplicate charge demands an inbox; a recomputed aggregate may not care. Then ask what the longest possible gap between two deliveries is, including manual replays, because that number is the retention and it is usually larger than the first guess.

**How it fails**

**How inbox deduplication goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicates still applied | Broker delivery id used as the key | Use a producer-generated identifier |
| Message silently lost | Recorded in a separate transaction before applying | One transaction for record and effect |
| Concurrent deliveries both applied | Read-then-write check instead of a constraint | Unique index and insert-first |
| Old replays processed again | Retention shorter than the replay window | Retention must exceed the longest redelivery gap |
| Inbox table dominates the database | No pruning at high throughput | Time partitioning with partition drops |
| Card charged twice despite the inbox | External call not idempotent | Pass the identifier as the processor's key |
| Messages lost on restart | Acknowledged before committing | Acknowledge after commit only |

> **Recording before applying converts duplication into loss**  
> If the identifier is written in its own transaction and the process dies before the effect is applied, the redelivery finds the identifier present and skips — the message is gone, permanently, and no retry will bring it back. This is strictly worse than the duplicate it was meant to prevent, and it is the most common way a well-intentioned inbox implementation fails.

**Where it is used**

- **Payment consumers**, where a duplicate is a real charge and the inbox pairs with processor idempotency keys.
- **Inventory and ledger updates**, where the effect is an increment and therefore not naturally repeatable.
- **Any consumer downstream of a transactional outbox**, since that pattern guarantees at-least-once by construction.
- **Webhook receivers**, where the sender retries aggressively and the identifier arrives in a header.
- **Dead letter replay tooling**, which is precisely the operation that exposes short retention.

**In the interview**

The strong answer names the transaction boundary and the key source without being asked, because those are the two ways it fails.

- **Say the effect and the inbox row commit together**, and give the failure for each split ordering — one loses, one duplicates.
- **Insist on a producer-generated identifier.** Broker delivery ids change, and using them makes the inbox decorative.
- **Use the unique constraint as the concurrency mechanism** rather than a read-then-write check, which races.
- **Tie retention to the longest redelivery gap**, including manual dead-letter replay, not to a convenient storage figure.
- **Extend to external calls** by passing the identifier as the downstream idempotency key, since those cannot be rolled back.

**Practice drill**

A consumer receives ChargeRequested and calls a payment processor. Design the inbox: schema, the transaction boundary, and where the acknowledgement goes. Explain what breaks if the inbox row is written in a separate transaction before the charge, and what breaks if it is written after. Then choose a retention period, justify it from the broker's retry policy and your team's replay practices, and estimate the table size at 1,000 messages per second.

**Go deeper**

Inbox deduplication makes a consumer idempotent by recording what it has processed in the same transaction as the processing.

**It is the necessary counterpart to at-least-once delivery, not an optional hardening.** Every messaging system that does not lose messages will sometimes repeat them, because acknowledging before processing is the only way to avoid duplication and it trades loss for it. Treating duplicates as a certainty to be absorbed rather than an anomaly to be prevented is what makes the consumer correct rather than usually correct.

**The single transaction is the whole mechanism.** Writing the identifier and applying the effect together means no crash can leave them disagreeing. Split them and each ordering fails distinctly: recording first turns a crash into permanent loss, because the redelivery will skip a message that was never applied; applying first turns it into the duplicate the pattern was built to stop. The atomic version is the only one that is safe in both directions.

**The key must originate with the producer.** Broker-assigned delivery identifiers can differ between attempts, and a message republished from an outbox after a relay crash is, from the broker's perspective, entirely new. A deduplication table keyed on anything the transport assigns will accumulate distinct identifiers for the same logical event and let every duplicate through, while appearing to work.

**Concurrency is resolved by the index, not by logic.** Two deliveries of the same message can be handled simultaneously by two workers, and a read-then-write check will let both pass. Inserting first and relying on a unique constraint makes the database arbitrate: exactly one insert succeeds, the other sees a conflict and skips. This is a case where pushing the decision down to a storage primitive is more correct than any amount of application code.

**Retention is determined by operations, not by storage cost.** The inbox recognises only what it still holds, so the window must exceed the longest gap that can occur between two deliveries of one message — broker retries, dead letter queues, and the manual replays a team actually performs. Choosing 24 hours because the table is large and then replaying a week-old dead letter queue processes everything again, silently.

**Effects outside the database do not join the transaction, and must be handled separately.** A card charge cannot be rolled back by the database that recorded it, so the identifier has to travel onward as the processor's idempotency key, letting the duplicate be absorbed where the effect actually happens. Where that is impossible, the honest position is that the handler is idempotent only up to the boundary of the transaction, and the remaining window must be acknowledged rather than assumed away.

**Related patterns:** Transactional Outbox · Idempotency Key · Work Queue · Retry With Jitter

---

### Work Queue

*Put units of work in a durable queue and let a pool of workers pull from it, so producers never wait and capacity scales by adding consumers.*

> **When you hear…** A request triggers slow work · load arrives in bursts · producers and consumers should scale independently

**Flow:** `Producer enqueues` → `Durable queue` → `Workers pull` → `Ack on success` → `Retry or dead letter`

**The problem**

Uploading a video returns only after transcoding finishes, so the request holds a connection for four minutes and any deploy, timeout or network blip loses the work entirely. Doing the work inside the request couples the user's patience to the system's throughput.

Traffic is also uneven. Sizing synchronous capacity for the peak wastes money for most of the day, and sizing it for the mean drops work at the peak, because there is nowhere to put the excess.

> **A queue is a buffer that converts a capacity problem into a latency problem**  
> Once work is durably enqueued, a burst that exceeds processing capacity no longer fails — it waits. Producers respond immediately with an identifier, consumers process at whatever rate they can sustain, and the two sides stop having to be the same size. The cost is that completion is no longer immediate, and the system must be able to say where the work got to.

**Mental model**

A durable list of tasks with competing consumers. Each task goes to exactly one worker at a time, is acknowledged on success, and returns to the queue if the worker fails.

1. **Enqueue** — The producer writes the task durably and returns an identifier.
2. **Lease** — A worker takes a task and holds an invisibility window on it.
3. **Process** — The work runs, with the lease extended if it takes longer than expected.
4. **Acknowledge** — The task is deleted only after success.
5. **Retry or park** — Failures return to the queue; repeated failures go to a dead letter queue.

> **The visibility timeout is a guess, and being wrong duplicates work**  
> A worker holds a task invisible for a fixed window. If processing outlives that window the task becomes visible again and a second worker starts it while the first is still running — the same work executes twice, concurrently. Either the timeout must exceed the true worst-case duration, or the worker must extend the lease as it goes, and handlers must be idempotent regardless.

**How it works**

**Lease, acknowledge, retry**

```text
WORKER LOOP
  task = queue.receive(visibility = 30 s)
  if none: continue
  try:
    process(task)            <- must be idempotent
    queue.ack(task)          <- delete
  except:
    queue.nack(task)         <- immediate retry
                                or let the lease lapse

LONG TASKS
  while processing:
    every 15 s: queue.extend(task, +30 s)
  -> the lease tracks reality instead of a guess

RETRY POLICY
  attempt 1  immediate
  attempt 2  +2 s
  attempt 3  +8 s
  attempt 4  +32 s        (exponential, jittered)
  attempt 5  -> DEAD LETTER QUEUE

WHY A DEAD LETTER QUEUE
  a task that fails deterministically will fail
  forever; retrying it consumes the capacity that
  healthy tasks need
  -> park it, alert, let a person look

WHAT ACK-BEFORE-PROCESS COSTS
  ack first  -> crash loses the task, silently
  ack after  -> crash duplicates the task
  choose duplication; it is recoverable
```

1. **Acknowledge only after the work succeeds** — Acknowledging on receipt turns every crash into silent work loss.
2. **Make handlers idempotent** — Visibility expiry and redelivery guarantee some tasks run twice.
3. **Extend the lease for long tasks** — A fixed timeout shorter than the work causes concurrent duplicate execution.
4. **Retry with exponential backoff and jitter** — Immediate retries of a failing dependency amplify the outage.
5. **Send repeat failures to a dead letter queue** — Poison tasks otherwise consume capacity indefinitely.
6. **Alert on queue depth and oldest message age** — Depth alone is ambiguous; age tells you whether it is draining.

**Sizing the worker pool**

```text
ARRIVALS 100 tasks/s
SERVICE TIME 200 ms per task

LITTLE'S LAW
  workers needed = arrival rate x service time
                 = 100 x 0.2 = 20 concurrent workers
  -> at exactly 20 the queue is marginally stable and
     latency explodes; run 25-30

UTILISATION AND WAIT
  70% utilisation -> wait ~ 2.3x service time
  90% utilisation -> wait ~ 9x service time
  95% utilisation -> wait ~ 19x service time
  -> the last 20% of capacity buys enormous latency

BURST ABSORPTION
  peak 500/s for 60 s with 30 workers (150/s capacity)
    arrivals  30,000
    processed  9,000
    backlog   21,000 tasks
    drain at 50/s spare -> ~7 minutes
  -> acceptable for transcoding, not for a task whose
     result someone is watching for

THE REAL QUESTION
  not can it absorb the burst, but is the resulting
  delay acceptable to whoever is waiting
```

| Metric | Value | Note |
|---|---|---|
| Producer latency | immediate | returns an id |
| Capacity | add workers | **scales horizontally** |
| Utilisation | run at 70% | not 95% |
| Guarantee | at-least-once | idempotent handlers |

> **Queues hide failure until the backlog is enormous**  
> A queue absorbing work looks identical to a queue processing it, from the producer's side: enqueue succeeds either way. If consumers have stopped, nothing fails, nothing errors, and the only visible signal is a number climbing on a dashboard nobody is watching. This is why oldest-message-age is the more useful alert than depth — a deep queue that is draining is healthy, and a shallow one that is not draining is not.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| SQS and cloud queues | Managed, effectively unbounded, simple | At-least-once; ordering only in FIFO mode |
| RabbitMQ | Rich routing, priorities, mature | Queue depth affects broker memory |
| Redis lists and streams | Very low latency, simple | Durability depends on persistence configuration |
| Database table as a queue | Transactional with business data | Needs SKIP LOCKED; contention at high rates |
| Kafka as a work queue | High throughput, replayable | Partition count caps consumer parallelism |
| Job frameworks | Scheduling, retries, dashboards included | Opinionated; another runtime to operate |

The database-as-queue row is more respectable than its reputation suggests. Below a few thousand tasks per second, SKIP LOCKED makes it work well, and enqueueing in the same transaction as the business write removes the dual-write problem entirely — which is the outbox pattern wearing a different hat.

**Trade-offs**

**Work queue against the alternatives**

| Approach | Producer waits | Absorbs bursts | Complexity |
|---|---|---|---|
| Synchronous processing | Yes, for the full duration | No | None |
| Thread pool in the same process | No | Only until restart | Low, but loses work on crash |
| Durable work queue | No | Yes | Queue, workers, retries, DLQ |
| Partitioned log | No | Yes | Ordering per key; replay |
| Scheduled batch | No | Yes | Latency equals the batch interval |

The in-process thread pool is the tempting shortcut and the one that bites: it removes the latency without providing durability, so a deploy or a crash silently discards everything queued in memory — which is precisely the work nobody knows to retry.

> **Ask before choosing it**  
> Who is waiting for this result, and how do they find out it is done? Asynchronous processing moves the completion problem to the client, so the design needs polling, a callback or a push before it is finished. A queue added without answering that leaves users staring at a spinner that never resolves.

**How it fails**

**How work queues go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Work silently disappears | Acknowledged before processing | Acknowledge only after success |
| A task runs twice concurrently | Visibility timeout shorter than the work | Extend the lease; make handlers idempotent |
| Queue grows without bound | Consumers stopped or too few | Alert on oldest-message age, not just depth |
| One poison task blocks everything | Infinite retries on a deterministic failure | Retry limit and a dead letter queue |
| Retries worsen an outage | Immediate retries against a failing dependency | Exponential backoff with jitter |
| Latency spikes at moderate load | Pool sized to average, running near capacity | Target 70% utilisation, not 95% |
| Dead letter queue quietly fills | Nobody alerted on it | Alert on any message entering the DLQ |
| Ordering violated | Competing consumers on related tasks | Partition by key, or use a log |

> **An unwatched dead letter queue is a data loss mechanism with extra steps**  
> Parking poison messages is correct, and it only works if someone looks. A dead letter queue that accumulates for weeks means the work was discarded with a ceremony rather than dropped outright — and because the tasks that fail deterministically are frequently the ones touching unusual, important cases, the loss is biased towards exactly what mattered most.

**Where it is used**

- **Video transcoding and image processing**, the canonical long-running task behind an immediate response.
- **Email and notification delivery**, where third-party latency and failure must not reach the request path.
- **Report generation**, where work is bursty and the user expects to be told when it is ready.
- **Order fulfilment steps**, often as a database-backed queue written in the same transaction as the order.
- **Machine learning inference batching**, where the queue smooths arrivals into efficient GPU batches.

**In the interview**

The differentiators are acknowledgement ordering, the visibility timeout, and knowing that the client still needs an answer.

- **Say acknowledge after processing** and give the failure for the reverse — silent loss, which no retry recovers.
- **Raise the visibility timeout as a source of duplicate concurrent execution** and offer lease extension as the fix.
- **Size the pool with Little's law** and then argue for headroom, because 95% utilisation multiplies queueing delay.
- **Insist on a dead letter queue with an alert.** Parking poison messages without watching them is quiet data loss.
- **Address completion notification.** Moving work off the request path is only half a design; the client still needs to learn the outcome.

**Practice drill**

A video upload triggers transcoding that takes 4 minutes. Design the queue path end to end, including what the upload request returns and how the user learns it is finished. Size the worker pool for 100 tasks per second at 200 ms each, and state the utilisation you would run at and why. Then trace three failures: a worker crashing mid-task, a task whose processing exceeds the visibility timeout, and a task that fails deterministically every time.

**Go deeper**

A work queue decouples the arrival of work from its execution, letting producers respond immediately while consumers process at a sustainable rate.

**It converts a capacity problem into a latency problem, and that is the actual trade.** Work that exceeds processing capacity waits rather than failing, so a burst becomes a backlog and a queue depth chart instead of a wall of errors. This is only an improvement if the delay is acceptable to whoever is waiting, which makes the first design question about the consumer of the result rather than about the queue.

**Acknowledgement ordering determines whether the system loses work or repeats it.** Acknowledging on receipt means a worker crash discards the task with nothing left to indicate it existed; acknowledging after success means the same crash redelivers it. The pattern chooses redelivery, because a duplicate can be absorbed by an idempotent handler while a silent loss can be absorbed by nothing.

**The visibility timeout is where duplicate execution actually comes from.** A lease shorter than the real processing time makes the task visible again while the first worker is still running, so two workers do the same job concurrently — a much sharper failure than sequential redelivery, since both may write. Extending the lease during processing keeps the timeout tied to reality rather than to an estimate made when the queue was configured.

**Utilisation governs latency far more than average capacity does.** Queueing delay rises non-linearly with utilisation: comfortable at 70%, painful at 90%, unusable at 95%. Pools sized to the average arrival rate sit permanently in that steep region, which is why a system that looks adequately provisioned on a throughput graph produces wildly variable completion times. Headroom is not waste; it is the mechanism that keeps waiting bounded.

**Poison messages must be parked, and parking must be watched.** A task that fails deterministically will keep failing, consuming retry capacity that healthy work needs, so a retry limit and a dead letter queue are structural rather than optional. But an unmonitored dead letter queue simply relocates the loss, and the messages that land there tend to represent unusual and important cases — so an alert on the first arrival, rather than a periodic review, is the correct posture.

**Queues fail quietly, so the instrumentation has to be chosen deliberately.** Enqueueing succeeds whether or not anything is consuming, which means a total consumer outage produces no errors anywhere in the request path. Depth alone is ambiguous — a deep queue draining fast is healthy — so the signal that actually distinguishes working from stalled is the age of the oldest unprocessed message, and systems that alert only on depth routinely discover the problem hours after it began.

**Related patterns:** Partitioned Log · Backpressure · Retry With Jitter · Inbox Deduplication

---

### Partitioned Log

*Append events to an ordered, replayable log split into partitions, so consumers read at their own pace with ordering guaranteed per key.*

> **When you hear…** Several consumers need the same stream · ordering per entity matters · events must be replayable after a bug or a new consumer

**Flow:** `Append by key` → `Hash to partition` → `Ordered offsets` → `Consumers track offset` → `Replay from any point`

**The problem**

A queue deletes a message once it is acknowledged, which is correct for tasks and wrong for events. When a second team needs the same order events, or a consumer's logic had a bug last Tuesday, the data required to serve them has already been consumed and destroyed.

Competing consumers also destroy ordering. Two workers pulling from the same queue can process an item's update before its creation, because nothing ties related messages to the same worker.

> **Keep the log and let readers hold the position**  
> If events are appended to an ordered log that is retained rather than consumed, then reading becomes a cursor over immutable data. Many consumers can read the same events independently, a new consumer can start from the beginning, and a broken one can rewind — because reading no longer destroys. Partitioning by key then gives ordering where it matters while still allowing parallelism.

**Mental model**

An append-only file per partition, with records addressed by a monotonically increasing offset. Producers append; consumers remember where they are.

1. **Partition** — A key is hashed to choose a partition, so all events for that key land in one place.
2. **Append** — Records are written in order and assigned offsets.
3. **Retain** — Records stay for a configured window regardless of who has read them.
4. **Consume** — Each consumer group tracks its own offset per partition.
5. **Replay** — Resetting the offset re-reads history without any cooperation from the producer.

> **Ordering is per partition only, and the partition count is nearly permanent**  
> There is no global order across partitions, so events for different keys have no defined relative sequence. Worse, adding partitions changes the hash mapping and sends a key to a new partition while its history remains in the old one — breaking the very ordering guarantee the design relied on. Partition count is a capacity decision made early and paid for late.

**How it works**

**Partitions, offsets and consumer groups**

```text
TOPIC orders, 6 partitions

producer.send(key = order_id, value = event)
  partition = hash(order_id) % 6
  -> every event for one order is in one partition,
     in append order

PARTITION 3
  offset:  ...  1041   1042   1043   1044
  events:       created updated paid  shipped
  -> strictly ordered within the partition

CONSUMER GROUPS
  group "billing"    at offset 1044   (caught up)
  group "analytics"  at offset 1012   (32 behind)
  group "search"     at offset 0      (new, replaying)
  -> independent cursors over the SAME records

PARALLELISM CEILING
  consumers in a group <= partitions
  6 partitions -> at most 6 useful consumers
  a 7th sits idle
  -> partition count is the throughput ceiling

REPARTITIONING HAZARD
  6 -> 12 partitions
  hash(key) % 6 = 3   but   hash(key) % 12 = 9
  -> new events for that key go to partition 9
  -> history stays in 3
  -> per-key ordering is broken across the change
```

1. **Choose the partition key for the ordering you need** — The key defines the unit of ordering; everything else is unordered relative to it.
2. **Over-provision partitions initially** — Adding them later breaks per-key ordering, so headroom is cheaper than the migration.
3. **Commit offsets after processing** — Committing first loses events on a crash, exactly as acknowledging early does in a queue.
4. **Set retention by replay need, not by disk comfort** — Retention is how far back a broken consumer can recover from.
5. **Watch consumer lag per partition** — Aggregate lag hides one stuck partition behind five healthy ones.
6. **Expect duplicates at rebalance** — Partition reassignment replays from the last committed offset; handlers must be idempotent.

**Choosing retention and reading lag**

```text
RETENTION IS A RECOVERY BUDGET
  7 days retention means:
    a consumer can be broken for up to 7 days and
    still recover every event by rewinding
    a new consumer can rebuild from 7 days of history
  -> shorter retention is cheaper and reduces the
     window in which mistakes are fixable

SIZING
  100,000 events/s x 1 KB x 86,400 x 7
    = ~60 TB before replication
  replication factor 3 -> ~180 TB
  -> retention is a real cost, not a free setting

LAG IS THE PRIMARY SIGNAL
  lag = latest offset - committed offset
  per partition, never only in aggregate

  5 partitions at lag 0 and one at lag 2,000,000
  -> aggregate looks fine
  -> one key range is hours behind and nobody knows

HOT PARTITION
  one key far more active than the rest
  -> that partition saturates while others idle
  -> the fix is the key, not more partitions
```

| Metric | Value | Note |
|---|---|---|
| Ordering | per partition | **not global** |
| Consumers | independent offsets | same records |
| Parallelism cap | partition count | set early |
| Retention | replay budget | real storage cost |

> **A hot partition cannot be fixed by adding partitions**  
> If one key dominates the traffic, every event for it goes to a single partition by definition, and that partition becomes the bottleneck no matter how many others exist. More partitions spread the other keys and leave the hot one exactly as it was. The remedy is to change the key — adding a suffix to spread a hot entity across several partitions — which means giving up strict ordering for that entity, and that has to be an explicit decision.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| Kafka | The reference implementation; huge throughput | Operational weight; partition count is sticky |
| Managed streaming services | Kafka semantics without running it | Partition or shard limits and pricing |
| Pulsar | Separates serving from storage; tiered offloading | Smaller ecosystem |
| Redis streams | Log semantics at very low latency | Retention bounded by memory |
| Traditional queue | Simpler when replay is not needed | Consumption destroys; no fan-out to new readers |
| Event store | Log as the system of record | A modelling commitment, not just transport |

The queue row is the honest comparison. If exactly one consumer processes each item, ordering does not matter and nobody will ever replay, a queue is simpler in every operational dimension and a log is unnecessary weight.

**Trade-offs**

**Log against queue**

| Property | Partitioned log | Work queue |
|---|---|---|
| Consumption | Non-destructive; offsets | Destructive; acknowledge and delete |
| Fan-out | Any number of independent groups | Requires separate queues per consumer |
| Ordering | Guaranteed per partition key | None with competing consumers |
| Replay | Rewind to any retained offset | Not possible once acknowledged |
| Parallelism | Capped by partition count | Any number of workers |
| Per-message retry | Awkward; blocks the partition | Natural; retry that message alone |

The last row is the log's real weakness. A single failing record at the head of a partition blocks every record behind it, so a log needs a deliberate strategy — skip and park to a side topic, or accept head-of-line blocking — where a queue handles it natively by retrying one message.

> **Ask before choosing it**  
> Will anyone ever need to read these events again — a second consumer, a rebuild, a bug fix? If yes, retention and replay justify the log. If the answer is genuinely no, and there is one consumer with no ordering requirement, the queue is the better-fitting tool.

**How it fails**

**How partitioned logs go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Per-key ordering broken | Partition count changed | Over-provision early; treat repartitioning as a migration |
| One partition saturated | Hot key concentrating traffic | Change the key; accept weaker ordering for that entity |
| Events lost on consumer crash | Offset committed before processing | Commit after processing completes |
| A stuck partition goes unnoticed | Only aggregate lag monitored | Alert on per-partition lag |
| Whole partition stalls | One record failing repeatedly at the head | Skip to a side topic after a retry limit |
| Replay impossible when needed | Retention shorter than time to notice | Set retention from detection time, not disk cost |
| Duplicates after a deploy | Rebalance replaying from the committed offset | Idempotent handlers |
| Idle consumers | More consumers than partitions | Consumer count cannot exceed partitions |

> **Repartitioning silently breaks the guarantee people built on**  
> Increasing partitions to add throughput remaps keys, so a key whose history sits in one partition begins writing to another. Ordering for that key is now split across two partitions with no defined relationship, and nothing errors — consumers keep running, events keep flowing, and the anomalies appear later as state that updated in the wrong sequence. This is why the initial partition count is treated as near-permanent.

**Where it is used**

- **Event backbones between services**, where the same stream feeds billing, analytics and search independently.
- **Change data capture pipelines**, which need per-row ordering and the ability to rebuild a downstream store.
- **Metrics and log aggregation**, where throughput is enormous and consumers are many.
- **Event sourcing**, where the log is the system of record and replay reconstructs state by design.
- **Stream processing jobs** that maintain materialised views and must be able to reprocess from a known offset.

**In the interview**

The strong answer distinguishes log from queue on replay and ordering, and volunteers the partition-count trap.

- **Lead with non-destructive consumption.** Independent offsets over retained records are what makes fan-out and replay possible at all.
- **State that ordering is per partition, never global**, and tie the partition key directly to the ordering requirement.
- **Raise repartitioning unprompted.** It breaks per-key ordering silently and is the most expensive mistake available here.
- **Name the parallelism ceiling** — consumers in a group cannot exceed partitions — since it turns partition count into a capacity decision.
- **Mention head-of-line blocking**, because per-message retry is the thing a queue does better and interviewers listen for the weakness.

**Practice drill**

Design an order event stream consumed by billing, analytics and search. Choose the partition key and justify it from the ordering requirement. Pick a partition count for 50,000 events per second and explain what you would have to do to change it later. Set a retention period and justify it as a recovery budget rather than a storage figure. Then say what happens when one record at the head of a partition fails repeatedly, and what you would do about it.

**Go deeper**

A partitioned log stores events as an ordered, retained sequence that many consumers read independently by tracking their own position.

**Non-destructive consumption is the property everything else follows from.** Because reading does not remove, several consumer groups can traverse the same records at different speeds, a new consumer can start from the beginning and rebuild its state, and a consumer that processed events incorrectly can rewind and try again. A queue cannot offer any of this, not because of an implementation gap but because acknowledgement is defined as removal.

**Partitioning is the compromise that makes ordering and parallelism coexist.** Strict global ordering would mean one writer and one reader; no ordering would mean arbitrary interleaving of a single entity's events. Hashing a key to a partition gives strict order within each key while allowing as many parallel streams as there are partitions, which is exactly the granularity most domains need — an order's events must be sequential, but two different orders need no relationship.

**The partition count is a long-lived commitment, and this surprises people.** It caps consumer parallelism, since a consumer group cannot usefully exceed it, and it cannot be raised without remapping keys. That remapping splits a key's history across two partitions with no defined ordering between them, silently, while everything continues to appear healthy. Choosing generously at the start is far cheaper than the migration that a later increase actually requires.

**Hot partitions are a key-design problem wearing a capacity disguise.** When one key carries a disproportionate share of traffic, its partition saturates while the others idle, and adding partitions redistributes everything except the key causing the problem. The only real remedies change the key — spreading one entity across several partitions by adding a suffix — and that trades away the strict ordering for that entity, which has to be a decision rather than an accident.

**Retention is a recovery budget.** It determines how long a consumer can be broken and still recover by replaying, and how much history a newly added consumer can build from. Setting it from available disk rather than from how long a problem typically takes to notice produces a system where the fix is theoretically available and practically expired — the bug is found on Thursday and the events from Monday are gone.

**Its weakness relative to a queue is per-message failure handling.** A record that fails repeatedly sits at the head of its partition and blocks everything behind it, because offsets advance in order. Queues retry one message and move on; logs require a deliberate policy — bounded retries then diverting the record to a side topic — and systems that omit that policy discover it when one malformed event halts a partition for hours.

**Related patterns:** Work Queue · Change Data Capture · Stream Processing · Consistent Hashing

---

## Reliability

### Idempotency Key

*The client sends a unique key with a request so the server can recognise a retry and return the original result instead of performing the action again.*

> **When you hear…** Requests with side effects over an unreliable network · clients that retry · a timeout that might have succeeded

**Flow:** `Client generates key` → `Server records key` → `Executes once` → `Stores response` → `Retry returns stored`

**The problem**

A payment request times out after eight seconds. The client has no idea whether the charge happened: the request may have been lost on the way, or processed successfully with the response lost on the way back. Both look identical from the outside.

Retrying risks charging twice. Not retrying risks a customer who paid nothing and believes they did, or who abandons the purchase. There is no safe choice available to the client alone, because the ambiguity lives at the server.

> **Let the client name the operation, not just describe it**  
> If the client attaches an identifier it generates once and reuses on every retry, the server can distinguish a new operation from a repeat of a known one. The ambiguity does not disappear — the client still cannot tell what happened — but it stops mattering, because retrying is now guaranteed to be safe.

**Mental model**

A server-side record keyed by the client's identifier, holding the outcome of the first attempt. Subsequent requests with the same key are answered from that record rather than executed.

1. **Generate** — The client creates a key once, before the first attempt, and reuses it for every retry.
2. **Claim** — The server inserts the key; a conflict means this operation is already known.
3. **Execute** — The first claimant performs the action and stores the response.
4. **Replay** — A repeat request returns the stored response, unchanged.
5. **Expire** — Keys are retained long enough to cover every realistic retry, then dropped.

> **A key generated per attempt provides nothing**  
> If the client creates a new identifier each time it retries, every attempt looks like a distinct operation and the server executes all of them. The key must be created once for the logical operation — when the user presses the button — and carried unchanged through every retry, including retries after a process restart, which means it frequently has to be persisted client-side.

**How it works**

**Claim, execute, store, replay**

```text
POST /payments
Idempotency-Key: 7c9e-4f21-...

SERVER
  BEGIN
    INSERT INTO idempotency (key, state)
    VALUES (k, 'in_progress')
    ON CONFLICT DO NOTHING
    if conflict:
      row = SELECT * FROM idempotency WHERE key = k
      if row.state == 'done':  return row.response
      else:                    return 409 in progress
  COMMIT

  result = perform_charge()

  UPDATE idempotency
  SET state = 'done', response = result
  WHERE key = k

  return result

WHY 'in_progress' EXISTS
  two concurrent retries must not both execute; the
  second sees in_progress and is told to wait rather
  than being given a wrong answer

WHAT IS STORED
  the response body AND status code
  -> a retry must see exactly what the first caller
     saw, including a 4xx

BIND THE KEY TO THE REQUEST
  store a hash of the request body with the key
  -> same key + different body = client bug
  -> reject with 422 rather than returning someone
     else's result
```

1. **Generate the key once per logical operation** — Per-attempt keys defeat the mechanism entirely.
2. **Claim atomically with a unique constraint** — Concurrent retries race, and application-level checks lose that race.
3. **Store the response, not just the fact of completion** — The retry must receive the original outcome, including its status code.
4. **Record an in-progress state** — A second concurrent attempt needs a defined answer while the first is still running.
5. **Bind the key to the request content** — The same key with a different body is a client bug and must be rejected, not silently answered.
6. **Set retention from the client's retry horizon** — Expiring before clients stop retrying reopens the duplicate window.

**Where the window still exists**

```text
SAFE
  request lost         -> retry executes, correct
  response lost        -> retry replays stored, correct
  client crash + retry -> replays stored, correct
  two concurrent tries -> one executes, one waits

STILL AMBIGUOUS
  server crashes AFTER charging the card and BEFORE
  writing state = 'done'
    -> key says in_progress forever
    -> retry sees in_progress and waits
    -> resolution requires reconciling with the card
       processor, not more retries

THIS IS WHY THE PROCESSOR NEEDS THE KEY TOO
  pass the same key downstream
  -> the duplicate is absorbed at the place the money
     actually moves
  -> your record can then be rebuilt from theirs

RETENTION
  24 h  covers automatic client retries
  7 d   covers mobile clients retrying after reinstall
       and support-driven resubmission
  -> shorter retention silently reopens the window
```

| Metric | Value | Note |
|---|---|---|
| Client duty | one key | **per operation** |
| Server duty | claim, store, replay | atomic |
| Residual gap | crash mid-execution | needs reconciliation |
| Pass downstream | same key | to the processor |

> **Returning a fresh result instead of the stored one defeats the purpose**  
> A server that re-executes and returns the new outcome on a repeated key has an idempotency table that records history without changing behaviour. The retry must produce the original response — identical body, identical status — because the client's entire reason for retrying was to learn what happened the first time, and giving it a second, different answer creates two records of one operation.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Key store | Relational table with a unique index | Must be transactional with the effect where possible |
| Key store | Redis with TTL | Fast, but persistence settings decide correctness |
| Key generation | UUIDv4 on the client | Must be stable across retries and restarts |
| Transport | Idempotency-Key header | The de facto convention for HTTP APIs |
| Downstream | Processor idempotency keys | Pass the same key onward; the effect is there |
| Related | Inbox deduplication | The same idea applied to messages |

Redis is the common choice and deserves one caveat: if the key store loses data on failover, the duplicate protection vanishes precisely during the incident when clients are retrying hardest. For money-moving operations the keys belong in the same durable store as the effect.

**Trade-offs**

**Ways to make an operation safe to retry**

| Approach | Client work | Server work | Covers concurrency |
|---|---|---|---|
| Idempotency key | Generate and persist a key | Table, claim, store response | Yes, with a constraint |
| Natural idempotence | None | Design the effect to be repeatable | Yes |
| Unique business constraint | None | Rely on an existing unique key | Yes |
| Client-side deduplication | Track outcomes locally | None | No |
| No retries at all | None | None | n/a — loses work instead |

Natural idempotence is worth reaching for first: setting a status is repeatable where appending a transaction is not, and an operation redesigned to be repeatable needs no key store, no retention policy and no reconciliation path.

> **Ask before choosing it**  
> What is the client's full retry behaviour, including after a crash or a reinstall? That determines both whether the key survives long enough to be reused and how long the server must retain it — and both answers are usually longer than the first estimate.

**How it fails**

**How idempotency keys go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicate charges anyway | Key regenerated per attempt | Generate once per operation and persist it |
| Two concurrent retries both execute | Read-then-write check | Unique constraint and an in-progress state |
| Client gets a different answer on retry | Server re-executed instead of replaying | Store and return the original response |
| Wrong result returned | Key reused for a different request | Bind a request hash to the key; reject mismatches |
| Duplicates return after a day | Retention shorter than client retry horizon | Retain beyond the longest realistic retry |
| Protection lost during an incident | Key store not durable across failover | Durable store for money-moving operations |
| Operation stuck in progress | Crash after the effect, before recording | Reconcile with the downstream system |

> **The crash between effect and record is the gap the pattern cannot close**  
> If the server charges the card and dies before writing the outcome, the key remains in progress and no amount of retrying resolves it — the truth lives at the payment processor, not in your database. This is why the same key must be passed downstream: it makes the processor the authority that can be queried, turning an unresolvable state into a reconciliation task.

**Where it is used**

- **Payment APIs**, where the Idempotency-Key header is an explicit part of the published contract.
- **Order submission**, where a double-tapped button and a retried request are the same problem.
- **Any HTTP POST behind a retrying client library**, since default retry policies are more aggressive than teams expect.
- **Webhook receivers**, where the sender retries and supplies a stable event identifier.
- **Mobile clients on poor networks**, where timeouts are routine and the key must survive app restarts.

**In the interview**

The signal is knowing where the guarantee stops, not describing the happy path.

- **Start from the ambiguous timeout** — request lost versus response lost — since that is what makes the pattern necessary.
- **Say the key is generated once per operation** and often has to be persisted client-side to survive a restart.
- **Use a unique constraint plus an in-progress state**, because concurrent retries are the case that read-then-write checks miss.
- **Insist the stored response is replayed verbatim**, including status codes; a fresh result is not idempotent behaviour.
- **Name the residual window** — crash after the effect, before recording — and resolve it by passing the key downstream and reconciling.

**Practice drill**

A payment endpoint times out for a mobile client that retries automatically and may also be restarted by the user. Design the idempotency scheme: where the key is generated, where it is persisted, the exact server-side sequence, and what a concurrent second attempt receives. Choose a retention period and justify it from client behaviour. Then identify the one failure the scheme cannot make safe, and describe the reconciliation process that resolves it.

**Go deeper**

An idempotency key lets a client retry a side-effecting request safely by giving the server a way to recognise repeats of one logical operation.

**It exists because a timeout is genuinely ambiguous.** A client that does not receive a response cannot distinguish a lost request from a lost reply, and no amount of client-side cleverness resolves that — the information is on the other side of the failure. Naming the operation moves the decision to the server, which does know, and converts an unanswerable question into a lookup.

**The key must identify the operation, not the attempt.** This is the most common implementation error: a key generated inside the retry loop makes every attempt distinct and the server executes each one. The identifier belongs to the user's intent — created when the button is pressed, reused unchanged by every retry, and frequently persisted locally so that it survives the client crashing and restarting mid-operation.

**Claiming must be atomic, because concurrent retries are normal.** A slow first attempt and an impatient second can arrive simultaneously, and a read-then-write check lets both through. Inserting the key under a unique constraint makes the database arbitrate, and an explicit in-progress state gives the loser a defined answer — wait and retry — rather than a wrong one.

**Replaying the original response is the behaviour, not merely recording that it happened.** The retry exists to learn the outcome of the first attempt, so returning a freshly computed result creates two answers for one operation and defeats the purpose. Storing the body and status code together, and returning them verbatim, is what makes the endpoint idempotent from the caller's perspective rather than just deduplicated internally.

**Binding the key to the request content catches a real class of bug.** A client that reuses a key for a different payload is malfunctioning, and answering it with the previous result quietly returns one user's outcome for another user's request. Storing a hash of the request alongside the key turns that into an explicit rejection, which is diagnosable, instead of a silent and confusing success.

**One window remains open, and honest designs name it.** A crash between performing the effect and recording it leaves the key in progress with the truth held only by the downstream system. Retrying cannot resolve this, because the server has no record of what it did — which is why the same key should travel onward to the payment processor or equivalent, making that system the authority that can be queried and the local record something that can be rebuilt rather than guessed.

**Related patterns:** Retry With Jitter · Inbox Deduplication · Transactional Outbox · Saga

---

### Retry With Jitter

*Retry failed calls with exponentially growing, randomised delays, so transient faults recover without the retries themselves becoming the outage.*

> **When you hear…** Transient network and dependency failures · retry storms observed during incidents · synchronised client behaviour

**Flow:** `Call fails` → `Wait backoff` → `Add randomness` → `Retry bounded times` → `Give up or degrade`

**The problem**

A dependency hiccups for two seconds. Every client retries immediately, so the instant it recovers it receives its normal load plus the entire backlog of retries, and falls over again. The retries have converted a two-second blip into a sustained outage.

Fixed-interval retries make it worse by synchronising. Clients that failed together retry together, producing regular spikes that keep re-breaking the service at exactly the moment it starts to recover.

> **Back off to give recovery room, randomise to avoid arriving together**  
> Exponential growth reduces pressure on a struggling dependency over time, which is what allows it to recover at all. Randomness spreads the retries that remain across the interval, so clients stop behaving as one synchronised crowd. Both are needed: backoff without jitter still produces coordinated waves, and jitter without backoff still applies full pressure.

**Mental model**

A bounded loop with a growing, randomised wait between attempts, applied only to failures that retrying could plausibly fix.

1. **Classify** — Decide whether the failure is transient at all; most are not.
2. **Wait** — Delay for an exponentially growing base interval.
3. **Randomise** — Spread the actual delay across that interval.
4. **Bound** — Cap both the number of attempts and the total elapsed time.
5. **Surrender** — Fail with a clear error, or degrade to a fallback.

> **Retries multiply at every layer that has them**  
> Three attempts in the client library, three in the service calling it, and three in the gateway in front produce twenty-seven requests for one user action. Each layer looks reasonable in isolation; together they amplify load by an order of magnitude precisely when the system is least able to take it. Retrying at one layer — usually the one closest to the failure — is the discipline that keeps this bounded.

**How it works**

**Backoff strategies compared**

```text
base = 100 ms, cap = 20 s, attempt n

NO JITTER (exponential only)
  delay = min(cap, base * 2^n)
  100, 200, 400, 800, 1600 ...
  -> every client retries at the SAME instants
  -> synchronised waves hit the recovering service

FULL JITTER          (recommended default)
  delay = random(0, min(cap, base * 2^n))
  -> retries spread uniformly across the window
  -> best load smoothing of the common options

DECORRELATED JITTER  (good for long outages)
  delay = min(cap, random(base, previous * 3))
  -> grows without synchronising, avoids collapsing
     to very small delays

BOUND TWO THINGS
  max attempts  (e.g. 5)
  max elapsed   (e.g. 30 s total)
  -> elapsed matters more; a user is not waiting
     through five exponential backoffs

RETRY BUDGET
  allow retries only while they are under ~10% of
  total requests
  -> when everything is failing, retries stop
     automatically instead of amplifying
```

1. **Retry only what is plausibly transient** — A 400 will fail identically forever; retrying it wastes capacity and delays the error.
2. **Use full jitter by default** — It smooths load better than fixed or partial jitter and is no harder to implement.
3. **Bound elapsed time, not just attempts** — The user's patience is a wall-clock budget, not a count.
4. **Retry at one layer only** — Nested retries multiply; pick the layer closest to the failure.
5. **Ensure the operation is idempotent first** — A retried write without an idempotency key duplicates the effect.
6. **Apply a retry budget** — Capping retries as a fraction of traffic stops amplification during a broad outage.

**What to retry, and what amplification costs**

```text
RETRY
  connection refused / reset
  timeout with no response
  429 too many requests   (honour Retry-After)
  503 service unavailable
  502 / 504 from a proxy

DO NOT RETRY
  400 bad request      deterministic
  401 / 403            will not change
  404 not found        will not change
  422 validation       deterministic
  409 conflict         usually needs new input

CAREFUL
  500 internal error   may be deterministic; retry
                       once, not five times
  timeout on a WRITE   the write may have succeeded
                       -> only retry with an
                          idempotency key

AMPLIFICATION ARITHMETIC
  3 layers x 3 attempts = 27x load
  dependency at 50% failure with 3 retries each
  -> effective load ~2x normal, while degraded
  -> this is how a partial failure becomes total
```

| Metric | Value | Note |
|---|---|---|
| Default strategy | full jitter | spread the load |
| Bound | elapsed time | **not just attempts** |
| Layering | retry once | not at every hop |
| Writes | need a key | or they duplicate |

> **Retrying a timed-out write duplicates it unless the operation is keyed**  
> A timeout means the outcome is unknown, and for a write the request may well have succeeded with only the response lost. Retrying it without an idempotency key performs the action twice. Read retries are safe by default; write retries are safe only when the receiver can recognise the repeat, which makes the two patterns dependent on each other rather than merely adjacent.

**Technologies**

| Layer | Mechanism | Note |
|---|---|---|
| Client libraries | Built-in retry policies | Often on by default; audit the settings |
| Service meshes | Retry policy per route | Centralised, but easy to stack with app retries |
| Cloud SDKs | Adaptive retry modes with budgets | Usually the best-tuned option available |
| Message consumers | Broker redelivery with backoff | Retry is inherent to the delivery model |
| Circuit breaker | Stops retries against a dead dependency | The necessary companion |
| Retry budget | Fraction-of-traffic cap | The most effective amplification control |

The circuit breaker line matters because retries and breakers solve opposite halves of the same problem: backoff handles a dependency that will recover in seconds, a breaker handles one that will not, and a system with only the first keeps hammering something that is already down.

**Trade-offs**

**Backoff strategies**

| Strategy | Load smoothing | Recovery speed | Use when |
|---|---|---|---|
| Immediate retry | None; amplifies | Fastest if it works | Almost never |
| Fixed interval | Poor; synchronises | Predictable | Simple internal calls |
| Exponential, no jitter | Poor; waves | Good | Rarely preferable to jitter |
| Exponential with full jitter | Best | Good | The default choice |
| Decorrelated jitter | Very good | Good over long outages | Sustained dependency failure |
| No retry, fail fast | Perfect | n/a | Non-idempotent, or a breaker is open |

Fail-fast belongs on the list as a legitimate option rather than an absence of one. Where the operation cannot be made idempotent and the caller can degrade gracefully, returning immediately is better than a retry policy that risks duplicating an effect.

> **Ask before choosing it**  
> How many other layers between this call and the user already retry? The honest answer is usually more than one, and the correct fix is often removing a retry rather than tuning the delays on all of them.

**How it fails**

**How retries go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Retry storm prolongs an outage | No backoff, or no jitter | Exponential with full jitter |
| Load amplified 27x | Retries stacked at three layers | Retry at one layer only |
| Duplicate writes | Retried a timed-out non-idempotent call | Idempotency key, or do not retry |
| User waits 30 seconds for an error | Attempts bounded but elapsed time not | Cap total elapsed time |
| Capacity wasted on doomed calls | Retrying 4xx responses | Classify errors before retrying |
| Rate limits hit harder | Ignoring Retry-After on 429 | Honour the server's stated delay |
| Dead dependency hammered indefinitely | Retries with no circuit breaker | Trip a breaker and stop trying |
| Thundering herd on recovery | Every client retrying at the same instant | Jitter is exactly this fix |

> **Retries turn partial failures into total ones**  
> A dependency failing half its requests receives roughly double its normal load once every failure is retried three times — at the moment it is least able to cope. The mechanism intended to ride out a degradation instead guarantees it deepens, which is why a retry budget that suppresses retries when failures are widespread is more valuable than any amount of delay tuning.

**Where it is used**

- **Cloud provider SDKs**, which ship adaptive retry modes with budgets precisely because of this failure mode.
- **Service meshes**, providing per-route retry policy that has to be reconciled with application-level retries.
- **Message consumers**, where redelivery with backoff is built into the delivery contract.
- **Mobile clients**, where transient connectivity makes retries essential and synchronised reconnection is a real hazard.
- **Any system with a published rate limit**, where honouring Retry-After is the difference between backing off and being blocked.

**In the interview**

Say why jitter exists, not just that it should be added — the synchronisation argument is what demonstrates understanding.

- **Explain the synchronised wave** that exponential backoff alone produces, and how randomisation breaks it.
- **Bound elapsed time as well as attempts**, since a user will not wait through five doublings for an eventual failure.
- **Raise multi-layer amplification unprompted** and argue for retrying at one layer rather than tuning three.
- **Tie write retries to idempotency keys.** A retried timeout on a non-idempotent write is a duplicate effect.
- **Pair retries with a circuit breaker**, because backoff handles seconds of failure and a breaker handles minutes of it.

**Practice drill**

A service calls a payment provider that intermittently times out. Design the retry policy: which status codes you retry, the backoff formula, the jitter strategy, and both bounds. Then compute the load amplification if the client library, your service and the gateway each retry three times, and say which layer you would remove. Finally, explain what must be true of the payment call before any retry is safe, and what you would do if it were not.

**Go deeper**

Retry with jitter recovers from transient failures by waiting longer between attempts and randomising when those attempts occur.

**The two halves solve different problems and neither is sufficient alone.** Exponential growth reduces the pressure a struggling dependency faces over time, giving it room to recover instead of being held down. Randomisation prevents the clients that failed together from returning together, which is what turns a recovering service back into a failing one. Backoff without jitter still produces coordinated waves; jitter without backoff still applies undiminished load.

**Retries amplify load exactly when the system can least absorb it.** A dependency failing half of its requests receives roughly double its normal traffic once each failure is retried, and layered retries compound multiplicatively — three attempts at three layers is twenty-seven requests for one user action. Every layer's policy looks reasonable in isolation, which is why the discipline is to retry at a single layer rather than to tune several.

**A retry budget is a stronger control than any delay schedule.** Capping retries at a small fraction of total traffic means that during a broad outage, when retrying cannot help anyway, the system stops generating them automatically. This scales the behaviour to conditions rather than to a configured guess, and it is the mechanism that distinguishes production-grade retry implementations from ones that merely delay.

**Classification precedes policy.** Most failures are not transient: a validation error, a missing resource, an authorisation failure will produce the identical result on every attempt. Retrying them consumes capacity, delays the error the caller needs to see, and adds latency to a request that was always going to fail — so deciding what is retryable is the first design step, not an optimisation applied afterwards.

**Write retries are only safe when the receiver can recognise them.** A timeout leaves the outcome unknown, and for a mutating call the operation may well have succeeded with only the response lost. Retrying then performs the effect twice, which makes retry policy and idempotency keys a single design rather than two independent ones — and a retry configuration applied to non-idempotent writes is a duplication mechanism, however well its delays are tuned.

**Bounding elapsed time matters more than bounding attempts, because a person is waiting.** Five attempts with exponential backoff can easily exceed thirty seconds, long after the user has given up or the upstream request has timed out anyway. A wall-clock budget makes the policy answerable to the experience it affects, and it pairs naturally with a circuit breaker: backoff is the right response to seconds of trouble, and something that stops trying altogether is the right response to minutes of it.

**Related patterns:** Circuit Breaker · Idempotency Key · Hedged Requests · Load Shedding

---

### Circuit Breaker

*Track failures against a dependency and stop calling it once they cross a threshold, failing fast until a probe shows it has recovered.*

> **When you hear…** A dependency is down and every call waits for a timeout · threads exhausted by a failing service · cascading failure

**Flow:** `Count failures` → `Threshold crossed` → `Open, fail fast` → `Wait, then probe` → `Close on success`

**The problem**

A recommendation service stops responding. Every request to the product page still calls it, waits three seconds for a timeout, and only then renders. Threads pile up waiting on a service that is definitely not going to answer, and the product page — which does not even need recommendations — becomes unavailable.

Retries make this worse rather than better. The dependency is not experiencing a transient blip; it is down, and each retry is another thread held for another timeout against something that cannot respond.

> **When a dependency is reliably failing, calling it is pure cost**  
> Past failures are evidence about the near future. Once a service has failed enough times in a row, the next call almost certainly fails too — so the correct action is to skip it and return immediately, spending nothing. Failing in one millisecond instead of three seconds is what keeps the caller alive, and it also removes the load that is preventing the dependency from recovering.

**Mental model**

A state machine per dependency with three states, driven by the observed failure rate.

1. **Closed** — Calls pass through; failures are counted.
2. **Open** — The threshold was crossed; calls fail immediately without being attempted.
3. **Half-open** — After a cooling period, a small number of probe calls are allowed.
4. **Close** — Probes succeed, normal traffic resumes.
5. **Re-open** — A probe fails, and the breaker returns to open for another interval.

> **An open breaker needs something to return**  
> Failing fast is only useful if the caller has an answer. A breaker without a fallback converts a slow error into a quick error, which helps the system's resource usage but not the user. Deciding what the product does when the dependency is unavailable — cached data, a default, a reduced page — is the part of the design that determines whether the breaker produces graceful degradation or just faster failure.

**How it works**

**The state machine and its thresholds**

```text
CLOSED
  every call executes
  record success / failure in a rolling window
  if failure rate > 50% over >= 20 requests in 10 s
    -> OPEN

OPEN
  every call returns immediately with the fallback
  no request reaches the dependency
  after 30 s
    -> HALF_OPEN

HALF_OPEN
  allow a few probes (e.g. 3 concurrent max)
  all succeed -> CLOSED
  any fails   -> OPEN, and lengthen the wait
                 (30 s, 60 s, 120 s ...)

WHY A MINIMUM REQUEST COUNT
  2 failures out of 2 is a 100% failure rate and
  means nothing
  -> without a volume floor, a low-traffic dependency
     trips on noise

WHY A ROLLING WINDOW
  a lifetime counter never recovers its statistics
  -> count only recent history

SLOW CALLS COUNT AS FAILURES
  a dependency answering in 8 s when its timeout is
  10 s is destroying you without failing
  -> treat p99 latency breaches as failures too
```

1. **Require a minimum request volume before tripping** — Otherwise low-traffic dependencies trip on two unlucky calls.
2. **Use a rolling window** — Cumulative counters never forget, so the breaker never reflects current conditions.
3. **Count slow calls as failures** — A dependency that responds just inside the timeout exhausts threads without ever erroring.
4. **Keep one breaker per dependency, not per process** — A shared breaker takes healthy dependencies down with the failing one.
5. **Limit concurrent probes in half-open** — A flood of probes re-breaks a service that has only just recovered.
6. **Define the fallback before the breaker** — Fast failure without an answer is only half a design.

**What it saves, and the fallback that makes it useful**

```text
WITHOUT A BREAKER
  recommendation service down
  200 req/s x 3 s timeout = 600 threads held
  -> thread pool of 200 exhausted in about 1 s
  -> the ENTIRE product page fails, not just
     recommendations

WITH A BREAKER
  after ~20 failures the breaker opens
  calls return in < 1 ms with cached or empty
  recommendations
  -> product page renders, minus one section
  -> thread pool healthy
  -> the dependency stops receiving load and can
     actually recover

FALLBACK LADDER
  1  cached value, even if stale
  2  a sensible default (popular items)
  3  omit the section entirely
  4  explicit error, only if the data is essential

THE DISTINCTION THAT MATTERS
  essential dependency  -> breaker gives a fast error
  optional dependency   -> breaker gives degradation
  -> most dependencies are more optional than the
     code currently assumes
```

| Metric | Value | Note |
|---|---|---|
| Failing call | 3 s → <1 ms | **threads freed** |
| Blast radius | one section | not the page |
| Recovery | probe then close | bounded |
| Requirement | a fallback | or it is just faster failure |

> **One breaker shared across dependencies takes down the healthy ones**  
> A breaker keyed on the process, or on a generic downstream label, trips because service A is failing and then refuses calls to services B and C as well. The breaker must be scoped to the thing that can independently fail — usually a service, sometimes a specific endpoint — so that an outage in one place stays in one place.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Library | Resilience4j, Polly, gobreaker | In-process, per-instance state |
| Service mesh | Envoy outlier detection | Applies without code changes; ejects bad hosts |
| Cloud load balancers | Health-based removal | Coarser, but the same principle |
| Bulkhead | Bounded pools per dependency | Complementary containment |
| Retry policy | Backoff with jitter | Handles transient; the breaker handles sustained |
| Fallback source | Cache, default, omission | The part that makes it user-visible progress |

Per-instance breaker state is worth noting: each process learns independently, so with fifty instances the dependency still receives fifty processes' worth of failures before all of them have tripped. Mesh-level outlier detection converges faster because the decision is made where traffic is routed.

**Trade-offs**

**Responses to a failing dependency**

| Approach | Threads held | Load on dependency | User sees |
|---|---|---|---|
| Wait for timeout every call | Many | Full | Slow failure |
| Retry with backoff | More | Higher | Slower failure |
| Circuit breaker, no fallback | Few | None while open | Fast failure |
| Circuit breaker with fallback | Few | None while open | Degraded but working |
| Bulkhead only | Bounded | Full | Partial failure, contained |
| Remove the dependency | None | None | Full function |

Bulkhead and breaker are frequently confused and are complementary: the bulkhead bounds how much of your capacity a dependency can consume, while the breaker decides when to stop calling it at all. Systems that survive dependency outages well usually have both.

> **Ask before choosing it**  
> What can this call return when the dependency is unavailable? If the honest answer is nothing useful, the breaker only makes the failure faster — which is still worth having for resource protection, but should be described as such rather than as graceful degradation.

**How it fails**

**How circuit breakers go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Breaker trips constantly on low traffic | No minimum request volume | Require a volume floor before evaluating |
| Breaker never opens | Cumulative counters diluted by history | Rolling window over recent requests |
| Threads exhausted despite the breaker | Slow calls not counted as failures | Treat latency breaches as failures |
| Healthy dependencies refused | One breaker shared across several | Scope per dependency |
| Recovering service knocked over again | Unlimited probes in half-open | Cap concurrent probes; lengthen backoff on failure |
| Users see errors instead of degradation | No fallback defined | Cache, default, or omit the section |
| Breaker flaps open and closed | Thresholds too close together | Separate trip and recovery criteria |
| Slow convergence across the fleet | Per-instance state with many instances | Mesh-level detection, or accept the lag |

> **A breaker on an essential dependency without a fallback just relocates the outage**  
> Opening the circuit on the database that the page fundamentally requires means every request now fails in one millisecond instead of three seconds. That genuinely protects the process, and it does nothing for the user. The value of the pattern is concentrated on dependencies that are optional in practice, and recognising which ones those are is usually a product conversation rather than an engineering one.

**Where it is used**

- **Product pages calling recommendation, review and inventory services**, where most are optional in practice.
- **Service meshes**, where outlier detection ejects failing hosts without any application change.
- **Third-party integrations**, whose availability is outside your control and whose failures must not propagate.
- **Mobile and edge clients**, which use breakers to avoid draining batteries retrying an unreachable endpoint.
- **Any fan-out request** where one slow backend would otherwise dictate the latency of the whole response.

**In the interview**

The differentiators are slow calls, the fallback, and scoping — the state machine itself is table stakes.

- **Give the three states with concrete thresholds** rather than describing them abstractly.
- **Say slow calls count as failures.** A dependency answering just inside its timeout exhausts threads without recording a single error.
- **Require a minimum volume**, or the breaker trips on noise whenever traffic is low.
- **Define the fallback explicitly** and distinguish optional from essential dependencies; that is where the pattern earns its value.
- **Note per-instance state** and that a large fleet converges slowly, which is why meshes do this at the routing layer.

**Practice drill**

A product page calls a recommendation service with a 3 second timeout, serving 200 requests per second with a 200-thread pool. Show how long the pool survives when the dependency stops responding. Design the breaker: thresholds, window, volume floor, open duration and probe policy. Choose the fallback and justify it. Then explain what changes in your answer if the failing dependency is the primary product database instead.

**Go deeper**

A circuit breaker stops calls to a dependency that is reliably failing, converting slow failures into immediate ones and removing load that prevents recovery.

**It treats recent failures as evidence rather than as isolated events.** A service that has failed its last twenty calls will almost certainly fail the next, so attempting it buys nothing and costs a held thread for the full timeout. Skipping the call is not pessimism; it is acting on the best available information, and the state machine exists to decide when that information is strong enough.

**Resource protection is the primary effect, and it is not symmetric.** Threads waiting on a dead dependency are the mechanism by which one service's outage becomes another's: two hundred threads held for three seconds each exhausts a pool in about a second, and the page fails entirely — including the parts that never needed the failing service. The breaker bounds that cost to a millisecond per call, which keeps the caller functioning regardless of what the callee is doing.

**Slow calls are the failure mode most implementations miss.** A dependency that responds in eight seconds against a ten-second timeout never registers an error, so a breaker configured only on exceptions stays closed while the thread pool drains. Counting latency breaches as failures closes this gap, and it matters because degraded-but-responding is far more common in practice than cleanly-down.

**The fallback is what turns the pattern from resource protection into user-visible resilience.** An open breaker must return something — a stale cached value, a sensible default, or an omitted section — and choosing that is a product decision. Without it, the user experiences a fast error instead of a slow one, which is a real improvement for the system and none at all for them. The corollary is that breakers pay off most on dependencies that are optional in practice, and identifying those honestly is where most of the value lies.

**Half-open is where breakers damage recovering services.** A dependency that has just come back cannot absorb the full suppressed load, so releasing all traffic at once re-breaks it and the cycle repeats. Limiting concurrent probes, and extending the open interval after each failed probe, lets the dependency come back gradually rather than being re-tested to destruction.

**Scoping and state distribution decide how the pattern behaves at scale.** A breaker keyed too broadly refuses healthy dependencies alongside the failing one; a breaker whose state is per-process means a fleet of fifty instances must each independently learn that a service is down, so the dependency still absorbs fifty processes' worth of failures during convergence. Moving the decision to the mesh or load balancer, where routing already happens, makes it both faster and shared — which is why infrastructure-level outlier detection has displaced library breakers in many environments.

**Related patterns:** Retry With Jitter · Bulkhead · Load Shedding · Hedged Requests

---

### Bulkhead

*Give each dependency or tenant its own bounded pool of resources, so one of them saturating cannot consume the capacity the others need.*

> **When you hear…** One slow dependency takes down unrelated features · a single tenant degrades everyone · shared thread or connection pools

**Flow:** `Shared pool` → `One dependency stalls` → `Pool exhausted` → `Partition pools` → `Failure contained`

**The problem**

A service has two hundred threads serving every endpoint. One downstream dependency becomes slow, and requests touching it hold threads for seconds at a time. Within a minute all two hundred threads are waiting on that one dependency, and endpoints that never call it are returning errors — they simply cannot get a thread.

The failure has nothing to do with the healthy endpoints. They were taken down by resource contention, because capacity was shared globally with no partition between what is failing and what is fine.

> **Compartments stop a leak from sinking the ship**  
> If each dependency can only ever consume a fixed share of threads or connections, then its failure is bounded by that share. Requests needing it queue or fail; requests that do not need it find the capacity they always had. The total capacity is not increased — it is fenced, so that failure stays where it started.

**Mental model**

Separate, bounded resource pools per dependency, tenant or class of work, sized so that the sum of the bounds does not exceed what the process can actually sustain.

1. **Identify** — Find the units that can independently fail — a dependency, a tenant, a request class.
2. **Partition** — Give each its own pool of threads, connections or concurrency permits.
3. **Bound** — Set a maximum for each, and a queue limit in front of it.
4. **Reject** — When a pool is full, fail that class quickly rather than borrowing from another.
5. **Observe** — Track saturation per pool, because that is where the early warning lives.

> **Partitioning reduces peak efficiency, and that is the point**  
> Separate pools cannot lend capacity to each other, so a burst on one path is rejected while another pool sits idle. That is strictly less efficient than a shared pool — and it is the property that stops one path consuming everything. Attempting to recover the efficiency by allowing borrowing rebuilds the shared pool and reintroduces exactly the failure the partition was meant to prevent.

**How it works**

**Partitioning strategies**

```text
BY DEPENDENCY  (most common)
  payments      max 50 concurrent
  recommender   max 20 concurrent
  search        max 30 concurrent
  -> recommender dying consumes at most 20

BY TENANT
  each tenant capped at N concurrent requests
  -> one customer's batch job cannot starve the rest
  -> essential for multi-tenant platforms

BY REQUEST CLASS
  interactive reads   70% of capacity
  background exports  20%
  admin operations    10%
  -> a report storm cannot delay page loads

SIZING
  sum of pools <= what the process can sustain
  50 + 20 + 30 = 100 concurrent, and the process
  must be able to run 100 concurrently

  per-pool size = arrival rate x latency + headroom
    payments: 100 req/s x 0.3 s = 30, so 50 is
    reasonable headroom

QUEUE IN FRONT OF EACH POOL, BOUNDED
  unbounded queue = memory exhaustion, and latency
  that grows without limit
  -> small queue, then reject
```

1. **Partition by what can independently fail** — A pool per dependency is the default; per tenant where isolation is contractual.
2. **Keep the sum of pools within real capacity** — Partitions are meaningless if their total exceeds what the process can run.
3. **Bound the queue in front of each pool** — An unbounded queue converts a concurrency limit into a memory leak.
4. **Reject rather than borrow** — Borrowing capacity across pools rebuilds the shared pool and the original failure.
5. **Isolate the most dangerous dependency first** — Full partitioning everywhere is rarely worth the operational complexity.
6. **Monitor saturation per pool** — A pool consistently at its limit is the early signal of the outage it will later cause.

**With and without partitions**

```text
SHARED POOL, 200 threads
  normal:  payments 30, recommender 20, search 25
           used 75 of 200, healthy
  recommender degrades to 5 s responses
           recommender requests: 40/s x 5 s = 200
           threads held
  -> all 200 consumed
  -> payments and search get NOTHING
  -> total outage from one optional dependency

PARTITIONED  50 / 20 / 30
  recommender degrades identically
  -> it saturates its 20 and rejects the rest
  -> payments still has 50, search still has 30
  -> recommendations fail; the rest of the site works

COST OF PARTITIONING
  a burst of 60 payment requests cannot borrow
  recommender's idle 20
  -> 10 rejected that a shared pool would have served
  -> that is the premium paid for containment

THREAD POOLS VS SEMAPHORES
  thread pool  isolates latency, real context switch,
               higher cost
  semaphore    just a counter, cheap, but the calling
               thread still blocks
  -> semaphores for in-process concurrency limits,
     separate pools where you must not block
```

| Metric | Value | Note |
|---|---|---|
| Shared pool | one failure | **total outage** |
| Partitioned | one failure | one feature |
| Cost | lower peak use | the premium |
| Signal | pool saturation | early warning |

> **An unbounded queue in front of a bounded pool undoes the bulkhead**  
> Limiting concurrency to twenty while letting an unlimited number of requests wait for one of those twenty slots means memory grows without bound and latency climbs until every waiting request has already timed out upstream. The pool is protected and the process is not. The queue must be small enough that waiting is still useful, with rejection beyond it.

**Technologies**

| Mechanism | Where | Note |
|---|---|---|
| Separate thread pools | JVM and similar runtimes | Strong isolation, real overhead per pool |
| Semaphores and concurrency limits | Any runtime | Cheap; the caller still blocks |
| Separate connection pools | Per database or per service | Prevents one dependency exhausting connections |
| Separate deployments | Per tenant or per workload class | The strongest isolation available |
| Mesh-level concurrency limits | Envoy and similar | Applied without code changes |
| Circuit breaker | Alongside | Bounds the damage; the breaker stops the calls |

Separate deployments are the extreme form and genuinely used: a dedicated fleet for a large tenant or for background processing isolates failure completely, at the cost of running and operating more things. Most systems stop well short, but the option is worth naming because it is the same idea taken to its limit.

**Trade-offs**

**Isolation strategies**

| Approach | Isolation | Efficiency | Complexity |
|---|---|---|---|
| Shared pool | None | Highest | Lowest |
| Semaphore per dependency | Concurrency only | High | Low |
| Thread pool per dependency | Concurrency and latency | Medium | Medium |
| Connection pool per dependency | Connection exhaustion | High | Low |
| Separate process or fleet | Complete | Lowest | Highest |
| Separate cluster per tenant | Complete, contractual | Lowest | Highest |

Semaphores are underrated in this table: a counter per dependency costs almost nothing, requires no extra threads, and prevents the most common form of exhaustion. Reaching for dedicated thread pools first is a frequent over-correction.

> **Ask before choosing it**  
> Which single dependency, if it became slow rather than failing outright, would consume the most capacity? Partition that one first. Full partitioning of every call path multiplies configuration and is rarely justified by the risk it removes.

**How it fails**

**How bulkheads go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Pools sum to more than real capacity | Sized independently without a total | Constrain the sum to sustainable concurrency |
| Memory grows during a dependency outage | Unbounded queues in front of pools | Bound the queue; reject beyond it |
| Isolation silently lost | Borrowing allowed between pools | Reject rather than borrow |
| Everything still fails together | Partitioned by endpoint, not by dependency | Partition by what can independently fail |
| Rejections during normal traffic | Pools sized below actual demand | Size from arrival rate times latency, plus headroom |
| Operational overhead outweighs benefit | Every call path partitioned | Isolate the riskiest few |
| Outage with no warning | Pool saturation not monitored | Alert on sustained saturation per pool |

> **Sizing pools independently silently rebuilds the shared pool**  
> Four pools of a hundred in a process that can sustain two hundred concurrent operations provides no isolation at all: two saturated pools consume everything, and the other two find nothing left. The partition only exists if the sum respects the real limit, which means pool sizes are a single allocation decision rather than several local ones.

**Where it is used**

- **Multi-tenant platforms**, where per-tenant limits are often a contractual guarantee rather than a tuning choice.
- **Gateways and aggregators**, which fan out to many backends and must not let one slow backend hold every thread.
- **Services with mixed interactive and batch work**, where a report storm would otherwise delay page loads.
- **Database connection pools split per workload**, preventing analytics queries from exhausting transactional connections.
- **Service meshes**, which apply per-upstream concurrency limits without touching application code.

**In the interview**

Distinguish it clearly from the circuit breaker and be specific about what a pool protects.

- **Draw the shared-pool failure first**: one slow dependency holding every thread while unrelated endpoints fail.
- **Partition by what can independently fail**, which is usually a dependency or a tenant — not by endpoint.
- **Constrain the sum of pool sizes** to real capacity, since independently sized pools provide no isolation.
- **Bound the queue.** A concurrency limit with an unlimited waiting room protects the pool and not the process.
- **Contrast with the circuit breaker**: the bulkhead bounds the damage, the breaker stops the calls, and mature systems use both.

**Practice drill**

A service with a 200-thread pool calls payments, recommendations and search. Show how many threads the recommendation path consumes when it degrades from 100 ms to 5 seconds at 40 requests per second, and what happens to the other two. Design the partitions: sizes, queue bounds, and the rejection behaviour. Justify each size from arrival rate and latency, and verify the sum against the process capacity. Then say which one you would isolate first if you could only do one.

**Go deeper**

A bulkhead partitions capacity so that one dependency, tenant or workload cannot consume the resources the others depend on.

**It addresses a failure that has nothing to do with the healthy paths.** Endpoints that never touch a failing dependency go down anyway, because the threads they need are all blocked waiting on it. Recognising that resource contention is the transmission mechanism — not logical coupling — is what makes partitioning the right remedy rather than more capacity or better timeouts.

**Slow is more dangerous than dead, and the bulkhead is aimed at slow.** A dependency that fails instantly returns threads immediately; one that responds in five seconds holds them, and concurrency demand grows as arrival rate times latency. A path consuming twenty threads at normal speed consumes two hundred at fifty times the latency, which is how a single degraded optional feature exhausts an entire pool without ever producing an error.

**The efficiency loss is the mechanism, not a side effect.** Pools that cannot lend to each other will reject requests while other pools sit idle, and that is precisely what prevents one path from taking everything. Adding borrowing to recover utilisation recreates the shared pool and, with it, the original failure — so the premium in rejected peak traffic is what containment actually costs and should be accepted deliberately.

**Sizing is one allocation decision, not several local ones.** Pools sized independently commonly sum to far more than the process can sustain, at which point two saturated partitions consume everything and the isolation exists only on paper. Each pool should be derived from its path's arrival rate and latency with headroom, and the total must be checked against real capacity — the constraint that makes the partition meaningful.

**The queue in front of each pool needs its own bound.** A concurrency limit with an unlimited waiting room protects the dependency and lets memory and latency grow without limit inside the caller, until every queued request has already timed out upstream. A short queue followed by rejection keeps waiting useful and keeps the failure fast, which is what the caller needed from the partition in the first place.

**It is complementary to the circuit breaker, and the pair is what actually survives outages.** The bulkhead bounds how much damage a failing dependency can do while it is still being called; the breaker decides to stop calling it. With only a bulkhead, the isolated pool stays permanently saturated against a dead service; with only a breaker, the pool is still fully exhausted during the window before it trips. Saturation per pool is also the best leading indicator either provides — a partition sitting at its limit is describing the outage it is about to cause.

**Related patterns:** Circuit Breaker · Load Shedding · Backpressure · Retry With Jitter

---

### Hedged Requests

*Send a second copy of a request to another replica once the first is slower than usual, and take whichever answer arrives first, cutting tail latency.*

> **When you hear…** Tail latency far worse than median · occasional slow replicas · latency matters more than a small load increase

**Flow:** `Send to replica A` → `Wait p95` → `No answer yet` → `Send to replica B` → `First response wins`

**The problem**

A service answers in 10 milliseconds at the median and 400 at the 99th percentile. The slow responses are not caused by expensive work — they are a garbage collection pause, a noisy neighbour, a momentarily unlucky disk. The same request sent to a different replica would have returned in 10 milliseconds.

Fan-out makes this dominate. A request that queries 100 shards and waits for all of them has a 99th-percentile experience on roughly two-thirds of requests, because the probability that at least one shard is slow approaches certainty as the fan-out grows.

> **Slowness is usually a property of the replica, not the request**  
> If the delay comes from transient local conditions rather than from the work itself, then asking a second replica is likely to succeed quickly while the first is still stalled. Issuing that second request only after the first has already exceeded its normal time means the extra load is confined to the small fraction of requests that were going to be slow anyway.

**Mental model**

A timer set to the usual latency. If the answer has not arrived by then, a duplicate goes to another replica and the first response wins.

1. **Send** — The request goes to one replica as normal.
2. **Wait** — A hedging delay, typically around the 95th percentile of normal latency.
3. **Hedge** — If no answer has arrived, an identical request goes to a different replica.
4. **Race** — Whichever response returns first is used.
5. **Cancel** — The loser is cancelled if the protocol allows, to avoid paying for work nobody needs.

> **Hedging a genuinely overloaded system accelerates its collapse**  
> The pattern assumes slow responses are local anomalies against a system with spare capacity. When the whole service is saturated, every request exceeds the hedging delay, every request is duplicated, and the load nearly doubles precisely when it is already too high. Hedging must be disabled automatically under high load, or it converts a degradation into an outage.

**How it works**

**The hedge, and what the delay costs**

```text
CHOOSING THE DELAY
  latency profile: p50 10 ms, p95 50 ms, p99 400 ms

  hedge at p95 = 50 ms
  -> only 5% of requests ever hedge
  -> extra load: 5%
  -> p99 becomes about 50 ms + p50 of the second
     replica = ~60 ms, down from 400 ms

  hedge at p50 = 10 ms
  -> 50% of requests hedge
  -> extra load: 50%
  -> better tail, much higher cost

  hedge at p99 = 400 ms
  -> 1% hedge, negligible cost
  -> but the tail is already 400 ms; too late to help

THE RULE
  hedge delay = the percentile of requests you are
  willing to duplicate
  p95 is the usual balance: 5% more load, tail cut
  by roughly an order of magnitude

CAP THE HEDGES
  one hedge, not a cascade
  -> three copies of every slow request is a load
     multiplier, not a latency fix
```

1. **Set the delay at a high percentile of observed latency** — The delay directly determines the fraction of requests duplicated.
2. **Send the hedge to a different replica** — Hedging to the same instance repeats whatever is making it slow.
3. **Cancel the loser where the protocol supports it** — Otherwise the work is done twice and paid for twice.
4. **Only hedge idempotent operations** — A duplicated write executes twice unless the receiver deduplicates it.
5. **Cap hedging as a fraction of total requests** — A budget stops the pattern amplifying load during an overload.
6. **Disable hedging automatically under high utilisation** — Spare capacity is the assumption the whole pattern rests on.

**Why fan-out makes this necessary**

```text
ONE SERVICE, p99 = 400 ms
  1% of requests are slow. Tolerable.

FAN-OUT TO 100 SHARDS, wait for all
  P(all fast) = 0.99^100 = 0.366
  -> 63% of requests hit at least one slow shard
  -> the p99 of a component becomes the median of
     the system

THIS IS THE TAIL-AT-SCALE PROBLEM
  as fan-out grows, rare slowness becomes the common
  case, and improving the median of each component
  does nothing about it

WITH HEDGING AT p95 PER SHARD
  each shard's effective p99 drops from 400 ms to
  about 60 ms
  -> P(all fast) rises dramatically
  -> the composed request becomes predictable

WHEN IT DOES NOT APPLY
  no replicas to hedge to
  non-idempotent operations
  the work itself is expensive (the slowness is real,
  not incidental)
  system running near capacity
```

| Metric | Value | Note |
|---|---|---|
| Hedge delay | p95 | **5% duplicated** |
| Tail effect | 400 ms → 60 ms | roughly 10× |
| Requirement | spare capacity | and replicas |
| Safety | idempotent only | or dedupe |

> **Hedging without a budget turns overload into collapse**  
> If load rises and every request starts exceeding the hedge delay, the mechanism doubles the traffic automatically, which pushes latency higher, which causes more hedging. The feedback is positive and fast. A budget that caps hedges at a small percentage of requests breaks the loop, and it is not optional in any system that can be overloaded.

**Technologies**

| Context | Support | Note |
|---|---|---|
| gRPC | Hedging policy in the service config | Configurable delay and attempt cap |
| Service meshes | Request hedging per route | No application change required |
| Storage systems | Speculative reads across replicas | Common internally in distributed stores |
| MapReduce-style systems | Backup tasks for stragglers | The same idea at batch granularity |
| Tied requests | Send both, cancel on start | Lower waste than hedging, needs protocol support |
| Request to two, always | Simplest form | Doubles load unconditionally; rarely justified |

Tied requests are the refinement worth knowing: both copies are sent immediately but each carries the identity of the other, and whichever starts executing first cancels its twin. This gets most of the latency benefit with far less duplicated work, at the cost of needing cooperation between the servers.

**Trade-offs**

**Ways to attack tail latency**

| Approach | Extra load | Tail improvement | Needs |
|---|---|---|---|
| Nothing | None | None | n/a |
| Hedge at p95 | About 5% | Large | Replicas, idempotence, spare capacity |
| Hedge at p50 | About 50% | Larger | Substantial spare capacity |
| Always send two | 100% | Largest | Double the capacity |
| Tied requests | Small | Large | Server-side cancellation support |
| Fix the underlying cause | None | Permanent | Time, and a findable cause |

The last row is the one to say out loud: hedging is a mitigation, not a cure. If the tail is caused by a specific fixable problem — an unbalanced shard, a garbage collection configuration, a single slow disk — fixing it is better than paying a permanent load premium to route around it.

> **Ask before choosing it**  
> Is the slowness incidental or intrinsic? Hedging helps when a different replica would have been fast and does nothing when the request is genuinely expensive everywhere — in that case it doubles the cost of the most expensive requests in the system for no benefit at all.

**How it fails**

**How hedging goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Load doubles during an overload | No hedging budget | Cap hedges as a fraction of requests; disable under load |
| Duplicate side effects | Hedged a non-idempotent operation | Hedge reads only, or require idempotency keys |
| No improvement at all | Slowness is intrinsic to the request | Diagnose the cause; hedging cannot help |
| Wasted capacity | Losing request never cancelled | Cancel on first response where supported |
| Hedge is as slow as the original | Sent to the same overloaded replica | Route the hedge to a different replica |
| Marginal benefit, high cost | Delay set too low | Set the delay from the observed percentile |
| Cascading duplicates | Hedging at several layers | Hedge once, at one layer |

> **Hedging is a positive feedback loop when capacity is tight**  
> Higher latency produces more hedges, more hedges produce more load, more load produces higher latency. In a system with spare capacity the loop is damped and the mechanism works beautifully; in a system near its limit it is self-reinforcing and fast. The budget that caps hedging is therefore a stability control rather than a cost optimisation, and it is the first thing to verify in any implementation.

**Where it is used**

- **Large-scale search and fan-out queries**, where one slow shard would otherwise dictate the whole response time.
- **Distributed storage engines**, which issue speculative reads to replicas as a standard internal technique.
- **Batch processing frameworks**, which launch backup tasks for stragglers — the same idea at a much coarser grain.
- **gRPC and mesh-based services**, where hedging policy is configuration rather than code.
- **Latency-sensitive read paths** sitting in front of replicated stores, where every replica can answer identically.

**In the interview**

The strong answer connects hedging to fan-out arithmetic and volunteers the overload hazard.

- **Do the fan-out maths.** A 1% component tail becomes a 63% system tail across 100 shards, and that is why the pattern exists.
- **Tie the delay to a percentile** and state the load consequence directly: hedging at p95 duplicates 5% of requests.
- **Require idempotence**, since a hedged write executes twice unless the receiver can recognise it.
- **Raise the overload feedback loop unprompted** and give the budget as the control that prevents it.
- **Say it is a mitigation** — if the tail has a findable cause, fixing it beats paying a permanent load premium.

**Practice drill**

A shard responds in 10 ms at p50, 50 ms at p95 and 400 ms at p99, and a query fans out to 100 shards waiting for all of them. Compute the probability that a query touches at least one slow shard. Choose a hedging delay, state the extra load it causes and the resulting p99 per shard. Then describe the budget you would apply and the condition under which hedging disables itself, and explain why that condition is necessary rather than merely prudent.

**Go deeper**

Hedged requests reduce tail latency by sending a duplicate to another replica once the first response is later than usual, and using whichever arrives first.

**It rests on slowness being a property of the replica rather than the request.** Garbage collection pauses, contended disks and noisy neighbours are local and transient, so an identical request elsewhere is likely to complete promptly. Where the delay is intrinsic — a genuinely expensive query — a second copy is equally slow, and the pattern adds cost while improving nothing, which is why diagnosing the source of the tail comes before deciding to hedge.

**Fan-out is what makes it necessary rather than merely nice.** Component tails compose brutally: a 1% chance of slowness per shard becomes a 63% chance across a hundred shards, so the rare case at the component becomes the common case at the system. Improving each component's median does nothing here; only compressing the tail changes the composed result, which is why large fan-out systems treat speculative execution as routine.

**The delay is a direct control on cost.** Hedging at the 95th percentile duplicates 5% of requests, at the median it duplicates half, and at the 99th it barely triggers in time to help. That makes the parameter a straightforward trade between extra load and tail improvement, tunable from the observed latency profile rather than guessed — and it is the reason hedging can deliver an order-of-magnitude tail reduction for a single-digit load increase.

**Spare capacity is the load-bearing assumption.** The extra requests have to go somewhere, and in a system running near its limit they push latency up, which triggers more hedges, which pushes latency further. The loop is positive and quick, so a budget capping hedges as a fraction of traffic is a stability requirement rather than an efficiency measure — without it, the mechanism that smooths the tail in normal conditions accelerates the collapse in bad ones.

**Idempotence bounds where it can be applied.** A duplicated read is free of consequences; a duplicated write happens twice unless the receiver recognises the repeat. In practice this confines hedging to read paths, or pairs it with idempotency keys — and the pairing is worth stating explicitly, because a hedging policy configured at the mesh applies to every route it matches, including ones the author was not thinking about.

**Cancellation is what separates a tidy implementation from a wasteful one.** Without it both copies run to completion and the system pays twice for every hedged request, which roughly doubles the cost of the slowest requests — the expensive ones. Tied requests refine this further by dispatching both immediately and letting whichever starts first cancel its twin, capturing most of the latency benefit with far less duplicated work, at the price of needing servers that cooperate.

**Related patterns:** Retry With Jitter · Circuit Breaker · Scatter Gather · Load Shedding

---

## Coordination

### Leader Election

*Have a group of identical processes agree on one of them to perform singleton work, with automatic replacement when that one fails.*

> **When you hear…** Work that must happen exactly once across a fleet · a coordinator role that cannot be duplicated · a singleton that must survive its host dying

**Flow:** `Candidates compete` → `Consensus picks one` → `Leader holds term` → `Heartbeats renew` → `Failure triggers re-election`

**The problem**

A nightly reconciliation job runs on every instance of a service, so with twelve instances it runs twelve times and produces twelve sets of adjustments. Pinning it to one named host fixes the duplication and creates a new problem: when that host is replaced during a deploy, the job simply stops happening and nobody notices for a week.

The requirement is genuinely contradictory on its face — exactly one process must do the work, and no specific process can be required to survive. Something has to decide which one, and that decision must itself survive failures.

> **Make the singleton a role, not a machine**  
> If the group can agree on which member currently holds a role, then the role survives any individual member. The hard part is the agreement: two processes both believing they hold it is far worse than nobody holding it, so the mechanism has to make split-brain impossible rather than unlikely.

**Mental model**

A contested, time-bounded role granted by a consensus system. Holding it requires continuous renewal; losing contact means losing it.

1. **Campaign** — Candidates attempt to acquire the role through a store that can decide atomically.
2. **Grant** — Exactly one wins, and receives a term identifier that increases with every election.
3. **Renew** — The leader repeatedly refreshes its claim within the lease period.
4. **Expire** — A leader that stops renewing loses the role automatically.
5. **Re-elect** — The remaining members compete again, under a higher term.

> **The leader can be deposed without knowing it**  
> A process that is paused, partitioned or simply slow may believe it is still the leader long after its lease expired and a successor was elected. It cannot detect this locally — from inside, a network partition looks exactly like a quiet period. Every design must assume the previous leader is still running and still acting, which is why the term number has to be checked by whatever the leader writes to.

**How it works**

**Terms, fencing and the split-brain window**

```text
ELECTION
  candidates attempt an atomic acquire against a
  consensus store (etcd, ZooKeeper, Consul)
  winner receives:
    role      = leader
    term      = 7          (monotonically increasing)
    lease     = 10 s

RENEWAL
  leader refreshes every 3 s
  miss the renewal window -> lease expires
  -> a new election begins, term becomes 8

THE DANGEROUS SEQUENCE
  leader (term 7) pauses for 15 s (GC, host stall)
  lease expires, new leader elected at term 8
  old leader resumes, still believes it is leader
  -> two leaders exist simultaneously
  -> this is UNAVOIDABLE; it is detected, not prevented

FENCING MAKES IT HARMLESS
  every write carries the term
  the resource records the highest term it has seen
    write(term = 8) -> accepted, highest = 8
    write(term = 7) -> REJECTED, stale term
  -> the deposed leader cannot corrupt anything

WITHOUT FENCING
  the old leader's writes land after the new
  leader's, silently overwriting them
  -> the election was correct and the data is wrong
```

1. **Use a real consensus system for the election** — Leadership decided by a single database row inherits that database's failure modes as split-brain.
2. **Give every term a monotonically increasing number** — It is the only thing that lets a resource distinguish current from deposed.
3. **Fence every side effect with the term** — Election alone does not prevent a stale leader from writing.
4. **Set the lease from realistic pause durations** — A lease shorter than a garbage collection pause produces constant spurious elections.
5. **Make the leader's work resumable** — A successor inherits partially completed work and must be able to continue or restart it cleanly.
6. **Expect leaderless intervals** — Between expiry and election nothing is running; the design must tolerate that gap.

**Choosing the lease, and what it costs**

```text
LEASE TOO SHORT  (2 s)
  a 3 s GC pause deposes a healthy leader
  -> elections every few minutes
  -> work restarted constantly, nothing completes

LEASE TOO LONG  (60 s)
  a crashed leader is not replaced for a minute
  -> the singleton work stalls for that long

CHOOSING
  lease > worst realistic pause (GC, disk stall,
  network blip) x a safety factor
  typical: 10-15 s lease, renew every 3-5 s
  -> detection of a dead leader takes up to one lease

AVAILABILITY OF THE ROLE
  failover time = lease remaining + election time
  10 s lease -> up to ~12 s with no leader
  -> if the work cannot pause for 12 s, leader
     election is the wrong pattern

WHAT LEADER ELECTION IS NOT
  it is not a throughput technique; the leader is a
  bottleneck by construction
  -> use it for coordination, not for data paths
```

| Metric | Value | Note |
|---|---|---|
| Guarantee | one leader | **with fencing** |
| Failover | lease + election | ~12 s typical |
| Split brain | possible | made harmless |
| Throughput | one process | not a scaling tool |

> **Election without fencing solves the wrong half of the problem**  
> Choosing a leader correctly is straightforward; preventing the previous one from acting is the difficult part. A system that elects through a robust consensus store and then writes to a database without checking terms has a correct election and a corruptible resource, because a paused leader will eventually wake and write. The fencing token is not a refinement — it is where the guarantee actually lives.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| etcd, ZooKeeper, Consul | Purpose-built; leases and watches included | Another distributed system to operate |
| Kubernetes lease objects | Already present in the cluster | Tied to cluster availability |
| Database row with a lease column | No new infrastructure | Split-brain if the database fails over |
| Kafka consumer group leadership | Reuses existing coordination | Semantics tied to partitions |
| Cloud-managed locks | Fully operated | Latency, and service-specific limits |
| Avoid it: partition the work | No coordination at all | Only where work can be split by key |

The last row deserves consideration first. Work that can be partitioned by key and assigned by consistent hashing needs no leader at all, and removes the failover gap along with the coordination system — which is often a better answer than electing a leader well.

**Trade-offs**

**Ways to run singleton work**

| Approach | Survives host failure | Split-brain risk | Cost |
|---|---|---|---|
| Pin to one host | No | None | None |
| Run everywhere with idempotent effects | Yes | n/a | Requires idempotent design |
| Partition work by key | Yes | None | Needs a partitionable workload |
| Leader election with fencing | Yes | Handled | Consensus system, failover gap |
| Leader election without fencing | Yes | Real | Looks correct, is not |
| External scheduler triggering one run | Yes | Low | Depends on the scheduler |

Running everywhere with idempotent effects is underused: if the work can be made naturally repeatable, twelve instances doing it produces the same outcome as one, and no coordination is needed at all.

> **Ask before choosing it**  
> Can the work be partitioned or made idempotent instead? Leader election adds a consensus dependency and a failover window to every operation that needs the role, and both alternatives remove the problem rather than managing it.

**How it fails**

**How leader election goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Two leaders write conflicting data | No fencing token on writes | Carry the term; reject stale terms at the resource |
| Constant re-elections | Lease shorter than realistic pauses | Size the lease above worst-case stalls |
| Work never completes | Every election restarts it from the beginning | Make the work checkpointed and resumable |
| Split-brain from the election store | Leadership held in a non-consensus store | Use a system designed for this |
| Singleton work stops silently | Leader crashed, nobody monitoring the role | Alert on leaderless duration and election rate |
| Leader becomes a bottleneck | Data path routed through it | Use it for coordination only |
| Failover too slow for the workload | Lease duration exceeds tolerance | Shorter lease, or a different pattern |

> **The deposed leader is not a rare edge case**  
> Long garbage collection pauses, host-level stalls and brief partitions are ordinary operational events, and each one can produce a leader that is still running and still convinced. Any design that treats this as improbable will encounter it, typically under load when the pauses are longest — which is also when the work being duplicated matters most.

**Where it is used**

- **Scheduled jobs across a fleet**, the most common application and the one that motivates it.
- **Kubernetes controllers**, which use lease objects so that exactly one replica reconciles at a time.
- **Database primaries**, where election decides which replica accepts writes and fencing prevents the old one from continuing.
- **Partition assignment in messaging systems**, where a coordinator distributes work among consumers.
- **Cluster management planes**, which elect a controller to make cluster-wide decisions.

**In the interview**

The answer that stands out is about the deposed leader, not about how the election works.

- **Say that split-brain is detected, not prevented** — a paused leader cannot know it was deposed — and build the design around that.
- **Introduce fencing tokens** and show the resource rejecting a stale term; this is where the guarantee actually lives.
- **Size the lease against real pauses** and state the failover window it implies, since that is what the workload must tolerate.
- **Make the work resumable**, because every failover hands a successor something half-finished.
- **Offer partitioning or idempotence as alternatives** before electing anything; removing the singleton is better than coordinating it.

**Practice drill**

A nightly reconciliation job must run exactly once across twelve instances. Design leader election: the store, the lease duration, the renewal interval, and the failover window it produces. Then trace a 15-second garbage collection pause on the leader: what happens, who is leader afterwards, and what the paused process does when it resumes. Show the fencing mechanism that makes its late writes harmless, and say what the job must do to resume safely after a mid-run failover.

**Go deeper**

Leader election lets a group of identical processes agree on one member to perform work that must happen exactly once, and to replace it automatically when it fails.

**It converts a machine into a role, which is what makes the singleton survivable.** Pinning work to a named host is correct until that host is replaced; electing a holder means the work continues across restarts, deploys and hardware failures without anyone intervening. The cost is a dependency on a coordination system and a window during which no one holds the role.

**The difficult part is not choosing, it is unchoosing.** A process that is paused by garbage collection or cut off by a partition cannot distinguish that from a quiet period, so it continues believing it is the leader while a successor is already operating. No protocol prevents this, because the deposed process has no way to learn the truth in time — which means the design must assume two leaders can exist simultaneously and make that harmless.

**Fencing tokens are where the guarantee lives.** Each term carries a number that only increases, every write carries its term, and the resource remembers the highest it has accepted. A stale leader's writes are then rejected by the thing being written to, rather than being prevented by the election. Systems that elect carefully and then write without fencing have built the easy half of the pattern and left the failure mode intact.

**Lease duration is a direct trade between false failovers and slow ones.** Short leases depose healthy leaders whenever a pause exceeds them, producing churn in which work restarts continually and nothing finishes; long leases leave a crashed leader unreplaced for their full duration. The lease must exceed realistic worst-case stalls, and the resulting failover window — lease remainder plus election time — is the interruption the workload has to tolerate.

**Every failover inherits partial work, so resumability is part of the design.** A successor takes over mid-run and must be able to determine what has already been done, which means checkpointing progress durably rather than holding it in memory. Without that, each election restarts from the beginning, and a workload longer than the interval between failures never completes at all.

**It is a coordination mechanism and a throughput bottleneck.** Routing a data path through the leader means all traffic passes through one process that can be deposed at any moment, which is both slower and more fragile than the alternatives. The better instinct is to avoid needing a leader — partitioning work by key or making effects idempotent removes the singleton entirely, along with the consensus dependency and the failover gap that come with it.

**Related patterns:** Distributed Lock · Leases · Heartbeat Failure Detection · Consistent Hashing

---

### Distributed Lock

*Grant exclusive access to a resource across processes using a shared store, with an expiry so a crashed holder cannot block it forever.*

> **When you hear…** Two processes must not act on the same resource simultaneously · a shared external resource with no locking of its own · coordination across machines

**Flow:** `Acquire with TTL` → `Holder does work` → `Renew or expire` → `Release explicitly` → `Fencing token on writes`

**The problem**

Two workers pick up the same account and both apply a balance adjustment. In one process a mutex would prevent this trivially; across processes there is no shared memory and no scheduler, so nothing stops them.

The obvious fix — a lock row in a shared store — introduces a worse failure than the one it solves: a worker that crashes while holding the lock blocks the resource permanently, and a worker that is merely slow cannot be distinguished from one that has died.

> **A distributed lock must expire, and expiry is what breaks mutual exclusion**  
> Because a holder can die silently, every distributed lock needs a time limit — and the moment a lock can expire, the holder can still be working when it does. Mutual exclusion is therefore not guaranteed by the lock itself; it is guaranteed by the resource rejecting writes from an expired holder. The lock reduces contention, and fencing provides the correctness.

**Mental model**

A key in a shared store, set conditionally with a time to live, owned by whoever set it and released only by them.

1. **Acquire** — Set the key only if absent, with an expiry and a unique owner identifier.
2. **Work** — Perform the exclusive operation, ideally briefly.
3. **Renew** — Extend the expiry if the work runs longer than expected.
4. **Release** — Delete the key, but only if still the owner.
5. **Fence** — Carry a monotonic token so late writes from a lapsed holder are rejected.

> **Releasing a lock you no longer hold releases someone else's**  
> If the holder's lease expired and another process acquired the lock, a plain delete removes the new owner's lock and lets a third process in. Release must be conditional on ownership — compare the owner identifier and delete atomically — or the lock actively causes the concurrency it was meant to prevent.

**How it works**

**Acquire, renew, release safely**

```text
ACQUIRE
  SET lock:account:42 <owner-uuid> NX PX 10000
  NX  -> only if it does not exist
  PX  -> expires in 10 s automatically
  -> atomic; exactly one caller succeeds

RENEW  (while working)
  if GET lock == my-uuid:
    PEXPIRE lock 10000
  -> extend only if still the owner

RELEASE  (must be conditional)
  if GET lock == my-uuid:
    DEL lock
  -> as a single atomic script, not two commands
  -> a plain DEL can delete someone else's lock

THE EXPIRY PROBLEM
  holder pauses 12 s (GC)
  lock expires at 10 s
  another process acquires it and starts working
  original holder resumes and continues working
  -> both are inside the critical section
  -> the lock did not fail; it did what it must

FENCING
  the store returns an increasing token on acquire
    acquire -> token 41
    acquire -> token 42
  every write carries the token; the resource keeps
  the highest seen and rejects lower
  -> the lapsed holder's writes are rejected
```

1. **Always set an expiry** — A lock without one becomes permanent the first time a holder crashes.
2. **Store a unique owner identifier** — Release and renewal must both be conditional on ownership.
3. **Make release atomic** — Check-then-delete as two operations can delete a successor's lock.
4. **Renew while working, do not simply pick a long TTL** — A long TTL delays recovery from a genuine crash.
5. **Use a fencing token for correctness** — Expiry makes overlapping holders possible; only the resource can reject them.
6. **Keep critical sections short** — Long ones make expiry likely and contention expensive.

**What the lock actually guarantees**

```text
EFFICIENCY USE  (common, and fine)
  goal: avoid doing the same work twice
  occasional overlap = wasted effort, no damage
  -> a plain lock with TTL is sufficient
  -> example: regenerating a cache entry

CORRECTNESS USE  (rare, and demanding)
  goal: the operation must NEVER happen twice
  overlap = corrupted data or double spend
  -> a lock alone is NOT sufficient
  -> needs fencing at the resource, or the operation
     must be conditional (compare-and-set)

ASKING WHICH ONE YOU HAVE
  if two holders overlap, what is the damage?
    wasted CPU        -> efficiency lock, ship it
    wrong balance     -> you need fencing, or you
                         need the database to enforce
                         it with a transaction

THE SIMPLER ANSWER
  if the resource is a database, a row lock or a
  conditional update enforces exclusion better than
  any external lock, because the enforcement is at
  the place the change happens
```

| Metric | Value | Note |
|---|---|---|
| Must have | expiry | or it wedges |
| Consequence | overlap possible | **by construction** |
| Correctness | fencing token | at the resource |
| Better option | database constraint | where available |

> **Most distributed locks are guarding something that could enforce exclusion itself**  
> If the protected resource is a row in a database, a conditional update or a transaction provides mutual exclusion at the point of change, with no expiry, no renewal and no split holder. An external lock is genuinely needed when the resource cannot arbitrate — an external API, a filesystem, a third-party system — and reaching for one when the database was already capable adds a failure mode for nothing.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| Redis SET NX PX | Fast, simple, ubiquitous | Failover can lose the lock; no fencing token natively |
| etcd or ZooKeeper | Consensus-backed, leases, watches | Heavier; higher latency per acquire |
| Database row lock | Transactional with the work | Scope limited to that database |
| Conditional update or version column | Exclusion without a lock at all | Requires the work to be a single update |
| Cloud lock services | Managed, with fencing tokens | Latency and service limits |
| Redlock across Redis nodes | Attempts safety without consensus | Contested; use consensus if correctness matters |

Redis is the most common choice and the least safe for correctness use: a failover can promote a replica that never received the lock, allowing a second acquisition while the first holder is still working. That is acceptable for efficiency locks and not for anything where overlap causes damage.

**Trade-offs**

**Ways to prevent concurrent action**

| Approach | Guarantees exclusion | Failure mode | Cost |
|---|---|---|---|
| Database transaction or row lock | Yes | Contention, deadlocks | None extra |
| Conditional update with a version | Yes | Retry on conflict | None extra |
| Distributed lock with TTL | Not under pauses | Overlapping holders | A store to operate |
| Distributed lock plus fencing | Yes, at the resource | Complexity | Store plus resource support |
| Partition work by key | Yes, structurally | Needs partitionable work | Routing |
| Make the operation idempotent | Not needed | n/a | Design effort |

Partitioning is the structural answer worth reaching for: if every account is handled by exactly one worker determined by a hash, no two workers ever contend for it and no lock is required at all.

> **Ask before choosing it**  
> If two holders overlapped, what would actually break? Wasted work means a simple lock is enough; incorrect data means the lock is not the mechanism you need and the resource must enforce it — either with a fencing token or with its own transactional guarantee.

**How it fails**

**How distributed locks go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Resource blocked forever | Lock acquired with no expiry | Always set a TTL |
| Two holders in the critical section | Holder paused past the expiry | Fencing tokens; or a conditional resource update |
| A process releases another's lock | Unconditional delete | Atomic check-owner-and-delete |
| Lock lost during failover | Non-consensus store promoting a stale replica | Consensus store where correctness matters |
| Constant lock expiry | TTL shorter than the work | Renew during work rather than lengthening the TTL |
| Throughput collapses | Long critical sections under contention | Shorten the section or partition the keys |
| Deadlock across several locks | Inconsistent acquisition order | Global ordering, or acquire one lock only |

> **A distributed lock cannot provide mutual exclusion on its own**  
> Expiry is mandatory because holders die silently, and expiry means a slow holder can still be working when its claim lapses. Every distributed lock therefore permits overlapping holders under pauses, and the only place that can prevent damage is the resource itself — through a fencing token it checks, or through its own transactional exclusion. A design whose correctness depends solely on the lock is relying on a guarantee the lock does not offer.

**Where it is used**

- **Cache regeneration**, where an efficiency lock prevents many processes rebuilding the same value.
- **Scheduled task coordination**, ensuring one instance runs a job — effectively lightweight leader election.
- **External API rate coordination**, serialising access to a third party that cannot arbitrate for itself.
- **Filesystem and object-store operations**, where the storage layer offers no locking primitive.
- **Migration and maintenance scripts**, preventing two operators from running the same destructive operation.

**In the interview**

The expected answer explains why expiry and mutual exclusion are in tension, rather than presenting the lock as sufficient.

- **State the dilemma directly**: without expiry a crash wedges the resource, with expiry a slow holder overlaps a new one.
- **Require conditional release** with an owner identifier, since a plain delete can free someone else's lock.
- **Introduce fencing tokens** and show the resource rejecting stale ones — that is where correctness comes from.
- **Separate efficiency from correctness use** and choose the mechanism accordingly; most locks are the former.
- **Suggest the database or partitioning instead** where the resource can enforce exclusion itself, because that removes the failure mode entirely.

**Practice drill**

Two workers must not adjust the same account balance concurrently. Write the acquire, renew and release operations, including what makes release safe. Then trace a 12-second pause on the holder of a 10-second lock: who holds the lock, what the paused worker does on waking, and what the balance ends up as. Add fencing and show what changes. Finally, argue whether an external lock was needed at all given that the balance lives in a database.

**Go deeper**

A distributed lock grants exclusive access to a resource across processes, using a shared store that can decide atomically who holds it.

**Expiry is mandatory and it is what undermines the guarantee.** A holder that crashes cannot release, so without a time limit the first failure blocks the resource permanently. With a time limit, a holder that is merely slow — paused by garbage collection, stalled on a disk — remains inside the critical section after its claim has lapsed and a successor has acquired it. Overlapping holders are therefore a structural property of distributed locking, not a bug in a particular implementation.

**Correctness has to be enforced where the change happens.** Since the lock cannot prevent overlap, the resource must reject work from a holder whose claim has expired, which requires a monotonically increasing fencing token carried on every write and remembered by the resource. Without it, the late writes of a lapsed holder land after the new holder's and silently overwrite them — the lock behaved exactly as designed and the data is still wrong.

**Distinguishing efficiency from correctness decides how much machinery is warranted.** A lock that prevents duplicate cache regeneration can tolerate occasional overlap, because the cost is wasted CPU; a lock guarding a balance adjustment cannot, because the cost is money. The first needs only a TTL and a conditional release; the second needs fencing or a resource that arbitrates for itself, and conflating them is how systems end up with a plausible-looking lock protecting something that genuinely needed a transaction.

**Release must be conditional on ownership, and this is a common omission.** A process whose lease expired and who then deletes the key is removing whatever lock currently exists — including a successor's — which admits a third process and produces exactly the concurrency the lock existed to prevent. Comparing the owner identifier and deleting in one atomic operation is what makes release safe, and splitting it into a read followed by a delete reintroduces the race.

**The choice of store determines what failures are possible.** A single-node cache with asynchronous replication can promote a replica that never saw the lock, permitting a second acquisition while the first holder works — fine for efficiency, unacceptable for correctness. A consensus-backed store costs more per acquisition and does not lose the lock on failover, which is the property that matters when overlap causes damage.

**Most distributed locks are guarding a resource that could enforce exclusion itself.** When the protected state is a database row, a transaction or a conditional update provides mutual exclusion at the point of change, with no expiry, no renewal and no possibility of two holders. External locks earn their place where the resource cannot arbitrate — third-party APIs, filesystems, systems outside your control — and partitioning work by key removes the contention entirely where the workload allows it, which is better than coordinating access well.

**Related patterns:** Leader Election · Leases · Idempotency Key · Consistent Hashing

---

### Leases

*Grant a right for a bounded period rather than indefinitely, so a holder that dies silently loses it automatically without anyone having to detect the failure.*

> **When you hear…** A holder can die without releasing · ownership must recover without operator action · caches or claims that must not outlive their validity

**Flow:** `Grant with deadline` → `Holder renews` → `Renewal stops` → `Deadline passes` → `Right released`

**The problem**

A process claims a resource and then vanishes — the host is terminated, the network drops, the kernel kills it. It never releases, and nothing else can tell whether it is dead or merely quiet. Every claim without an expiry becomes permanent the first time this happens.

Detecting the failure explicitly is harder than it looks: an unanswered heartbeat is indistinguishable from a partition, and a system that waits for certainty waits forever.

> **Make the right decay unless actively renewed**  
> If a grant carries a deadline, then a holder that stops functioning stops renewing, and the right expires on its own. Nobody has to decide that the holder is dead — the absence of renewal is the decision, and it is made by the passage of time rather than by a judgement that could be wrong.

**Mental model**

A time-bounded grant with a deadline both parties can reason about. Continuing to hold it requires continuing to ask.

1. **Grant** — The issuer gives a right valid until a specific time.
2. **Use** — The holder acts, knowing when its right expires.
3. **Renew** — The holder extends the deadline before it arrives.
4. **Lapse** — Renewal stops, the deadline passes, and the right ends with no action required.
5. **Reassign** — The issuer is free to grant it to someone else.

> **The holder's clock and the issuer's clock disagree**  
> A lease is a deadline, and two machines rarely agree precisely on what time it is. Clock skew means a holder may believe it has two seconds left when the issuer already considers the lease expired. Safe designs have the holder stop acting well before the nominal deadline, and never rely on the two sides computing the same instant.

**How it works**

**Sizing a lease and using it safely**

```text
GRANT
  lease = { holder: worker-7, expires_at: T + 10 s }

HOLDER
  renew at T + 4 s  -> expires_at = T + 14 s
  renew at T + 8 s  -> expires_at = T + 18 s
  ...
  renewal fails     -> STOP WORKING IMMEDIATELY
                       do not wait for expiry

THE SAFETY MARGIN
  issuer considers it expired at expires_at
  holder should stop at expires_at - skew - latency
  typical: stop at 80% of the lease
  -> a holder acting right up to the deadline will
     sometimes act after it

RENEWAL INTERVAL
  renew every lease/3
  -> two renewals can fail before the lease lapses
  -> tolerates transient blips without churn

SIZING THE LEASE
  too short: normal pauses cause spurious expiry
  too long:  dead holders block the resource
  lease > worst realistic pause x 2
  typical 10-30 s for coordination
          seconds to minutes for caches
```

1. **Renew at roughly a third of the lease period** — Two missed renewals then become survivable rather than fatal.
2. **Stop working when renewal fails, not when the deadline passes** — By the deadline the issuer may already have reassigned the right.
3. **Leave a safety margin for clock skew and latency** — Both sides compute the deadline independently and will not agree exactly.
4. **Use durations rather than absolute timestamps across machines** — Relative time is far less sensitive to clock disagreement.
5. **Pair with fencing where the right protects a resource** — Expiry admits overlap; only the resource can reject the lapsed holder.
6. **Size from real pause durations** — A lease shorter than a garbage collection pause guarantees churn.

**Where leases appear**

```text
COORDINATION
  leader election   the leadership term is a lease
  distributed lock  the TTL is a lease
  -> both inherit the expiry-versus-exclusion tension

CACHING
  a cache entry TTL is a lease on the right to serve
  that value without rechecking
  -> read leases: a client may cache until the lease
     expires; the server must not change the value
     without first waiting out or revoking the lease

RESOURCE ASSIGNMENT
  DHCP addresses, partition ownership, shard
  assignment
  -> a node that vanishes releases its share
     automatically

MEMBERSHIP
  a node is a member while its lease is renewed
  -> failure detection with no explicit detector

THE COMMON SHAPE
  every one of these replaces the question
  "is the holder alive?" with
  "did the holder renew?"
  -> the second question always has an answer
```

| Metric | Value | Note |
|---|---|---|
| Failure handling | automatic | **no detector needed** |
| Renewal | every lease/3 | tolerates blips |
| Stop acting | on failed renewal | not at deadline |
| Hazard | clock skew | keep a margin |

> **A holder that acts right up to the deadline will sometimes act after it**  
> Network latency on the renewal, a scheduling delay, and a small clock difference each move the effective boundary. A holder that keeps working until its own clock shows expiry has no margin for any of them, and will occasionally perform an action that the issuer has already reassigned. Stopping early is not caution — it is the only way the arithmetic works out.

**Technologies**

| Use | Implementation | Note |
|---|---|---|
| Coordination | etcd and ZooKeeper leases | Sessions expire when renewal stops |
| Locking | Redis SET with PX | TTL is the lease; renewal is an explicit extend |
| Membership | Kubernetes lease objects | Controllers renew to signal liveness |
| Caching | HTTP max-age and TTLs | A lease on serving without revalidating |
| Networking | DHCP leases | The original consumer-facing example |
| Read leases | Distributed filesystems and databases | The server must respect them before mutating |

Read leases are the least familiar and most instructive: a server that has granted one cannot change the underlying value until it expires or is revoked, which is what makes client-side caching safe without a validation round trip on every read.

**Trade-offs**

**Ways to reclaim a right from a failed holder**

| Approach | Detection needed | Recovery time | Risk |
|---|---|---|---|
| Explicit release only | n/a | Never, if the holder died | Permanent block |
| Lease with expiry | None | Up to one lease | Overlap under pauses |
| Heartbeat with a detector | Yes | Detector interval | False positives |
| Operator intervention | Human | Minutes to hours | Slow and error-prone |
| Lease plus fencing | None | Up to one lease | Overlap made harmless |

The lease row and the heartbeat row are closer than they look: a lease is a heartbeat whose failure has a predefined consequence, which is why it needs no separate detector and no decision about what to do when a heartbeat is missed.

> **Ask before choosing it**  
> What is the longest pause a healthy holder can experience? That number sets the lease floor, and teams routinely underestimate it — garbage collection, disk stalls and host-level freezes are all longer in production than in testing, and a lease below them produces constant unnecessary failovers.

**How it fails**

**How leases go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Constant spurious expiry | Lease shorter than real pauses | Size above worst-case stalls |
| Two holders acting | Holder continued past expiry | Stop on failed renewal; add fencing |
| Right held long after death | Lease far too long | Shorten, accepting more renewal traffic |
| Expiry mid-operation | Operation longer than the lease | Renew during work, or checkpoint and resume |
| Clock skew causes disagreement | Absolute timestamps across machines | Use durations and a safety margin |
| Renewal storms | Very short leases across many holders | Longer leases, or jittered renewal |
| Stale data served | Cache lease outlived the value | Revoke, or keep leases short for volatile data |

> **Leases turn failure detection into a timing problem, and timing problems need margins**  
> Because the whole mechanism rests on deadlines computed independently on two machines, every source of delay — clock drift, network latency, scheduler jitter, a long pause — erodes the gap between what the holder believes and what the issuer believes. Without a deliberate margin the two sides will eventually disagree at exactly the wrong moment, and the resource will be held by two parties who each believe they are the only one.

**Where it is used**

- **Coordination systems**, where sessions are leases and losing one relinquishes every right it held.
- **Kubernetes**, which expresses both leader election and node liveness as renewable lease objects.
- **Distributed filesystems**, granting read and write leases so clients can cache without revalidating.
- **Partition and shard assignment**, where a vanished node's share is reassigned automatically on expiry.
- **DHCP**, the original example most people have encountered, reclaiming addresses from devices that left.

**In the interview**

The insight to articulate is that a lease replaces failure detection with a deadline, and that deadlines need margins.

- **Say it removes the need for a detector.** Absence of renewal is the failure signal, and it always has an answer where liveness checks do not.
- **Renew at a third of the period** so transient blips do not cause failover, and explain the arithmetic.
- **Stop working when renewal fails**, not at the nominal deadline, because the issuer may already have reassigned.
- **Raise clock skew** and argue for durations and a safety margin rather than shared absolute timestamps.
- **Connect it to locks and leader election** as the same mechanism, inheriting the same expiry-versus-exclusion tension.

**Practice drill**

A worker holds a lease on a partition. Choose the lease duration and renewal interval, justifying both from realistic pause durations. Say exactly when the worker must stop processing and why that is earlier than the deadline. Then trace a network blip that causes two renewals to fail and a third to succeed, and a longer partition that causes expiry — describing what the issuer does and what the worker does on reconnecting.

**Go deeper**

A lease is a right granted for a bounded period, which expires unless the holder actively renews it.

**It replaces failure detection with the passage of time.** Deciding whether a silent process is dead or partitioned is undecidable from the outside, and any system that waits for certainty waits indefinitely. A lease sidesteps the question entirely: the holder either renewed or it did not, and that has a definite answer at every moment. The failure handling is therefore automatic rather than the result of a judgement that might be wrong.

**The renewal interval sets the tolerance for transient trouble.** Renewing at roughly a third of the lease means two consecutive renewals can fail without losing the right, which absorbs ordinary network blips and brief scheduling delays. Renewing only just before expiry makes every hiccup fatal and produces churn in which rights bounce between holders and no work completes.

**The holder must stop earlier than the deadline, and this is the subtle part.** Renewal latency, scheduler delay and clock disagreement all shift the effective boundary, so a holder that works right up to its own computed expiry will sometimes act after the issuer has already reassigned the right. Stopping at a fraction of the lease is what keeps the two views from overlapping, and treating the deadline as a hard stop rather than a target is the difference between a safe implementation and one that fails rarely and confusingly.

**Sizing is a direct trade between spurious expiry and slow recovery.** A lease below realistic pause durations deposes healthy holders whenever garbage collection or a disk stall runs long; a very long lease leaves a dead holder's right unavailable for its full duration. Production pauses are consistently longer than the ones seen in testing, which is why leases sized from measurements in a quiet environment tend to cause churn once real load arrives.

**Expiry admits overlap, so leases guarding a resource need fencing.** A holder paused past its deadline resumes believing it still holds the right while a successor is already acting, and nothing in the lease mechanism prevents both from writing. The resource itself must reject the lapsed holder, using a monotonically increasing token, which is the same conclusion distributed locks and leader election reach by the same route — because all three are leases wearing different names.

**Once recognised, the pattern appears everywhere.** Cache TTLs are leases on the right to serve a value without rechecking; DHCP grants leases on addresses; partition assignment, cluster membership and leader terms are all rights that decay unless renewed. Seeing them as one mechanism is what makes the recurring questions — how long, how often to renew, what margin, what happens on expiry — transferable rather than rediscovered separately in each system.

**Related patterns:** Leader Election · Distributed Lock · Heartbeat Failure Detection · Cache Invalidation

---

### Heartbeat Failure Detection

*Have nodes send periodic signals and declare a node failed when they stop, accepting that partitions and pauses make the verdict a guess.*

> **When you hear…** Failed nodes must be removed from rotation · work must be reassigned · membership has to reflect reality automatically

**Flow:** `Periodic beat` → `Monitor tracks arrivals` → `Beats stop` → `Timeout elapses` → `Declared failed`

**The problem**

A node stops responding. It may have crashed, it may be paused, it may be perfectly healthy behind a broken network link. From the outside these are indistinguishable, and yet the system has to decide whether to reassign its work.

Waiting longer improves accuracy and delays recovery. Deciding quickly recovers fast and sometimes evicts a node that was fine, potentially causing two nodes to do the same work. There is no setting that avoids both errors.

> **Failure detection is a guess with a tunable error profile**  
> No protocol can distinguish a crashed node from an unreachable one, because the evidence is identical. What a detector can do is choose where to sit between detecting too eagerly and detecting too late, and make the consequences of a wrong guess survivable — which is why detection is always paired with fencing or idempotence rather than trusted on its own.

**Mental model**

A periodic signal, a monitor that tracks arrival times, and a timeout beyond which absence is treated as failure.

1. **Beat** — Each node sends a signal at a fixed interval.
2. **Observe** — A monitor records the time of the last arrival.
3. **Suspect** — Missing beats raise suspicion without immediate action.
4. **Declare** — After a threshold, the node is treated as failed.
5. **Recover** — A returning node rejoins, and must handle having been declared dead.

> **A node that was declared dead does not know it**  
> The evicted node is frequently still running, still holding resources and still processing. It learns of its eviction only when it next communicates, which may be long after its work has been reassigned. Every system using heartbeats must therefore assume a declared-dead node is still active, which is exactly the assumption that makes fencing necessary.

**How it works**

**Interval, threshold and what they cost**

```text
FIXED THRESHOLD
  interval  1 s
  threshold 5 missed beats
  -> detection in ~5 s
  -> a 6 s GC pause causes a FALSE positive

THE TRADE
  threshold 3  -> detect in 3 s, more false positives
  threshold 10 -> detect in 10 s, fewer false
                  positives, slower recovery
  -> there is no value that avoids both errors

ADAPTIVE (phi accrual)
  track the DISTRIBUTION of inter-arrival times
  output a suspicion level, not a boolean
    phi = 1   probably fine
    phi = 8   very likely dead
  act at different thresholds for different actions:
    phi 2  -> stop routing new work
    phi 8  -> reassign its partitions
  -> cheap actions on weak evidence, expensive
     actions on strong evidence

WHO MONITORS WHOM
  central monitor  simple, single point of failure,
                   and its own network view
  all-to-all       n^2 traffic, does not scale
  gossip           each node monitors a few, and
                   suspicion spreads
```

1. **Separate suspicion from declaration** — Cheap precautions on weak evidence, disruptive actions only on strong evidence.
2. **Size the threshold against real pause durations** — A threshold below the worst garbage collection pause guarantees false positives.
3. **Prefer adaptive detection where pause behaviour varies** — A fixed timeout tuned for the median is wrong for the tail.
4. **Do not let one monitor's network view decide alone** — A monitor behind a broken link declares healthy nodes dead.
5. **Require confirmation from several observers** — Indirect probing distinguishes a node failing from a link failing.
6. **Make reassignment safe against the node returning** — Detection will be wrong sometimes; fencing is what makes that survivable.

**Why false positives are the dangerous direction**

```text
FALSE NEGATIVE  (dead node not yet detected)
  requests to it fail or time out
  -> bad, but bounded and self-correcting
  -> callers retry elsewhere

FALSE POSITIVE  (healthy node declared dead)
  its partitions are reassigned
  it is still running and still writing
  -> TWO nodes own the same data
  -> divergence, duplicate work, corruption

-> a detector that is slightly slow is far safer than
   one that is slightly eager

THE CASCADE
  network congestion delays heartbeats
  -> nodes declared dead
  -> their work is reassigned
  -> reassignment causes data movement
  -> data movement increases congestion
  -> more heartbeats delayed
  this is how a partial network problem becomes a
  cluster-wide outage

THE CONTROL
  cap how many nodes may be evicted at once
  if more than X% appear dead, assume the network is
  at fault, not the nodes
```

| Metric | Value | Note |
|---|---|---|
| Detection time | interval × threshold | tunable |
| False positive | two owners | **the dangerous one** |
| Better detector | adaptive suspicion | graded response |
| Safety valve | cap evictions | stops cascades |

> **Mass eviction is almost always the network, not the nodes**  
> When a third of a cluster appears to fail simultaneously, the overwhelmingly likely explanation is a network event rather than a third of the hardware dying at once. A detector that acts on that observation reassigns enormous amounts of work, generates traffic on an already-degraded network, and deepens the problem. Capping how many nodes may be evicted in a window converts a cascade into a degradation.

**Technologies**

| Approach | Used by | Note |
|---|---|---|
| Fixed timeout heartbeats | Most schedulers and load balancers | Simple; badly suited to variable pauses |
| Phi accrual detection | Cassandra and similar | Suspicion level rather than a boolean |
| SWIM-style gossip | Consul, Serf | Scales; indirect probes cut false positives |
| Lease expiry | etcd, ZooKeeper, Kubernetes | Detection folded into the lease |
| Load balancer health checks | Every fleet | Removes from rotation without cluster-wide consequences |
| Indirect probing | SWIM | Asks others to confirm before declaring |

Indirect probing is the cheapest large improvement available: before declaring a node dead, ask a few other nodes to probe it. A node that answers them is clearly alive and the problem is the link between it and the original observer — which is a different fault with a different remedy.

**Trade-offs**

**Detector designs**

| Design | Detection speed | False positives | Scalability |
|---|---|---|---|
| Short fixed timeout | Fast | Many | Fine |
| Long fixed timeout | Slow | Few | Fine |
| Adaptive suspicion | Tunable per action | Few | Fine |
| All-to-all heartbeats | Fast | Few | Poor beyond ~100 nodes |
| Gossip with indirect probes | Moderate | Few | Very good |
| Central monitor | Fast | Depends on its view | Single point of failure |

Central monitoring hides a subtle bias: the monitor's own network position determines what it sees, so a monitor behind a degraded link declares perfectly healthy nodes dead and the cluster acts on one machine's misperception.

> **Ask before choosing it**  
> What happens if this verdict is wrong in each direction? A slow detection that leaves requests failing is usually recoverable; an eager detection that gives two nodes the same partition may not be. The asymmetry should decide the threshold, not the desire for fast failover.

**How it fails**

**How failure detection goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Healthy nodes evicted | Threshold below real pause durations | Raise it; use adaptive detection |
| Two owners for one partition | False positive with no fencing | Fencing tokens on every write |
| Cluster-wide cascade | Mass eviction during congestion | Cap concurrent evictions |
| Dead node serves for minutes | Threshold far too high | Balance against the false-positive cost |
| Monitor's view taken as truth | Single observer | Indirect probes and confirmation |
| Detection traffic dominates | All-to-all heartbeats at scale | Gossip-based membership |
| Returning node causes chaos | No handling of rejoin after eviction | Fence it out; require a clean rejoin |

> **Detection and reassignment together can amplify a network problem into an outage**  
> Congestion delays heartbeats, delayed heartbeats cause evictions, evictions trigger data movement, and data movement adds traffic to the congested network. Each step is a correct local response and the composition is a positive feedback loop. The only reliable brake is a limit on how much reassignment may happen at once, applied before the loop has a chance to run.

**Where it is used**

- **Cluster membership in distributed databases**, deciding which nodes hold which replicas.
- **Kubernetes node liveness**, where a node that stops renewing has its pods rescheduled.
- **Load balancer health checks**, the narrowest and safest form — removal from rotation only.
- **Consumer group coordination**, detecting a dead consumer and rebalancing its partitions.
- **Gossip-based systems**, where detection is distributed and suspicion spreads rather than being centrally decided.

**In the interview**

State the impossibility first; candidates who present detection as solvable miss the entire design problem.

- **Say a crash and a partition are indistinguishable**, so detection is a guess with a tunable error profile.
- **Argue the asymmetry.** A false positive gives two owners and can corrupt; a false negative merely fails requests.
- **Offer graded suspicion** — cheap actions on weak evidence, reassignment only on strong evidence.
- **Raise the cascade** where eviction causes data movement that delays more heartbeats, and cap concurrent evictions.
- **Pair detection with fencing**, because the verdict will sometimes be wrong and only the resource can make that harmless.

**Practice drill**

A 50-node cluster uses 1-second heartbeats with a 5-beat threshold. Say what happens during a 6-second garbage collection pause on one node, including what its partitions do and what the node does on waking. Then a network event delays heartbeats from 20 nodes at once: describe the sequence that follows, why it can worsen, and the specific control you would add. Finally, choose between a fixed threshold and adaptive suspicion for this cluster and justify it.

**Go deeper**

Heartbeat failure detection infers that a node has failed from the absence of its periodic signals, and acts on that inference.

**The verdict is unavoidably a guess.** A crashed node and a healthy but unreachable node produce identical evidence — silence — and no protocol can separate them, because the information needed is on the far side of the failure. What a detector controls is not accuracy but the balance between deciding too early and deciding too late, and understanding that the choice is between error types rather than between right and wrong is the foundation of every sound design here.

**The two errors are not equally costly, and that asymmetry should drive the threshold.** Failing to detect a dead node leaves requests timing out, which is bad and self-correcting as callers retry elsewhere. Wrongly declaring a live node dead reassigns its work while it continues to operate, producing two owners for the same data — a divergence that is not self-correcting and may not be detectable afterwards. A detector tuned slightly slow is therefore much safer than one tuned slightly fast.

**Graded suspicion is better than a boolean.** Tracking the distribution of arrival times and producing a suspicion level allows cheap precautions on weak evidence — stop sending new work — and expensive ones only on strong evidence. A single timeout forces every consequence to share one threshold, so the value that is right for pausing traffic is simultaneously wrong for reassigning partitions, and one of the two is always badly served.

**A single observer's view is not the cluster's view.** A monitor sitting behind a degraded link sees silence from healthy nodes and, if trusted alone, causes the whole cluster to act on its private misperception. Asking other nodes to probe before declaring distinguishes a failing node from a failing link — a different fault with a different remedy — and it is among the cheapest accuracy improvements available.

**Detection and recovery can form a feedback loop.** Congestion delays heartbeats, delays cause evictions, evictions trigger data movement, and that movement congests the network further. Every step is locally reasonable and the composition turns a partial network problem into a cluster-wide outage. Capping how many nodes may be evicted in a window is the brake, and it encodes a useful prior: when many nodes appear to fail simultaneously, the network is the likelier culprit.

**Because the verdict will sometimes be wrong, detection must be paired with something that makes it harmless.** A node declared dead does not know it, keeps running, and keeps writing until it next communicates. Fencing tokens let the resource reject those writes, idempotent handling makes duplicate work benign, and leases fold the detection into the grant itself. Detection alone decides who should act; it cannot stop the other party from acting, and designs that forget this are correct exactly until the first long pause.

**Related patterns:** Leases · Gossip Membership · Leader Election · Circuit Breaker

---

### Gossip Membership

*Each node exchanges state with a few random peers, so membership and failure information spreads through the cluster without any central registry.*

> **When you hear…** Hundreds or thousands of nodes · no central coordinator wanted · membership must survive partitions and keep working

**Flow:** `Pick random peers` → `Exchange state` → `Merge by version` → `Spread exponentially` → `Whole cluster converges`

**The problem**

Every node needs to know which other nodes exist and which are alive. Having each one heartbeat every other produces traffic proportional to the square of the cluster size — at a thousand nodes that is a million connections, which is infrastructure spent entirely on knowing who is present.

A central registry removes the traffic and adds a dependency that must be more available than the cluster it describes. When it is unreachable, no node can learn about any change, including the failure of the node it is currently trying to talk to.

> **Information spreads fastest when everyone repeats it to a few others**  
> If each node tells a small random sample of peers what it knows, and they do the same, knowledge reaches the whole cluster in a number of rounds that grows only logarithmically with size. No node needs a complete picture of who to inform, no node is essential to the process, and the traffic each one generates is constant regardless of how large the cluster becomes.

**Mental model**

Periodic pairwise exchanges with random peers, where each side merges the other's view using version numbers to decide what is newer.

1. **Select** — Every interval, each node picks a few peers at random.
2. **Exchange** — They send each other their view of cluster membership.
3. **Merge** — Each entry is kept or replaced according to its version.
4. **Spread** — Because every recipient repeats it, information reaches everyone exponentially fast.
5. **Converge** — All nodes eventually agree, without anyone coordinating.

> **Convergence is eventual, so nodes act on stale membership**  
> During the seconds it takes information to propagate, different nodes hold different views: one still routes to a node that another has already marked dead. Gossip does not provide a consistent membership snapshot, and any decision requiring all nodes to agree at the same instant — such as who owns a partition exclusively — needs consensus rather than gossip.

**How it works**

**Why gossip scales, and how fast it spreads**

```text
TRAFFIC
  all-to-all:  each node talks to n-1 others
    n = 1000 -> ~1,000,000 pairings
  gossip:      each node talks to k peers (k = 3)
    n = 1000 -> 3000 messages per round
  -> per-node cost is CONSTANT as the cluster grows

SPREAD
  each round, the number of informed nodes roughly
  multiplies by (k + 1)
    round 0:     1
    round 1:     4
    round 2:    16
    round 3:    64
    round 4:   256
    round 5:  1024
  -> ~log(n) rounds; at 1 s intervals, a 1000-node
     cluster learns anything in about 5 s

MERGING
  each entry carries { node, state, version }
  on exchange, keep the higher version
    alive(v7) vs suspect(v9) -> suspect wins
  -> versions, not timestamps; clocks disagree

SWIM REFINEMENT
  direct probe fails
  -> ask k other nodes to probe indirectly
  -> only if all fail, mark suspect
  -> suspect, then dead after a timeout
  -> cuts false positives from single bad links
```

1. **Gossip with a small constant number of peers** — Three or four is enough; more raises traffic without materially faster spread.
2. **Version every entry** — Merging needs a total order that does not depend on synchronised clocks.
3. **Use indirect probes before suspecting** — One broken link should not convict a node the rest of the cluster can reach.
4. **Introduce a suspect state before dead** — It gives the node a window to refute the rumour, which sharply cuts false positives.
5. **Let a node refute its own death** — A returning node increments its version and the correction spreads the same way.
6. **Do not build exclusive ownership on gossip** — Eventual agreement cannot support decisions that require simultaneous agreement.

**What gossip is good for and what it is not**

```text
GOOD FOR
  membership       who exists, who seems alive
  failure rumours  spread without a central detector
  configuration    values that tolerate brief
                   disagreement
  load information which nodes are busy

NOT GOOD FOR
  leader election      needs agreement at an instant
  exclusive ownership  two nodes may both believe
                       they own a partition
  ordered operations   gossip has no global order
  strong consistency   convergence is eventual by
                       construction

THE COMMON COMBINATION
  gossip for membership and liveness
  + consensus (Raft) for the few decisions that must
    be exclusive
  -> the cheap mechanism handles the high-volume,
     tolerant information; the expensive one handles
     the small, critical set

PARTITION BEHAVIOUR
  network splits into A and B
  each side gossips internally and converges
  each side believes the other side is dead
  on heal, versions reconcile and views merge
  -> gossip keeps working on both sides, which is the
     point, and is also why it cannot arbitrate
```

| Metric | Value | Note |
|---|---|---|
| Traffic per node | constant | **independent of n** |
| Spread time | log(n) rounds | ~5 s at n=1000 |
| Consistency | eventual | not a snapshot |
| Pair with | consensus | for exclusive decisions |

> **A partition produces two confident, incompatible views**  
> Each side of a network split gossips internally, converges, and concludes that the other side has died. Both views are internally consistent and both are wrong about half the cluster. This is not a flaw to be fixed — continuing to operate on both sides is precisely the availability gossip is chosen for — but it means anything requiring a single answer must be decided by a mechanism that refuses to proceed without a quorum.

**Technologies**

| System | Use | Note |
|---|---|---|
| SWIM protocol | Membership with indirect probing | The reference design most implementations follow |
| Consul and Serf | Cluster membership and events | SWIM in production form |
| Cassandra | Node state and token ownership | Gossip plus versioned state |
| Redis Cluster | Node and slot membership | Gossip between nodes on a separate port |
| Kubernetes | Central API server instead | A deliberate alternative choice |
| Consensus systems | The exclusive decisions | Paired with gossip, not replaced by it |

Kubernetes is the instructive counterexample: it uses a central, consensus-backed store for membership rather than gossip, accepting a dependency in exchange for a consistent view. At the cluster sizes it targets that is the better trade, which shows the choice is about scale and consistency needs rather than about one approach being superior.

**Trade-offs**

**Membership approaches**

| Approach | Traffic | Consistency | Central dependency |
|---|---|---|---|
| All-to-all heartbeats | O(n²) | Fast within a node's view | None |
| Central registry | O(n) | Consistent | Yes, must outlive the cluster |
| Consensus-backed store | O(n) | Strongly consistent | Yes, quorum required |
| Gossip | O(n) total, O(1) per node | Eventual | None |
| Gossip plus consensus | O(n) | Eventual plus strong where needed | Only for critical decisions |

The final row is what large systems actually run: gossip carries the high-volume, failure-tolerant information, and a small consensus group decides the handful of things that genuinely cannot be decided twice.

> **Ask before choosing it**  
> How many nodes, and does anything depend on every node agreeing at the same moment? Below a few dozen nodes a central registry is simpler and gives a consistent view; above a few hundred the quadratic and central options both strain, and gossip becomes the only comfortable choice.

**How it fails**

**How gossip membership goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Healthy node marked dead | Single failed probe path | Indirect probes and a suspect state |
| Slow convergence | Too few peers or too long an interval | Raise fan-out or frequency modestly |
| Traffic grows with cluster size | Fan-out proportional to n | Keep the peer count constant |
| Both sides of a partition act as the cluster | Gossip used for ownership decisions | Use consensus for exclusivity |
| Stale state resurrected | Merging on timestamps across skewed clocks | Version numbers, not wall clocks |
| Rumours never settle | No refutation mechanism | Let a node increment its version to refute |
| Split-brain data ownership | Ownership derived from membership view | Fence writes; require quorum for ownership |

> **Deriving exclusive ownership from a gossip view is a split-brain generator**  
> Because each node's membership picture is eventually consistent, two nodes can simultaneously conclude that they own a partition — one having heard that the previous owner died, the other never having heard it. Both act, both write, and the data diverges with no record of which was authoritative. Gossip is the right way to learn who probably exists; it is never sufficient for deciding who exclusively owns something.

**Where it is used**

- **Cassandra and Dynamo-style stores**, using gossip for node state and token ownership across large rings.
- **Consul and Serf**, providing SWIM-based membership and event propagation as a service.
- **Redis Cluster**, where nodes gossip slot ownership and liveness on a dedicated bus.
- **Large service meshes**, propagating endpoint health without every proxy polling a central registry.
- **Peer-to-peer and blockchain networks**, where there is no central authority available even in principle.

**In the interview**

Show the scaling argument with numbers and be explicit about what gossip cannot decide.

- **Contrast the traffic directly**: a million pairings for all-to-all at a thousand nodes against a constant few per node.
- **Give the logarithmic spread** — informed nodes multiply each round, so a thousand-node cluster converges in about five.
- **Merge on versions, not timestamps**, because clock skew makes wall-clock merging incorrect.
- **Describe suspect plus indirect probing** as the mechanism that keeps false positives low without slowing detection much.
- **State the limit plainly**: eventual membership cannot support exclusive ownership, so pair gossip with consensus for that.

**Practice drill**

Design membership for a 1,000-node cluster. Compare the message volume of all-to-all heartbeats with gossip at a fan-out of three, and estimate how many rounds a failure rumour needs to reach every node. Describe the merge rule and why it uses versions. Then trace a network partition splitting the cluster 600/400: what each side believes, what happens if partition ownership is decided from that view, and what you would use instead for ownership.

**Go deeper**

Gossip membership propagates cluster state through randomised pairwise exchanges, so every node learns who exists and who appears alive without a central authority.

**Its scaling property is the reason it exists.** All-to-all heartbeating costs traffic quadratic in cluster size, which is untenable past a few hundred nodes, while a central registry replaces that cost with a dependency that must be more available than everything depending on it. Gossip gives each node a constant workload — a few exchanges per interval — regardless of how large the cluster grows, and no node is required for the mechanism to function.

**Information spreads exponentially, which makes the logarithmic round count possible.** Each informed node informs several more, so the informed population multiplies every round and even very large clusters converge in a handful of intervals. This is why increasing fan-out has rapidly diminishing returns: going from three peers to six roughly doubles traffic to save one round, which is almost never the right trade.

**Merging must be version-based, because clocks disagree.** Deciding which of two views is newer by wall-clock time means a node with a fast clock can resurrect stale state and a node with a slow one can have its updates ignored. Monotonic version numbers per entry give a total order that depends on nothing external, and they also give a node a way to refute a rumour of its own death by incrementing past the version that declared it.

**Suspicion and indirect probing are what keep false positives manageable.** A single failed probe indicates only that one path failed, so asking several other nodes to probe before concluding anything separates a dead node from a broken link. Adding an intermediate suspect state gives the accused node a window to object, and that window converts most transient network trouble into a rumour that dies quietly rather than an eviction.

**Eventual convergence is the defining limitation.** For seconds after any change, different nodes hold different views, so one may route to a node another has already buried. That is acceptable for load balancing, health awareness and configuration, and it is unacceptable for anything requiring every participant to agree at the same instant — which is why deriving exclusive partition ownership from a gossip view produces two confident owners and divergent data.

**The mature pattern is gossip plus consensus, each doing what it is good at.** Gossip carries the high-volume, failure-tolerant information cheaply and keeps working on both sides of a partition, which is exactly the availability large clusters need. A small consensus group handles the few decisions that must have a single answer, refusing to proceed without a quorum. Systems that try to make gossip do the second job inherit split-brain; systems that try to make consensus do the first inherit a scaling ceiling and a hard dependency.

**Related patterns:** Heartbeat Failure Detection · Leader Election · Consistent Hashing · Anti Entropy

---

## Partitioning

### Consistent Hashing

*Map keys and nodes onto the same ring so adding or removing a node moves only its share of keys instead of remapping everything.*

> **When you hear…** A node pool that changes size · modulo hashing causing mass remapping · distributed caches and partitioned stores

**Flow:** `Hash keys to ring` → `Hash nodes to ring` → `Key goes clockwise` → `Node leaves` → `Only its keys move`

**The problem**

A cache is spread across ten servers by hashing the key modulo ten. Adding an eleventh server changes the divisor, so nearly every key now maps somewhere different — roughly 91% of the cache is instantly invalid, and the database receives the resulting miss storm.

The same arithmetic applies when a server fails. A pool that changes size for any reason experiences near-total remapping, which makes routine capacity changes and ordinary hardware failures equally catastrophic.

> **Stop mapping keys to positions in a list**  
> Modulo hashing makes every key's destination depend on the total number of nodes, so any change to that number changes almost every answer. If instead both keys and nodes are placed on a fixed circular space, a key belongs to whichever node it meets going clockwise — and removing a node only affects the keys that were pointing at it, because no other key's path changes.

**Mental model**

A circle of hash values. Nodes occupy positions on it; a key is owned by the first node found clockwise from the key's own position.

1. **Place nodes** — Hash each node identifier to a position on the ring, many times over.
2. **Place keys** — Hash each key to a position on the same ring.
3. **Walk clockwise** — The first node encountered owns the key.
4. **Remove a node** — Its keys pass to the next node clockwise; nothing else moves.
5. **Add a node** — It takes a contiguous arc from its clockwise successor; nothing else moves.

> **Without virtual nodes the distribution is badly uneven**  
> Ten randomly placed points on a circle do not divide it into ten equal arcs — some nodes receive several times the share of others, and removing a node hands its entire range to one unlucky successor. Assigning each physical node a hundred or more positions is what makes the distribution acceptably even, and it is not an optimisation but a requirement.

**How it works**

**The ring, and what each change costs**

```text
MODULO HASHING, 10 -> 11 nodes
  hash(key) % 10  vs  hash(key) % 11
  -> ~91% of keys change owner
  -> a near-total cache flush

CONSISTENT HASHING, 10 -> 11 nodes
  the new node claims one arc
  -> ~1/11 of keys move (about 9%)
  -> and ONLY from its clockwise neighbour

VIRTUAL NODES
  without: 10 nodes = 10 arcs, wildly uneven
    worst arc can be 3-5x the average
  with 150 virtual nodes each:
    1500 points around the ring
    -> arcs average out; typical imbalance < 5%
    -> a departing node's load spreads across many
       successors rather than landing on one

LOOKUP
  sorted array of virtual node positions
  binary search for the first position >= hash(key)
  wrap to index 0 if past the end
  -> O(log n), trivially fast

REPLICATION ON THE RING
  walk clockwise past the owner to the next N
  DISTINCT physical nodes
  -> distinct matters; three virtual nodes of the
     same machine is one copy, not three
```

1. **Use 100 to 200 virtual nodes per physical node** — Fewer produces uneven arcs; many more costs memory for little gain.
2. **Weight nodes by giving larger machines more positions** — It is the natural way to handle a heterogeneous fleet.
3. **Walk to distinct physical nodes for replicas** — Otherwise replicas can land on the same machine and provide no redundancy.
4. **Keep the ring definition identical everywhere** — Clients disagreeing about node positions route the same key to different places.
5. **Move data deliberately when the ring changes** — The ring says where a key belongs; it does not move anything.
6. **Consider bounded loads or rendezvous hashing** — They address the imbalance that virtual nodes only reduce.

**Where it still does not help**

```text
WHAT IT FIXES
  pool size changes no longer remap everything
  failures affect one node's share
  scaling up is incremental, not catastrophic

WHAT IT DOES NOT FIX
  HOT KEYS
    one key is 30% of traffic
    -> it hashes to exactly one node, always
    -> that node saturates; the ring is irrelevant
    -> needs key splitting or caching, not hashing

  LARGE PARTITIONS
    one tenant has 100x the data of others
    -> its keys still land wherever they hash
    -> uneven storage despite even key count

  COORDINATED MOVEMENT
    the ring decides ownership instantly
    the DATA has to be copied, which takes time
    -> during the transfer, who serves the key?
    -> needs explicit handoff or read-repair

RENDEZVOUS HASHING (HRW)
  for each key, compute hash(key, node) for every
  node, pick the highest
  -> no ring, no virtual nodes, naturally balanced
  -> O(n) per lookup instead of O(log n)
  -> often the better choice for small node counts
```

| Metric | Value | Note |
|---|---|---|
| Modulo, 10→11 | 91% move | cache flush |
| Consistent, 10→11 | ~9% move | **only one arc** |
| Virtual nodes | 100-200 each | for evenness |
| Not solved | hot keys | a different problem |

> **The ring assigns ownership instantly and moves no data**  
> The moment a node joins, the ring says it owns an arc — but the data for those keys is still on the previous owner and may take minutes to copy. Without an explicit handoff protocol, reads for that arc go to a node that has nothing, and the system either serves misses or returns empty results. Caches tolerate this because a miss is merely slow; stores do not, and need the new owner to serve from the old one until the transfer completes.

**Technologies**

| System | Use | Note |
|---|---|---|
| Memcached clients | Distributing keys across a cache pool | The original popular application |
| Cassandra and Dynamo-style stores | Token ring ownership | Virtual nodes are standard |
| Riak | Partition assignment | Fixed partition count mapped onto nodes |
| Load balancers | Sticky routing by key | Ring hashing as a balancing policy |
| Rendezvous hashing | The same goal, no ring | Better balance; O(n) lookups |
| Bounded-load consistent hashing | Caps per-node load | Overflows to the next node when full |

Bounded-load variants are worth knowing because they address the residual imbalance directly: each node has a capacity limit, and keys that would exceed it spill to the next node clockwise. This keeps the movement properties while preventing any single node from being handed a disproportionate share.

**Trade-offs**

**Key-to-node mapping schemes**

| Scheme | Keys moved on change | Balance | Lookup cost |
|---|---|---|---|
| Modulo hashing | Nearly all | Excellent | O(1) |
| Consistent hashing, no virtual nodes | 1/n | Poor | O(log n) |
| Consistent hashing with virtual nodes | 1/n | Good | O(log n) |
| Rendezvous hashing | 1/n | Excellent | O(n) |
| Fixed partitions mapped to nodes | Whole partitions | Good, controllable | O(1) |
| Central lookup table | Whatever you choose | Perfect | A lookup dependency |

Fixed partitions — creating, say, 1024 partitions once and assigning them to nodes — is what many production systems actually do. It keeps movement bounded, makes rebalancing explicit and inspectable, and avoids the ring entirely at the cost of choosing the partition count up front.

> **Ask before choosing it**  
> How often does the node set actually change, and what happens during the transfer? If the pool is stable and changes are planned, fixed partitions with an explicit assignment map are easier to reason about; the ring earns its place where membership changes frequently or automatically.

**How it fails**

**How consistent hashing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Severely uneven load | Too few virtual nodes | 100-200 positions per physical node |
| One node overwhelmed after a failure | Departing node's arc landing on one successor | Virtual nodes spread it across many |
| Replicas on the same machine | Walking to positions, not distinct nodes | Skip virtual nodes of the same physical host |
| Keys routed inconsistently | Clients with different ring definitions | Single source of truth for membership |
| Empty reads after scaling | Ownership moved before the data did | Handoff protocol or read from the old owner |
| A hot key still saturates one node | Hashing cannot split a single key | Split the key or cache it locally |
| Storage imbalance despite even keys | Key sizes vary enormously | Weight by data volume, not key count |

> **Even distribution of keys is not even distribution of load**  
> The ring balances how many keys each node owns, which is only a proxy for what actually matters — request volume and data size. One key receiving a third of all traffic, or one tenant holding a hundred times more data than others, produces a saturated node while the ring reports a perfect split. Diagnosing imbalance by key count when the real skew is in access or size is a persistent source of confusion.

**Where it is used**

- **Distributed caches**, the original motivation, where adding capacity must not invalidate everything.
- **Dynamo-style databases**, using a token ring with virtual nodes for both ownership and replica placement.
- **Load balancers with session affinity**, routing a user consistently to the same backend while the pool changes.
- **Content delivery networks**, choosing which edge cache holds which object.
- **Sharded application tiers**, where a request must reach the instance holding a particular in-memory state.

**In the interview**

Lead with the modulo arithmetic; it makes the motivation concrete in one line.

- **Quantify the problem first**: going from ten to eleven nodes remaps about 91% of keys under modulo hashing.
- **Give the consistent-hashing figure** — roughly one node's share moves — and say it comes from only one neighbour.
- **Insist on virtual nodes** with a number, and explain both effects: even arcs and spread-out failure impact.
- **Note that replicas must be distinct physical nodes**, since walking the ring naively can place copies on one machine.
- **Say what it does not solve.** Hot keys and uneven data sizes are separate problems the ring cannot address.

**Practice drill**

A cache pool grows from 10 to 11 nodes. Compute the fraction of keys remapped under modulo hashing and under consistent hashing, and say where the moved keys come from in each case. Choose a virtual node count and justify it by describing what happens with too few. Then explain what a client should do for keys in a newly assigned arc while the data is still being copied, and what changes if this is a persistent store rather than a cache.

**Go deeper**

Consistent hashing maps keys and nodes into the same circular space so that changing the node set relocates only a small, contiguous share of keys.

**It removes a coupling that modulo hashing creates.** When a key's destination is computed from the total node count, every key depends on that count, and changing it invalidates nearly every mapping at once. Placing nodes on a ring makes each key depend only on its own position and the nearest node clockwise, so a change affects the keys in one arc and leaves everything else untouched — which is what turns adding capacity from an incident into a routine operation.

**Virtual nodes are not a refinement; without them the scheme barely works.** A handful of randomly placed points divides a circle very unevenly, so some nodes receive multiples of the average load, and a departing node dumps its whole range onto a single successor. Giving each machine a hundred or more positions averages the arcs out and spreads a failure's load across many nodes, which is the behaviour people assume the ring provides by itself.

**Ownership and data movement are separate events, and conflating them causes outages.** The ring declares a new node the owner the instant it joins, while the data for that arc still lives on the previous owner and takes time to copy. Caches absorb this as extra misses; persistent stores cannot, and need an explicit handoff in which the new owner serves from the old one, or reads fall back, until the transfer completes.

**Balanced key counts do not imply balanced load.** The ring distributes keys, but traffic concentrates on popular keys and storage concentrates on large ones, so a node can be saturated while the distribution looks perfect. A single key receiving a large share of requests always lands on one node no matter how the ring is arranged, which means hot keys require splitting or local caching — a different pattern entirely, not a hashing parameter to tune.

**Replica placement needs an extra rule.** Walking clockwise for additional copies naturally encounters more virtual nodes, and several of them may belong to the same physical machine, producing replicas that all die together. Skipping to distinct hosts — and, in rack- or zone-aware systems, to distinct failure domains — is what makes the replication factor mean what it claims.

**Alternatives are often the better fit, and knowing why matters.** Rendezvous hashing achieves the same movement properties with better balance and no virtual nodes, at the cost of evaluating every node per lookup, which is fine for modest fleets. Fixed partitions assigned to nodes through an explicit map give bounded movement with an inspectable, controllable assignment, which many production systems prefer precisely because rebalancing becomes a deliberate operation rather than an emergent consequence of a hash function.

**Related patterns:** Sharding · Hot Key Mitigation · Gossip Membership · Quorum Replication

---

### Sharding

*Split data across independent databases by a chosen key, so capacity scales horizontally at the cost of cross-shard operations.*

> **When you hear…** One database cannot hold the data or serve the writes · vertical scaling exhausted · a natural partition key exists

**Flow:** `Choose shard key` → `Route by key` → `Independent shards` → `Cross-shard queries hard` → `Rebalance deliberately`

**The problem**

A single database has reached the largest instance available and is still saturating on writes. Read replicas absorb reads but every write still goes to one machine, and the working set no longer fits in memory, so latency has become unpredictable.

Splitting is the only remaining direction, and it is irreversible in practice. Once data lives in several places, transactions, joins and unique constraints that spanned the whole dataset no longer work, and the application has to change to match.

> **Sharding buys capacity by giving up the single-database guarantees**  
> Each shard is an ordinary database with ordinary performance; putting ten of them together gives roughly ten times the capacity. What is lost is everything that depended on all the data being in one place — cross-entity transactions, arbitrary joins, global uniqueness and global ordering. The shard key determines which queries stay easy and which become distributed problems, which makes it the most consequential decision in the design.

**Mental model**

Independent databases holding disjoint subsets of the data, with a routing rule mapping every request to exactly one of them.

1. **Choose the key** — Pick the attribute that partitions the data along the access pattern.
2. **Map** — Define how key values become shard assignments.
3. **Route** — Every query resolves its shard before executing.
4. **Isolate** — Each shard runs independently, with its own resources and its own failures.
5. **Rebalance** — Adding capacity moves ranges of keys, deliberately and slowly.

> **The shard key is chosen once and is extremely expensive to change**  
> Changing it means rewriting every row into a new arrangement while the system serves traffic, typically through a dual-write migration lasting weeks. Teams therefore live with an unsuitable key far longer than they should, working around it with cross-shard queries and application-level joins — which is why the time spent choosing it is the best-spent time in the project.

**How it works**

**Choosing the shard key**

```text
THE TEST
  which single value appears in almost every query?
  that is your candidate

  e-commerce   -> customer_id
  multi-tenant -> tenant_id
  messaging    -> conversation_id
  social       -> user_id

A GOOD KEY
  high cardinality       many distinct values
  even distribution      no value dominates
  present in most queries so routing is possible
  stable                 it never changes for a row

A BAD KEY
  status / country / type   low cardinality, skewed
  timestamp                 all writes hit one shard
  auto-increment id         same, and no locality

THE CONSEQUENCE OF THE CHOICE
  shard by customer_id:
    "orders for customer X"      one shard, fast
    "all orders today"           every shard, slow
  shard by order_date:
    "all orders today"           one shard, fast
    "orders for customer X"      every shard, slow
  -> you are choosing which query is cheap; you
     cannot have both

CO-LOCATION
  put related entities on the SAME shard by using
  the same key
    customer, their orders, their addresses
  -> keeps the common transaction local
```

1. **Choose the key from the dominant access pattern** — Everything else follows from it, and it is nearly immutable.
2. **Co-locate related entities under the same key** — It keeps the transactions you actually need inside one shard.
3. **Use hash or range mapping deliberately** — Hash distributes evenly; range keeps scans local and creates hot spots.
4. **Route through a layer, not through hardcoded connections** — Rebalancing requires the mapping to be changeable at runtime.
5. **Plan for cross-shard queries** — Some will exist; decide whether they scatter, or are served from a separate store.
6. **Choose the shard count with headroom** — Resharding is a migration; over-provisioning logical shards avoids most of them.

**Cross-shard operations, and the logical shard trick**

```text
WHAT BECOMES HARD
  joins across shards      application-side, or a
                           denormalised copy
  transactions across      saga or two-phase commit
  global uniqueness        a separate service, or a
                           key that includes the shard
  global ordering          per-shard only
  aggregate queries        scatter-gather, or a
                           warehouse

SCATTER-GATHER COST
  query 16 shards, wait for all
  latency = the SLOWEST shard, not the average
  -> component tail becomes system median
  -> fine occasionally, unusable as the main pattern

LOGICAL SHARDS  (the standard mitigation)
  create 1024 logical shards on day one
  map them to 4 physical databases (256 each)
  growth: move logical shards, not rows
    4 -> 8 databases: move 512 logical shards
  -> rebalancing becomes a file copy plus a mapping
     change, not a row-by-row migration
  -> the expensive resharding never happens

READ PATH FOR CROSS-SHARD NEEDS
  keep a denormalised, differently-keyed copy in a
  search index or warehouse
  -> the shards serve the operational path; the copy
     serves the analytical one
```

| Metric | Value | Note |
|---|---|---|
| Capacity | n× per shard | **linear** |
| Lost | joins, transactions | across shards |
| Key choice | near-permanent | choose carefully |
| Mitigation | logical shards | 1024 from day one |

> **Scatter-gather latency is governed by the slowest shard**  
> A query touching sixteen shards completes when the last one answers, so the 99th percentile of a single shard becomes roughly the median of the composed query. This is the same tail-at-scale arithmetic that makes fan-out expensive everywhere, and it means a query pattern that requires touching every shard is not merely slower — it is systematically unpredictable, and no amount of per-shard optimisation fixes it.

**Technologies**

| Approach | Examples | Note |
|---|---|---|
| Application-level sharding | Routing logic in the service | Full control; full responsibility |
| Sharding middleware | Vitess, Citus | Routing and rebalancing handled for you |
| Natively sharded databases | MongoDB, Cassandra, CockroachDB | Sharding is built in, with its own constraints |
| Managed distributed SQL | Spanner-style systems | Cross-shard transactions, at a cost |
| Functional partitioning | Split by service, not by key | Often the better first step |
| Do not shard | Bigger instance, read replicas, archiving | Frequently sufficient for years |

The last row is genuine advice. Modern single instances handle very large datasets and high write rates, and archiving cold data or moving one heavy table out often buys several years — during which the access patterns become clearer and the eventual shard key choice is better informed.

**Trade-offs**

**Scaling a database**

| Approach | Write capacity | Query flexibility | Operational cost |
|---|---|---|---|
| Bigger instance | Limited by hardware | Unchanged | Lowest |
| Read replicas | Unchanged | Unchanged, with replica lag | Low |
| Functional partitioning | Per service | Lost across services | Medium |
| Sharding by key | Linear with shards | Lost across shards | High |
| Distributed SQL | Linear | Mostly preserved | High, plus latency |
| Archiving cold data | Unchanged | Unchanged | Low |

Functional partitioning deserves attention before key-based sharding: moving one heavy domain into its own database often removes the pressure entirely, and it preserves query flexibility within each domain rather than fragmenting everything by a single key.

> **Ask before choosing it**  
> Which queries must remain fast, and does one key appear in all of them? If the important queries have no key in common, sharding will make half of them scatter-gather regardless of the choice — and that usually means the data should be split functionally, or served from a second, differently-keyed store.

**How it fails**

**How sharding goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| One shard saturated | Skewed shard key | Choose a high-cardinality even key; split hot values |
| All writes hitting one shard | Time-based or sequential key | Hash the key, or shard on something else |
| Most queries touch every shard | Key does not match access patterns | Re-evaluate the key; add a differently-keyed read store |
| Latency dominated by one slow shard | Scatter-gather as the main pattern | Denormalise so the common query is single-shard |
| Resharding takes months | Physical shards with no logical layer | Many logical shards mapped to few physical ones |
| Cross-shard transactions needed constantly | Related entities on different shards | Co-locate by using the same key |
| Unique constraints violated | Uniqueness enforced per shard only | Separate uniqueness service, or include the shard in the key |
| Rebalancing causes an outage | Moving data without a handoff | Dual-write, verify, then cut over |

> **A skewed shard key reproduces the original problem with more machines**  
> Sharding by a low-cardinality or unevenly distributed attribute concentrates a large fraction of traffic on one shard, which then saturates exactly as the single database did — except now there is routing, rebalancing and cross-shard complexity on top. The capacity gain is proportional to how evenly the key distributes, and a bad key delivers none of it while charging the full operational price.

**Where it is used**

- **Multi-tenant SaaS platforms**, sharding by tenant so each customer's data and transactions stay local.
- **Large e-commerce systems**, sharding by customer with orders and addresses co-located.
- **Messaging systems**, sharding by conversation so a thread's history and writes stay on one shard.
- **Vitess-style deployments**, where logical shards and a routing layer make rebalancing routine.
- **Time-series stores**, sharding by time range where recent data is hot and old data is archived.

**In the interview**

Spend the time on the key and on what becomes hard; the mechanics of routing are the easy part.

- **Derive the key from the dominant query**, and state explicitly which queries become expensive as a result.
- **Say the key is effectively permanent** and that changing it is a multi-week dual-write migration.
- **Propose logical shards from day one** — 1024 mapped onto a few databases — so rebalancing is a mapping change.
- **Do the scatter-gather arithmetic.** The slowest shard sets the latency, so fan-out queries are systematically unpredictable.
- **Suggest cheaper options first**: functional partitioning, archiving, a bigger instance. Sharding before those is usually premature.

**Practice drill**

An e-commerce database has outgrown its largest instance. Choose a shard key and justify it against the three most common queries, stating which becomes expensive. Design the logical-to-physical mapping and say how you would go from 4 databases to 8. Then handle two specific problems: a query for all orders placed today, and a transaction spanning two customers. Finally, state what you would try before sharding at all.

**Go deeper**

Sharding splits data across independent databases by a key, trading the guarantees of a single database for horizontal capacity.

**The shard key is the design, and everything else is consequence.** It determines which queries resolve to one shard and which must touch all of them, which entities can participate in a local transaction, and how evenly load distributes. Because changing it requires rewriting the entire dataset while serving traffic, the choice is effectively permanent, and teams routinely spend years working around a key that was picked in an afternoon.

**What is lost is specific and worth enumerating before committing.** Joins across shards become application-level work or require a denormalised copy; transactions spanning shards need sagas or a distributed protocol; global uniqueness and global ordering stop being available from the database. None of these are insurmountable, but each one is code that did not exist before, and the total is the real price of the capacity gained.

**Scatter-gather is the failure mode that makes a bad key visible.** A query touching every shard finishes when the slowest one answers, so the tail latency of a single shard becomes the typical latency of the query. Occasional fan-out is tolerable; fan-out as the dominant pattern produces a system whose response times are unpredictable in a way no per-shard tuning addresses, and it is the clearest signal that the key does not match how the data is actually read.

**Logical shards convert resharding from a migration into a configuration change.** Creating far more logical partitions than physical databases — a thousand or so mapped onto a handful — means growth moves whole partitions rather than individual rows, which can be done by copying files and updating a mapping. Systems built without this layer face a genuine data migration every time they add capacity, which is why that migration keeps being postponed past the point where it was needed.

**Even distribution is what the capacity gain is proportional to.** A key with low cardinality or heavy skew concentrates traffic on one shard, which saturates exactly as the original single database did, while the system now also carries routing, rebalancing and cross-shard complexity. Verifying the distribution of the proposed key against real data, before committing, is the step that separates a successful shard from an expensive one.

**Sharding should be a late decision, not an early one.** A single modern instance holds far more data and sustains far higher write rates than most teams assume, and archiving cold rows, moving one heavy table out, or splitting the database functionally by domain often buys several years. That time is valuable in itself: access patterns become clear, the dominant queries stabilise, and the shard key chosen afterwards is chosen with evidence rather than with a guess about how the product will be used.

**Related patterns:** Consistent Hashing · Hot Key Mitigation · Scatter Gather · Saga

---

### Hot Key Mitigation

*Spread or absorb traffic for a single dominant key, since partitioning by hash always sends one key to exactly one place.*

> **When you hear…** One partition saturated while others idle · a celebrity account or viral item · a counter every request touches

**Flow:** `One key dominates` → `Hashes to one node` → `Node saturates` → `Split or replicate key` → `Load spreads`

**The problem**

A social platform shards by user. One account with eighty million followers receives a thousand times the traffic of a typical user, and every one of those requests lands on the single shard that owns it. That shard is saturated while the other fifteen sit at ten percent.

Nothing about the partitioning is wrong. Hashing sends a key to one place by definition, which is precisely the property that makes routing work — and it means a single key's load is fundamentally unsplittable by the partitioning scheme itself.

> **If one key is too big, stop treating it as one key**  
> The load cannot be divided while the key is atomic, so the remedy is to make it non-atomic: split the key into variants that hash differently, replicate it so several nodes can serve it, or absorb the traffic before it reaches the partition at all. Each of these changes what the key means, which is why the right choice depends on whether the traffic is reads, writes, or both.

**Mental model**

Three distinct remedies chosen by traffic type: absorb reads in a cache, replicate reads across nodes, or split writes across sub-keys that are recombined on read.

1. **Detect** — Identify which keys are hot, continuously rather than once.
2. **Classify** — Determine whether the pressure is reads, writes, or storage.
3. **Absorb** — For reads, cache in front so most requests never reach the partition.
4. **Split** — For writes, shard the key itself into N sub-keys and aggregate on read.
5. **Adapt** — Hotness changes hourly; the mitigation must be applied and removed automatically.

> **Splitting a key destroys the atomicity that made it a single key**  
> A counter split into a hundred sub-counters can no longer be read atomically — the total is a sum of values read at slightly different moments, and it is never exactly right. For a view count this is irrelevant; for an inventory level that must never oversell it is unacceptable. The split is a trade of correctness properties, not a free optimisation.

**How it works**

**Remedies by traffic type**

```text
HOT READS
  1  local cache in every application instance
       TTL 1-5 s, no coordination
       -> 16 instances = 16x reduction, almost free
       -> serves slightly stale data
  2  replicate the key to every partition
       read from any, write to all
       -> reads scale perfectly, writes get expensive
  3  CDN or edge cache for anything public
       -> the request never reaches your system

HOT WRITES  (counters, rate limits, likes)
  split the key:
    counter:post:123        -> saturated
    counter:post:123:0..99  -> 100 sub-counters
    write: pick a random suffix, increment
    read:  sum all 100
  -> writes spread 100x
  -> reads cost 100 lookups and are approximate

HOT PARTITION FROM MANY KEYS
  not one key, but a tenant whose keys all land in
  one place
  -> include a spreading component in the key
  -> or give that tenant dedicated capacity

DETECTION
  sample request keys continuously
  heavy-hitter sketch (count-min) per partition
  -> alert when one key exceeds ~5% of a partition
```

1. **Cache hot reads locally before anything more elaborate** — A few seconds of in-process TTL removes most read pressure with no coordination.
2. **Split hot write keys into a fixed number of sub-keys** — Spreads writes by that factor; reads become an aggregation.
3. **Choose the split factor from the actual overload** — Too small does not help; too large makes every read expensive.
4. **Detect hotness continuously and automatically** — Yesterday's hot key is not today's, and manual lists are always stale.
5. **Apply and remove mitigation dynamically** — A permanent split penalises reads for a key that is no longer hot.
6. **Consider dedicated capacity for a permanently hot tenant** — Sometimes the honest answer is that one customer needs their own shard.

**What each remedy costs**

```text
SCENARIO
  post:123 receives 50,000 reads/s and 5,000
  writes/s; a shard handles 10,000 ops/s

LOCAL CACHE, 1 s TTL, 20 app instances
  reads reaching the shard: 20/s (one per instance
  per second)
  writes unchanged: 5,000/s -> still overloaded
  -> solves reads completely, writes not at all
  -> data up to 1 s stale

SPLIT WRITES INTO 10 SUB-KEYS
  each sub-key: 500 writes/s
  spread across 10 shards -> comfortable
  reads must sum 10 keys
  -> combine with the local cache: 20 reads/s x 10
     lookups = trivial
  -> total is eventually consistent

REPLICATE TO ALL 16 SHARDS
  reads: 50,000 / 16 = 3,125 per shard, fine
  writes: 5,000 x 16 = 80,000 total, far worse
  -> only sensible for read-heavy keys

THE COMBINATION THAT USUALLY WINS
  local cache for reads
  + key splitting for writes
  -> each mechanism applied to the traffic it suits
```

| Metric | Value | Note |
|---|---|---|
| Local cache | reads only | **seconds stale** |
| Key splitting | writes | approximate reads |
| Replication | read-heavy | writes multiply |
| Detection | continuous | hotness moves |

> **A static list of hot keys is obsolete before it is deployed**  
> Hotness is created by events — a post goes viral, a product is featured, a celebrity joins — and it shifts within hours. Hard-coding known hot keys means the mitigation protects yesterday's traffic while today's hot key saturates a shard unprotected. Detection has to be a running measurement feeding an automatic response, and the response must expire when the key cools or reads pay the aggregation cost forever.

**Technologies**

| Remedy | Mechanism | Note |
|---|---|---|
| Local in-process cache | Short TTL per instance | Cheapest and most effective for reads |
| Distributed cache in front | Shared cache layer | Still one key, so it can move the hot spot |
| Key splitting | Suffixed sub-keys aggregated on read | The standard remedy for hot writes |
| Full replication of hot items | Copy to every partition | Reads scale; writes multiply |
| Heavy-hitter detection | Count-min sketch, sampling | Cheap, continuous identification |
| Dedicated capacity | A shard per large tenant | Honest answer for permanent skew |

A shared distributed cache is a partial remedy worth flagging: the hot key remains a single key in that cache, so it can simply relocate the hot spot from the database to one cache node. Local caching avoids this because the key exists independently in every instance.

**Trade-offs**

**Hot key remedies**

| Remedy | Helps reads | Helps writes | Cost |
|---|---|---|---|
| Local cache with short TTL | Enormously | No | Staleness of a few seconds |
| Distributed cache | Yes | No | May move the hot spot |
| Key splitting | No | Yes, by the split factor | Approximate, more expensive reads |
| Replicate to all partitions | Yes | Makes worse | Write amplification |
| Dedicated shard for the key or tenant | Yes | Yes | Operational and routing complexity |
| Rate limit the hot key | Protects others | Protects others | Degrades that key deliberately |

Rate limiting the hot key is a legitimate last resort and is used in practice: deliberately degrading the one item that is overwhelming a shard preserves service for everything else, which is a better outcome than a shard that fails for all its keys.

> **Ask before choosing it**  
> Is the hot key read-heavy or write-heavy? The remedies are almost disjoint — caching does nothing for writes, splitting does nothing for reads — so answering this first prevents applying an elaborate mitigation to the wrong half of the traffic.

**How it fails**

**How hot key mitigation goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Mitigation protects the wrong key | Static hot-key list | Continuous detection driving automatic response |
| Reads become expensive everywhere | Key splitting left in place after cooling | Expire the split when traffic normalises |
| Hot spot moved, not removed | Distributed cache with the same single key | Local caches, or replicate the cached entry |
| Counter drifts or double counts | Split sub-counters summed inconsistently | Accept approximation, or do not split |
| Writes get worse | Replicated a write-heavy key | Split instead of replicating |
| Oversold inventory | Split a key that required atomicity | Do not split keys with correctness constraints |
| Shard still saturated | Skew from many keys, not one | Re-key the tenant, or give it dedicated capacity |

> **Splitting a key that carries a correctness constraint is how inventory oversells**  
> A stock level maintained as a hundred sub-counters cannot be checked atomically, so two requests can each read a total that says one item remains and both proceed. For view counts and likes the approximation is harmless and the split is the right answer; for anything that must never go negative, the atomicity is the feature and the correct remedies are caching, rate limiting or dedicated capacity — never division.

**Where it is used**

- **Social platforms with celebrity accounts**, where one profile's reads dwarf the median by orders of magnitude.
- **E-commerce flash sales**, where a single product becomes the entire write load for a shard.
- **View and like counters**, the canonical use of key splitting with approximate reads.
- **Rate limiters**, where a single API consumer's counter concentrates on one node.
- **Multi-tenant platforms**, where a large customer's keys collectively overload one partition.

**In the interview**

Say early that partitioning cannot help here; that framing is what makes the rest of the answer coherent.

- **State the constraint**: a single key hashes to one node, so no partitioning scheme can split its load.
- **Separate read remedies from write remedies** and note they are almost disjoint.
- **Offer local caching first for reads** — a one-second TTL across twenty instances removes nearly all of it.
- **Describe key splitting for writes** with a concrete factor, and name what it costs: approximate, more expensive reads.
- **Require continuous detection** and automatic expiry, since hotness shifts within hours and static lists protect the wrong thing.

**Practice drill**

A post receives 50,000 reads and 5,000 writes per second on a shard that handles 10,000 operations per second. Choose remedies for each half of the traffic, with concrete parameters, and compute the resulting load on the shard. Say what correctness property you gave up and whether it matters for a view counter. Then redo the analysis for an inventory count that must never oversell, and explain which remedies are no longer available.

**Go deeper**

Hot key mitigation addresses load concentrated on a single key, which partitioning cannot spread because a key maps to one place by construction.

**The constraint is structural, not a tuning failure.** Hashing exists to send a given key consistently to one node, and that property is what makes routing possible at all. A key receiving a disproportionate share of traffic therefore saturates its node regardless of how many nodes exist or how evenly the ring is arranged, which is why adding capacity does nothing and why the remedy has to change the key rather than the partitioning.

**Read and write remedies are almost entirely separate.** Caching removes read pressure and has no effect on writes; splitting a key spreads writes and makes reads more expensive; replicating to every partition scales reads and multiplies writes. Establishing which half of the traffic is the problem before choosing is what prevents deploying an elaborate mechanism against the wrong bottleneck, and systems with both hot reads and hot writes usually need one of each.

**Short-lived local caching is the highest-leverage read remedy.** A one-second TTL in each application instance reduces the shard's read load by the instance count, requires no coordination, invalidation or shared infrastructure, and costs only a second of staleness on a value that is changing constantly anyway. A shared distributed cache is a weaker answer for this specific problem, because the hot key remains one key in that cache and can simply relocate the hot spot.

**Key splitting trades atomicity for write throughput, and the trade is not always acceptable.** A counter divided into sub-counters spreads writes by the split factor while making the total a sum of values read at different instants — never precisely correct. That is entirely fine for views and likes, where nobody can tell and nothing depends on exactness, and it is wrong for a stock level, where the atomic check is the mechanism preventing overselling. Recognising which constraint the key carries decides whether splitting is available.

**Hotness is transient, so the mitigation must be too.** Keys become hot because of events that pass within hours, and a hard-coded list protects last week's traffic while today's viral item saturates a shard unprotected. Detection needs to be a continuous measurement — sampling with a heavy-hitter sketch is cheap enough to run always — feeding an automatic response, and that response has to expire, or reads keep paying the aggregation cost for keys that cooled long ago.

**Some skew is permanent and deserves an honest answer.** A tenant genuinely a hundred times larger than the median, or an account that will always be extraordinary, is not a transient anomaly to be smoothed; dedicated capacity or a shard of their own is simpler and more predictable than an ever more elaborate mitigation. Deliberately rate limiting the hot key is also legitimate at the extreme, because degrading one item preserves service for every other key sharing that partition, which is the better of two bad outcomes.

**Related patterns:** Consistent Hashing · Sharding · Cache Aside · Token Bucket

---

## Replication

### Quorum Replication

*Write to a majority of replicas and read from a majority, so any read is guaranteed to see the most recent write without contacting them all.*

> **When you hear…** Replicated data that must not serve stale reads · availability required despite replica failures · tunable consistency needed per operation

**Flow:** `N replicas` → `Write waits for W` → `Read waits for R` → `R plus W over N` → `Overlap guarantees freshness`

**The problem**

Data is stored on five replicas. Waiting for all five on every write means one slow or failed replica blocks all writes; waiting for just one means a subsequent read may reach a replica that never received it and return stale data.

Neither extreme is acceptable. Full agreement gives consistency and no availability; single-replica acknowledgement gives availability and no guarantee about what a read will see.

> **Overlapping majorities make agreement unnecessary**  
> If a write is acknowledged by more than half the replicas and a read consults more than half, the two sets must share at least one replica — and that shared replica has the latest write. Freshness comes from set intersection rather than from contacting everyone, which means some replicas can be down or slow without affecting correctness at all.

**Mental model**

Three numbers: N replicas, W acknowledgements required for a write, R responses required for a read. When R plus W exceeds N, the read and write sets must overlap.

1. **Write** — Send to all N, return once W have acknowledged.
2. **Read** — Query enough replicas to collect R responses.
3. **Compare** — Use version information to determine which response is newest.
4. **Return** — Serve the newest value found among the R.
5. **Repair** — Update the replicas that were behind, so the divergence does not persist.

> **The overlap guarantees the latest value is present, not that it is chosen**  
> A quorum read reaches at least one replica holding the newest write — but it also reaches replicas holding older versions, and something must decide which is newest. That requires version vectors or timestamps carried with the data. A system that returns the first response, or resolves conflicts by wall-clock time across skewed machines, has the overlap and still serves stale or wrong values.

**How it works**

**Choosing N, W and R**

```text
THE RULE
  R + W > N   guarantees the read set overlaps the
              write set

N = 3
  W=2, R=2   R+W=4 > 3   balanced, the usual default
             tolerates 1 failure for both
  W=3, R=1   R+W=4 > 3   fast reads, writes fail if
             any replica is down
  W=1, R=3   R+W=4 > 3   fast writes, reads need all
  W=1, R=1   R+W=2 < 3   NO guarantee; eventual only

N = 5
  W=3, R=3   tolerates 2 failures on each side
  W=4, R=2   read-optimised
  W=2, R=4   write-optimised

LATENCY
  a quorum operation is as slow as the Wth fastest
  replica, not the slowest
  -> N=5, W=3 waits for the 3rd of 5 responses
  -> this is why quorums beat full replication on
     tail latency

AVAILABILITY
  writes survive N - W failures
  reads survive N - R failures
  -> W=2, R=2, N=3 survives exactly one failure
```

1. **Default to N=3 with W=2 and R=2** — It tolerates one failure on both paths and is the balanced choice.
2. **Tune W and R per operation where the store allows it** — Some reads need freshness; many can accept a faster, weaker read.
3. **Carry version information with every value** — Overlap only helps if the newest version can be identified.
4. **Repair stale replicas on read** — Otherwise divergence persists and accumulates.
5. **Spread replicas across failure domains** — Three replicas in one rack tolerate one disk failure, not one rack failure.
6. **Understand that quorum is not a transaction** — Concurrent writes can both succeed and produce a conflict to resolve.

**What a quorum does not give you**

```text
SLOPPY QUORUM
  the designated replicas are unreachable
  -> write to ANY W reachable nodes instead
  -> the write succeeds, availability preserved
  -> BUT it did not go to the real replicas, so
     R + W > N no longer holds
  -> hinted handoff later moves it home
  -> during that window, reads can be stale

CONCURRENT WRITES
  two clients write different values at the same
  time, both reach quorums
  -> both succeed
  -> the versions conflict
  -> resolution: last-write-wins (loses data), or
     version vectors plus application merge

LAST-WRITE-WINS HAZARD
  decided by wall-clock timestamps
  clock skew of 50 ms between nodes
  -> the objectively later write can lose
  -> silently, with no error anywhere

WHAT QUORUMS DO NOT DO
  no atomic multi-key operations
  no read-your-writes across different clients
  no ordering between keys
  -> for those, consensus (Raft/Paxos) is required,
     which is a stronger and more expensive tool
```

| Metric | Value | Note |
|---|---|---|
| Rule | R + W > N | **overlap** |
| Default | 3 / 2 / 2 | one failure tolerated |
| Latency | Wth fastest | not the slowest |
| Not provided | atomicity | use consensus |

> **Sloppy quorums trade the guarantee for availability, quietly**  
> When the proper replicas are unreachable, some systems accept the write on whichever nodes are available so the operation succeeds. This keeps writes flowing during a partition and means the arithmetic no longer holds — the write is not on the replicas a future read will consult, so a quorum read can miss it entirely. It is a deliberate and often correct choice, but a system configured for strong reads that silently degrades to sloppy quorums is not providing what its configuration claims.

**Technologies**

| System | Approach | Note |
|---|---|---|
| Cassandra | Tunable per query | ONE, QUORUM, ALL selectable per statement |
| DynamoDB | Eventually or strongly consistent reads | The quorum is hidden behind two options |
| Riak | Explicit N, R, W | Dynamo's model exposed directly |
| MongoDB | Write and read concerns | Majority is the quorum equivalent |
| etcd and ZooKeeper | Consensus, not quorum voting | Stronger: ordering and atomicity too |
| Single primary with replicas | Primary decides | Simpler; failover is the hard part |

The distinction between the last two rows matters in discussion: consensus systems also use majorities, but they order operations through a leader, which gives atomicity and linearizability that plain quorum voting does not — at the cost of the leader being a bottleneck and a failover point.

**Trade-offs**

**Replication acknowledgement strategies**

| Strategy | Consistency | Write availability | Latency |
|---|---|---|---|
| Write all, read one | Strong | Poor; any failure blocks | Slowest replica |
| Write one, read all | Strong | Excellent | Reads slowest |
| Quorum W=R=majority | Strong enough | Tolerates a minority down | Middle replica |
| Write one, read one | Eventual | Excellent | Fastest |
| Sloppy quorum | Weakened | Highest | Fast |
| Consensus | Linearizable, ordered | Tolerates a minority down | Leader round trip |

Write-one-read-one is a legitimate choice for data where staleness is harmless — metrics, activity feeds, caches — and it is substantially faster and more available than any quorum. The mistake is applying it uniformly to data where a stale read has consequences.

> **Ask before choosing it**  
> Does every read need the latest value, or only some of them? Tuning per operation is usually available and frequently better than one setting for the whole store, since most systems have a small set of reads that require freshness and a large set that does not.

**How it fails**

**How quorum replication goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Stale reads despite a quorum | R + W not greater than N | Check the arithmetic against the configuration |
| Stale reads with correct arithmetic | Sloppy quorum during a partition | Know when the store degrades; account for it |
| Writes lost silently | Last-write-wins with skewed clocks | Version vectors, or an application merge |
| Replicas diverge permanently | No read repair or anti-entropy | Repair on read; run background reconciliation |
| All replicas lost together | Replicas in one failure domain | Spread across racks and zones |
| Latency worse than expected | W or R set equal to N | Use a majority, not everything |
| Concurrent writes produce surprises | Expecting transactional behaviour | Quorums are not transactions; use consensus |

> **Last-write-wins discards data according to clock skew**  
> When conflicting versions are resolved by comparing wall-clock timestamps, the winner is decided by whose machine had the later clock rather than by what actually happened last. A few tens of milliseconds of skew is enough for a genuinely later write to lose, and the loss is silent — no error, no conflict surfaced, just a value that quietly reverts. Version vectors detect the conflict instead, which lets the application decide rather than the clocks.

**Where it is used**

- **Dynamo-style databases**, where tunable N, R and W are the core of the consistency model.
- **Cassandra deployments**, choosing consistency per query so critical reads are strong and bulk reads are fast.
- **Object stores**, acknowledging writes once a durable majority of copies exist.
- **Distributed caches with replication**, trading read consistency for latency deliberately.
- **Consensus-backed configuration stores**, which use majorities plus ordering for stronger guarantees.

**In the interview**

The arithmetic is quick to state; spend the time on what overlap does not give you.

- **State R + W > N and why it works** — the sets must intersect, so a read touches a replica with the newest write.
- **Give a default and its failure tolerance**: N=3, W=2, R=2 survives one replica being down on both paths.
- **Note the latency property.** A quorum waits for the Wth fastest replica, which is why it beats waiting for all.
- **Raise conflict resolution.** Overlap finds the newest version only if versions are comparable; last-write-wins loses data under skew.
- **Distinguish quorum from consensus**: no atomicity, no ordering across keys, and sloppy quorums can void the guarantee entirely.

**Practice drill**

Design replication for a user profile store with five replicas. Choose W and R, verify the overlap condition, and state how many replica failures each path tolerates. Then work through two scenarios: two clients writing different values concurrently, and a partition in which the store falls back to a sloppy quorum. Say what a reader can observe in each case, and what mechanism you would add so the divergence does not persist.

**Go deeper**

Quorum replication guarantees fresh reads by requiring the read and write sets to overlap, rather than by contacting every replica.

**The guarantee comes from set intersection, which is why it tolerates failures.** If more than half the replicas acknowledged the write and more than half are consulted on read, at least one replica is in both sets and holds the newest value. Nothing needs to coordinate, no replica is essential, and the operation proceeds while a minority is down — properties that waiting for all replicas cannot provide.

**It also improves tail latency, which is an underrated benefit.** A write that waits for every replica is as slow as the slowest, so one degraded machine sets the latency for all operations. A quorum waits for the Wth fastest response and simply leaves the stragglers behind, which means transient slowness on a minority of replicas never reaches the client at all.

**Overlap locates the newest value; it does not identify it.** Among the R responses are several versions, and something must determine which is current. That requires version information carried with the data — logical versions or version vectors — because deciding by wall-clock timestamp means clock skew picks the winner. Last-write-wins configured over skewed clocks discards genuinely newer writes silently, which is among the most confusing data-loss modes a system can have.

**Sloppy quorums preserve availability by abandoning the arithmetic.** When the designated replicas are unreachable, accepting the write on whatever nodes are available keeps the system writable during a partition, and the write is then not where a future quorum read will look. This is frequently the right operational choice, and it means the configured consistency is a best case rather than a guarantee — so a design claiming strong reads must also know when its store silently degrades.

**Tuning per operation is usually the right granularity.** Most systems have a small number of reads that genuinely need the latest value and a large number that do not, so a single store-wide setting either pays for consistency everywhere or provides it nowhere. Choosing per query costs a little care at each call site and avoids both the latency of unnecessary strong reads and the surprises of accidentally weak ones.

**A quorum is weaker than consensus, and the gap is where designs go wrong.** Overlapping majorities give per-key freshness and nothing more: no atomic multi-key operations, no ordering between keys, and concurrent writes that both succeed and leave a conflict behind. Systems needing a single agreed sequence of operations — leader election, configuration, anything with invariants spanning keys — require a consensus protocol, which uses majorities too but adds the ordering that makes those guarantees possible, at the cost of a leader and its failover.

**Related patterns:** Read Repair · Anti Entropy · Geographic Replication · Leader Election

---

### Geographic Replication

*Keep copies of data in several regions so reads are local and a region failure is survivable, accepting that writes cannot be both fast and globally consistent.*

> **When you hear…** Users spread across continents · a region outage must not be an outage · data residency requirements

**Flow:** `Regions worldwide` → `Local reads fast` → `Writes cross oceans` → `Choose consistency` → `Handle conflicts`

**The problem**

A service hosted in one region serves users in Sydney with a round trip of roughly 250 milliseconds before any work is done. Several dependent requests turn that into seconds of waiting caused entirely by the speed of light.

A single region is also a single failure domain. When it becomes unavailable — and every region eventually does — the whole service is unavailable, regardless of how well it is engineered internally.

> **Distance is a fixed cost that can be paid on writes or on reads, but not avoided**  
> Data in one place means somebody is always far away. Copying it to several regions makes reads local everywhere, and moves the problem to writes: either a write waits for distant regions to acknowledge, which makes it slow but consistent, or it returns locally and propagates afterwards, which makes it fast and allows conflicting concurrent writes. Every geo-replication design is a choice between those two costs.

**Mental model**

Full or partial copies of the data in each region, with a replication topology deciding where writes are accepted and how they reach everywhere else.

1. **Place** — Decide which regions hold copies, and of what data.
2. **Route** — Send each user to the nearest region that can serve them.
3. **Accept writes** — In one region only, or in any region.
4. **Propagate** — Ship changes across regions, synchronously or asynchronously.
5. **Reconcile** — Detect and resolve conflicts when more than one region accepts writes.

> **Cross-region latency is physics, not a tuning parameter**  
> London to Sydney is about 250 milliseconds round trip at realistic fibre speeds, and no amount of engineering reduces it. A synchronous write requiring acknowledgement from both adds that to every write, permanently. Designs that assume cross-region calls can be optimised into insignificance are budgeting for a latency that cannot be achieved.

**How it works**

**Topologies and what each costs**

```text
SINGLE WRITE REGION  (primary-secondary)
  writes -> one region, replicated out async
  reads  -> local everywhere
  + no conflicts, ever
  + simple mental model
  - distant users see slow writes (250 ms+)
  - primary region failure means no writes until
    failover
  -> the right default for most systems

MULTI-REGION WRITES, ASYNCHRONOUS
  every region accepts writes locally
  + fast writes everywhere
  + survives any region failure
  - CONCURRENT CONFLICTING WRITES are now possible
  - needs CRDTs, version vectors, or an
    application merge rule

SYNCHRONOUS ACROSS REGIONS
  write waits for a quorum spanning regions
  + strongly consistent globally
  - every write pays inter-region latency
  - a region partition blocks writes
  -> reserve for data where consistency is worth
     hundreds of milliseconds

PARTITIONED BY GEOGRAPHY
  European users' data lives in Europe
  each region is authoritative for its own users
  + local reads AND local writes
  + satisfies data residency rules
  - cross-region access is slow but rare
  -> often the best answer available
```

1. **Start with one write region and local read replicas** — It removes conflicts entirely and solves the read latency, which is usually most of the problem.
2. **Partition by user geography where the data allows** — Local reads and local writes, with cross-region access as the rare case.
3. **Reserve synchronous cross-region writes for data that requires it** — The latency is permanent; pay it only where consistency is worth it.
4. **Design conflict resolution before enabling multi-region writes** — Conflicts are certain, and deciding afterwards means data has already been lost.
5. **Measure replication lag continuously** — It determines how much data a region failure loses.
6. **Practise regional failover** — An untested failover path fails at the moment it is needed.

**Failover, lag and what is lost**

```text
ASYNC REPLICATION LAG
  typical 50-500 ms, seconds under load
  region fails -> everything not yet replicated is
  LOST
  -> RPO (recovery point objective) = the lag
  -> "we lose up to 2 seconds of writes" is a
     product decision, not just a technical one

SYNC REPLICATION
  RPO = 0, nothing is lost
  cost: every write pays cross-region latency
  -> and a partition means writes stop

FAILOVER MECHANICS
  detect        30-60 s (and false positives are
                catastrophic here)
  promote       seconds
  redirect DNS  60-300 s depending on TTL
  -> realistic RTO is minutes, not seconds

THE SPLIT-BRAIN RISK
  network partition between regions
  both promote themselves to primary
  both accept writes
  -> divergent histories that must be merged by hand
  -> prevention: promotion requires a quorum of
     regions, or a witness in a third region

DATA RESIDENCY
  some data legally may not leave a jurisdiction
  -> geographic partitioning is not an optimisation
     but a requirement
```

| Metric | Value | Note |
|---|---|---|
| Cross-region RTT | ~250 ms | **unavoidable** |
| Async RPO | lag, seconds | data lost on failure |
| Sync RPO | zero | latency on every write |
| Realistic RTO | minutes | DNS dominates |

> **Two regions both promoting themselves is the worst outcome available**  
> A partition between regions leaves each side unable to tell whether the other is down or merely unreachable, and a failover policy that promotes on local judgement will promote both. Both then accept writes to the same logical dataset, producing two divergent histories that must eventually be merged manually with real data loss. Promotion must require agreement — a quorum of regions, or an independent witness — rather than being a decision either region can take alone.

**Technologies**

| Approach | Examples | Note |
|---|---|---|
| Async cross-region replicas | Most managed databases | Simple; non-zero data loss on failover |
| Globally distributed SQL | Spanner-style systems | Strong consistency, at the cost of write latency |
| Multi-region active-active stores | DynamoDB global tables, Cassandra | Fast local writes; conflict resolution required |
| CRDTs | Collaborative and offline-first systems | Conflicts merge automatically by construction |
| CDN and edge caching | Read-only content | Removes most of the latency problem where applicable |
| Geographic partitioning | Per-region authoritative data | Best latency and residency compliance |

Edge caching is the cheapest large win and the most overlooked in this conversation: a substantial fraction of what makes distant users wait is static or cacheable, and serving it from an edge location removes that latency without touching the database topology at all.

**Trade-offs**

**Geographic strategies**

| Strategy | Read latency | Write latency | Conflicts | Data loss on failure |
|---|---|---|---|---|
| Single region | Poor for distant users | Poor for distant users | None | Total outage |
| Single write region, read replicas | Good | Poor for distant users | None | Replication lag |
| Multi-region async writes | Good | Good | Certain | Replication lag |
| Multi-region sync writes | Good | Poor everywhere | None | None |
| Geographic partitioning | Good | Good | Rare | Per-region |

Geographic partitioning is the strongest row and is available more often than people assume: most user data is accessed overwhelmingly by users in one region, so making each region authoritative for its own users gives local reads and writes with conflicts only in the rare cross-region case.

> **Ask before choosing it**  
> How much data may be lost if a region disappears right now? That single number chooses between asynchronous and synchronous replication, and it is a product decision rather than a technical one — which means it should be asked of the business, not assumed by the engineer.

**How it fails**

**How geographic replication goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Split brain across regions | Local promotion decisions | Require quorum or a third-region witness |
| Writes lost on failover | Asynchronous replication lag | Accept a defined RPO, or replicate synchronously |
| Failover takes far too long | DNS TTL and manual steps | Short TTLs, automated and rehearsed failover |
| Conflicting writes silently resolved | Last-write-wins across regions | Version vectors or CRDTs |
| Every write pays ocean latency | Synchronous replication applied to everything | Reserve it for data that needs it |
| Compliance violation | Data replicated outside its jurisdiction | Geographic partitioning with enforced boundaries |
| Failover never worked | Path untested | Regular drills, including full region loss |

> **Asynchronous replication means a defined amount of data is lost when a region fails**  
> The replication lag at the moment of failure is exactly the window of writes that no longer exist anywhere. Users received success responses for them. Whether that is acceptable — two seconds of orders, a minute of profile edits — is a business decision that must be made explicitly, because the alternative costs cross-region latency on every write, and choosing by default means choosing without anyone having agreed to the loss.

**Where it is used**

- **Global consumer applications**, replicating read paths to every region while writes go home.
- **Financial systems with residency rules**, where geographic partitioning is a legal requirement rather than a performance choice.
- **Globally distributed SQL deployments**, paying write latency in exchange for consistent global reads.
- **Collaborative editors**, using CRDTs so concurrent regional writes merge without a coordinator.
- **Content delivery networks**, which solve the read half of the problem for everything cacheable.

**In the interview**

Anchor the discussion in the latency number and in the data-loss question; both make the trade concrete.

- **State the physical cost** — roughly 250 milliseconds across the world — and note it cannot be engineered away.
- **Ask how much data may be lost on a region failure**, because that chooses between async and sync replication.
- **Propose geographic partitioning** where access is regional, since it gives local reads and writes with rare conflicts.
- **Raise split brain unprompted** and require quorum-based promotion rather than a local decision.
- **Give realistic failover numbers.** Detection plus promotion plus DNS means minutes, and saying seconds signals inexperience.

**Practice drill**

Users are in North America, Europe and Asia. Design the replication topology, naming where writes are accepted and how reads are served in each region. State the recovery point and recovery time objectives your design achieves, with the arithmetic behind them. Then handle a partition between Europe and North America: say what each region does, what a user in each sees, and what mechanism prevents both from promoting themselves.

**Go deeper**

Geographic replication places copies of data in several regions so that reads are local and a regional failure is survivable.

**The read half is straightforward and the write half is the whole design.** Local replicas remove hundreds of milliseconds from every read for distant users, at no cost to consistency if writes still go to one place. Everything difficult follows from deciding where writes are accepted: one region means distant users wait, several regions mean concurrent conflicting writes, and requiring all regions to agree means every write pays the distance.

**Inter-region latency is a physical constant and budgets must respect it.** A round trip across the world is roughly a quarter of a second at realistic speeds, so a synchronous cross-region write cannot be fast, ever. Designs that treat this as an optimisation problem waste effort; designs that accept it choose deliberately where to spend it, and usually conclude that only a small subset of data justifies synchronous global agreement.

**Asynchronous replication converts a region failure into a defined quantity of lost data.** Whatever had not yet replicated when the region disappeared is gone, despite users having received success responses for it. The size of that window is the replication lag, and whether it is acceptable is a product question — losing two seconds of orders is a business decision, not a technical detail — which means it should be stated explicitly rather than inherited from a default configuration.

**Geographic partitioning is frequently the best answer and is underused.** Most user data is accessed almost entirely from one region, so making each region authoritative for its own users delivers local reads and local writes with conflicts confined to genuinely cross-region access. It also satisfies data residency requirements structurally rather than through policy, which matters wherever data legally may not leave a jurisdiction.

**Split brain across regions is the most damaging failure in the space.** A partition leaves each side unable to distinguish the other's failure from its own isolation, and a promotion policy decided locally will promote both. Two regions then accept writes to the same dataset and produce divergent histories that must be reconciled by hand with real loss. Requiring a quorum of regions, or an arbitrating witness in a third location, is what makes promotion a decision no single region can take alone.

**Failover is a practised operation or it is a theoretical one.** Detection, promotion, and DNS propagation compose into minutes rather than seconds, and every untested step is a place where the real event diverges from the plan. Regular drills that actually remove a region are what turn a documented recovery objective into an achievable one — and they routinely reveal that the recovery time everyone assumed was several times optimistic.

**Related patterns:** Quorum Replication · Read Repair · Anti Entropy · CDN

---

### Read Repair

*Detect stale replicas during ordinary reads and update them, so the act of reading gradually repairs divergence at no extra cost.*

> **When you hear…** Replicas drift after failures or dropped writes · eventual consistency that must actually converge · repair without a separate process

**Flow:** `Read several replicas` → `Compare versions` → `Return newest` → `Push to stale ones` → `Divergence shrinks`

**The problem**

A write reaches two of three replicas because the third was briefly unavailable. The quorum succeeded and the client was told the write was durable, but one replica now holds an older value indefinitely — nothing in the write path will ever revisit it.

Divergence accumulates. Every dropped message, brief partition and restart leaves another replica slightly behind, and without a mechanism to converge, the probability that a read encounters stale data grows over time.

> **Reads already fetch several copies, so they already know who is wrong**  
> A quorum read contacts multiple replicas and compares their versions to decide what to return. That comparison identifies exactly which replicas are behind, at no additional cost — the information is a by-product of work the read was doing anyway. Sending the newest value back to the stale ones turns every read into an opportunity for repair.

**Mental model**

Repair as a side effect of reading. The read collects versions, determines the newest, serves it, and pushes it to whichever replicas were behind.

1. **Fetch** — Read from several replicas as the consistency level requires.
2. **Compare** — Use version information to find the newest value.
3. **Serve** — Return the newest to the client immediately.
4. **Repair** — Send it to the replicas that had older versions.
5. **Converge** — Frequently read keys stay consistent without any background work.

> **It only repairs what is read, which is the opposite of what needs repairing**  
> Popular keys are read constantly and repair themselves within seconds. Rarely read keys — old records, cold data, the long tail — can stay divergent indefinitely, and those are precisely the ones where a replica failure is most likely to lose the only current copy. Read repair is a necessary mechanism and never a sufficient one.

**How it works**

**Blocking and background repair**

```text
BLOCKING READ REPAIR
  read R replicas
  versions differ
  -> push the newest to the stale ones
  -> WAIT for acknowledgement
  -> then return to the client
  + guarantees monotonic reads afterwards
  - adds latency to the read that found the problem

BACKGROUND READ REPAIR
  return to the client immediately
  repair asynchronously
  + no added latency
  - a concurrent read may still see the stale value
  -> the common default

PROBABILISTIC READ REPAIR
  even when the consistency level does not require
  reading all replicas, read the extras on a small
  percentage of requests (e.g. 10%)
  -> catches divergence on replicas the read would
     not otherwise have consulted
  -> cost is proportional to the sampling rate

WHAT IT NEEDS
  versions that can be compared reliably
    logical versions, or version vectors
  NOT wall-clock timestamps across skewed machines
  -> otherwise repair can push an OLDER value over a
     newer one, actively corrupting data
```

1. **Repair in the background by default** — Blocking repair adds latency to exactly the reads that were already slow.
2. **Use blocking repair where monotonic reads matter** — It guarantees a subsequent read will not see the older value again.
3. **Sample extra replicas occasionally** — Divergence on replicas the read never consults is otherwise invisible.
4. **Compare logical versions, not wall clocks** — Skew can make repair overwrite a newer value with an older one.
5. **Pair with anti-entropy for cold data** — Unread keys are never repaired by reads, by definition.
6. **Monitor how often repairs occur** — A rising repair rate is an early signal of replication trouble.

**What it covers and what it misses**

```text
COVERAGE BY ACCESS FREQUENCY
  read 1000x/day  -> repaired within seconds of any
                     divergence
  read 1x/week    -> divergent for up to a week
  read never      -> divergent forever

-> the keys most likely to be lost are the ones
   least likely to be repaired

WHY THAT MATTERS
  replica A holds the only current copy of a cold key
  replica A dies
  -> the value silently reverts to the older version
     on B and C
  -> nobody notices, because nobody reads it
  -> until someone does

THE PAIRING
  read repair      fast, free, covers hot data
  anti-entropy     slow, costly, covers everything
  hinted handoff   covers writes missed during a
                   known outage
  -> all three together give convergence with
     reasonable cost; any one alone leaves a gap

COST OF PROBABILISTIC REPAIR
  10% sampling on a read-heavy system
  -> 10% more replica reads
  -> catches cold-ish keys that get some traffic
  -> still misses truly cold data
```

| Metric | Value | Note |
|---|---|---|
| Cost | near zero | reads already fetch |
| Coverage | read keys only | **cold data missed** |
| Default mode | background | no added latency |
| Requires | comparable versions | not wall clocks |

> **Repairing on wall-clock timestamps can overwrite newer data with older**  
> If versions are ordered by the clock of whichever node wrote them, a node running fast can stamp an older value with a later time, and read repair will then dutifully propagate that older value over the newer one on every other replica. The mechanism designed to converge on the correct value converges on the wrong one, and it does so permanently and silently.

**Technologies**

| System | Behaviour | Note |
|---|---|---|
| Cassandra | Blocking and background read repair | Repair chance is configurable per table |
| Dynamo-style stores | Read repair as described | Introduced alongside anti-entropy in the original design |
| Riak | Read repair on divergence | Version vectors make comparison safe |
| Object stores | Repair on access | Often paired with background scrubbing |
| Anti-entropy | Background comparison | Covers what reads never touch |
| Hinted handoff | Replay of writes missed during an outage | Covers a known window rather than arbitrary drift |

Hinted handoff is the third member of this family and is worth distinguishing: when a replica is known to be down, the coordinator stores the writes it missed and replays them on recovery. That handles planned and detected outages, while read repair and anti-entropy handle drift nobody noticed.

**Trade-offs**

**Convergence mechanisms**

| Mechanism | Cost | Coverage | Latency impact |
|---|---|---|---|
| Background read repair | Negligible | Read keys only | None |
| Blocking read repair | Low | Read keys only | Adds to the read |
| Probabilistic repair | Proportional to sampling | Slightly wider | Small |
| Hinted handoff | Storage for hints | Writes during known outages | None |
| Anti-entropy | High; scans and compares | Everything | None, runs separately |
| No repair | None | None | None |

The final row is not a straw man: systems where all data is rewritten frequently, or where replicas are rebuilt wholesale rather than repaired, genuinely do not need these mechanisms. Knowing why you do not need one is as useful as knowing how it works.

> **Ask before choosing it**  
> What fraction of the keyspace is read regularly? If most data is cold, read repair covers almost nothing and the real convergence work has to be done by anti-entropy — which is a much more expensive commitment and should be planned rather than discovered.

**How it fails**

**How read repair goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Cold data stays divergent | Repair only triggers on reads | Run anti-entropy for full coverage |
| Newer values overwritten | Ordering by skewed wall clocks | Logical versions or version vectors |
| Read latency degraded | Blocking repair on every divergence | Repair in the background by default |
| Divergence found too late | Never reading extra replicas | Probabilistic sampling of additional replicas |
| Repair traffic overwhelms replicas | Large-scale divergence repaired all at once | Throttle repair writes |
| Divergence growing unnoticed | Repair rate not monitored | Alert on the rate of repairs performed |
| Deleted data resurrected | Repairing over a delete with no tombstone | Tombstones with a defined lifetime |

> **Repairing over a deletion resurrects data unless deletes leave a marker**  
> If a delete simply removes the value on two replicas and the third still holds it, a read sees a present value and an absent one — and without a tombstone recording that the deletion is the newer event, repair concludes the value is the newer state and restores it everywhere. Deleted records reappear, which in systems handling personal data is a compliance failure rather than an inconvenience.

**Where it is used**

- **Dynamo-style databases**, where read repair is a core part of the eventual consistency story.
- **Cassandra**, exposing repair chance per table so hot tables converge without background work.
- **Distributed object stores**, repairing on access and scrubbing in the background for the rest.
- **Replicated caches**, where refreshing stale replicas on read is effectively the same mechanism.
- **Geo-replicated systems**, repairing regional divergence found during cross-region reads.

**In the interview**

The insight worth stating is that repair is free because the read already had the information.

- **Say the comparison is a by-product** of a quorum read, which is why the mechanism costs almost nothing.
- **Name the coverage gap immediately**: only read keys are repaired, so cold data diverges indefinitely.
- **Distinguish blocking from background** and explain when monotonic reads justify the added latency.
- **Require comparable versions**, because wall-clock ordering lets repair propagate older values over newer ones.
- **Complete the family** with hinted handoff and anti-entropy, and say what each one covers that the others do not.

**Practice drill**

Three replicas hold a key; one missed the last write. Trace a quorum read: which replicas answer, how the newest value is identified, what the client receives, and what repair does. Then explain what happens to a key that is written once and never read again, and design the mechanism that covers it. Finally, describe what could go wrong if versions were ordered by wall-clock time and one node's clock ran fast.

**Go deeper**

Read repair converges replicas by detecting stale copies during ordinary reads and updating them with the newest value.

**Its efficiency comes from reusing work already being done.** A quorum read fetches several copies and compares their versions to decide what to return, which simultaneously identifies exactly which replicas are behind. The repair then costs only the write to those replicas — there is no scanning, no coordination and no background job, which is why it is enabled by default in systems that use it.

**Its coverage is inversely related to where the risk is.** Frequently read keys repair themselves within seconds of any divergence, while cold keys are never read and therefore never repaired. That is precisely backwards from a durability perspective: the value held correctly on only one replica is most likely to be lost, and the less it is read the longer that vulnerable state persists — which is why read repair is always paired with something that scans.

**Blocking and background modes answer different questions.** Background repair returns to the client immediately and fixes the replicas afterwards, so a concurrent read can still see the old value; blocking repair waits for the stale replicas to acknowledge before responding, guaranteeing that a subsequent read cannot go backwards. The first is the right default because it adds no latency; the second is worth its cost only where monotonic reads matter to the application.

**It depends entirely on versions being comparable, and clocks are not.** Deciding which value is newer by the wall-clock time of the writing node means a node with a fast clock can label an older value as later, and read repair will then propagate that older value across every replica. The mechanism built to converge on the truth converges on the error instead, permanently and without any signal — which is why logical versions or version vectors are a prerequisite rather than a refinement.

**Deletions need explicit markers or repair undoes them.** A value absent on two replicas and present on a third looks like divergence in which the present value is the data, so without a tombstone recording that the deletion happened later, repair restores the record everywhere. Deleted data reappearing is an operational surprise in general and a compliance failure where personal data is concerned, and tombstone lifetime becomes a real constraint — they must outlive any replica's possible staleness.

**It is one of three complementary mechanisms, and the set matters more than any member.** Hinted handoff replays writes missed during a detected outage, read repair fixes drift on data people actually read, and anti-entropy scans everything on a slow cycle to catch what the other two miss. Each is cheap where the others are expensive, and a system running only one of them has a predictable gap — usually the cold, rarely-read data that nobody discovers is wrong until they need it.

**Related patterns:** Anti Entropy · Quorum Replication · Geographic Replication · Cache Invalidation

---

### Anti Entropy

*Periodically compare replicas in the background and reconcile differences, so divergence is bounded even for data nobody reads.*

> **When you hear…** Cold data diverging unnoticed · replicas rebuilt or restored from backup · a guarantee that divergence is bounded

**Flow:** `Build digests` → `Exchange summaries` → `Find differing ranges` → `Drill down` → `Transfer only diffs`

**The problem**

Read repair converges data that is read, which leaves the rarely accessed majority of most datasets drifting indefinitely. A replica that missed writes during an hour-long outage stays wrong until something happens to read each affected key.

Comparing replicas naively is prohibitive. A billion keys per replica cannot be enumerated and compared over the network on any useful schedule — the comparison itself would cost more than the data it protects.

> **Compare summaries, not data, and only descend where they differ**  
> A hash covering a range of keys is tiny, and two replicas whose hashes match over a range are provably identical there and need no further work. Arranging those hashes in a tree lets a comparison start with a single value covering everything and descend only into subtrees that disagree — so the cost scales with the amount of divergence rather than with the size of the dataset.

**Mental model**

A tree of hashes over the keyspace, exchanged between replicas. Matching nodes prune entire subtrees; mismatches are followed downward until the differing keys are identified.

1. **Build** — Each replica computes hashes over ranges of its data, combined upward into a tree.
2. **Exchange roots** — If the top hashes match, the replicas are identical and the comparison is over.
3. **Descend** — Where hashes differ, compare the children of that node only.
4. **Locate** — Continue until the specific differing keys are found.
5. **Reconcile** — Transfer only those keys, resolving versions as they are applied.

> **Anti-entropy is expensive and will compete with production traffic**  
> Building the tree reads the data, comparison generates network traffic, and reconciliation writes. Run unthrottled it can saturate disks and links, and because it typically runs on a schedule it tends to start at an inconvenient moment. Every production deployment needs it throttled and scheduled, and the throttle needs to be low enough that nobody notices it is running.

**How it works**

**Merkle trees and the cost of comparison**

```text
NAIVE COMPARISON
  1,000,000,000 keys x 16 B of hash = 16 GB
  transferred per replica pair, per run
  -> impossible on any useful schedule

MERKLE TREE
               root
           /          \
        h(A)          h(B)
       /    \        /    \
    h(A1)  h(A2)  h(B1)  h(B2)
     ...    ...    ...    ...
  leaves cover ranges of keys

COMPARISON
  exchange roots
    match    -> identical, DONE, one hash exchanged
    differ   -> exchange children of the root
  recurse only into differing subtrees

COST
  depth 20 tree over 1e9 keys
  one differing key
  -> ~20 hash comparisons to locate it
  -> a few kilobytes instead of 16 GB

  1000 differing keys, scattered
  -> more subtrees differ, cost rises
  -> still proportional to DIVERGENCE, not data size

RESOLUTION AT THE LEAVES
  transfer the differing keys with their versions
  apply the newer version, as decided by version
  vectors or logical clocks
  -> same version comparison problem as read repair
```

1. **Use a hash tree so cost scales with divergence** — Full comparison is impossible at any real data size.
2. **Throttle reconciliation aggressively** — Unthrottled repair competes directly with production traffic.
3. **Schedule runs for quiet periods** — The work is not urgent; it only needs to complete within the divergence window.
4. **Keep trees incrementally updated where possible** — Rebuilding from scratch every run reads the entire dataset.
5. **Resolve with versions, not timestamps** — The same clock-skew hazard as read repair, with wider reach.
6. **Run it after any known outage or restore** — A replica rebuilt from backup is divergent by definition.

**Scheduling and what it guarantees**

```text
THE GUARANTEE
  run every T -> divergence is bounded by T
  run weekly  -> a replica can be wrong for a week
  run daily   -> wrong for at most a day
  -> choose T from how long you can tolerate an
     undetected inconsistency

TOMBSTONE INTERACTION  (the sharp edge)
  deletes leave tombstones that expire after G
  if anti-entropy runs LESS often than G:
    replica missed the delete
    tombstone expires everywhere else
    anti-entropy compares: value present vs absent
    -> the value is treated as newer
    -> DELETED DATA IS RESURRECTED
  -> the iron rule: repair interval < tombstone
     lifetime, always

WHEN TO RUN IT IMMEDIATELY
  a replica returns after a long outage
  a node is rebuilt or restored from backup
  after any incident involving dropped writes
  -> scheduled runs handle drift; these are known
     divergences and should not wait

COST CONTROL
  compare a subset of ranges per run
  -> full coverage over several runs
  -> bounded cost per run, longer T
```

| Metric | Value | Note |
|---|---|---|
| Comparison cost | log-scale | **by divergence** |
| Guarantee | bounded by interval | choose deliberately |
| Iron rule | interval < tombstone TTL | or deletes revert |
| Must be | throttled | competes with traffic |

> **Repair intervals longer than tombstone lifetimes resurrect deleted data**  
> A replica that missed a deletion holds the old value. Once the tombstones recording that deletion have expired on the other replicas, a comparison sees a value on one side and nothing on the other, with no evidence that the absence is the newer state — so the value is restored everywhere. This is a well-documented operational hazard, it is entirely preventable by keeping the repair interval below the tombstone lifetime, and it is still one of the more common ways deleted records come back.

**Technologies**

| System | Mechanism | Note |
|---|---|---|
| Cassandra repair | Merkle tree comparison between replicas | Must complete within the tombstone window |
| Dynamo-style stores | Merkle trees as originally described | Paired with read repair and hinted handoff |
| Object stores | Background scrubbing and verification | Also detects bit rot, not only divergence |
| Distributed filesystems | Periodic checksum verification | Repairs from parity or other replicas |
| Read repair | Convergence on access | Covers hot data cheaply |
| Full rebuild | Copy a replica wholesale | Simpler when divergence is extensive |

Full rebuild is the pragmatic alternative when a replica has been out for a long time: comparing two datasets that differ substantially costs more than streaming a fresh copy, and most operators have a threshold beyond which they replace rather than repair.

**Trade-offs**

**Convergence strategies compared**

| Strategy | Covers cold data | Cost | Latency to converge |
|---|---|---|---|
| Read repair only | No | Negligible | Immediate for hot keys, never for cold |
| Hinted handoff | Only known outages | Low | On replica return |
| Anti-entropy on a schedule | Yes | High per run | Up to the interval |
| Continuous background repair | Yes | Sustained | Shorter |
| Full replica rebuild | Yes | Very high | Immediate once complete |
| Nothing | No | None | Never |

Continuous low-rate repair is increasingly preferred over periodic large runs: the same work spread evenly avoids the operational spike, keeps the divergence window shorter, and removes the temptation to skip a scheduled run because the cluster is busy.

> **Ask before choosing it**  
> How long may a replica hold wrong data before it matters? That interval sets the schedule, and it must also sit below the tombstone lifetime — if the two constraints conflict, the tombstone lifetime has to grow, because the alternative is deleted data returning.

**How it fails**

**How anti-entropy goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Deleted data returns | Repair interval exceeds tombstone lifetime | Repair more often, or extend tombstone retention |
| Production latency spikes during repair | Unthrottled comparison and transfer | Throttle; schedule for quiet periods |
| Repair never completes | Dataset too large for the window | Subset ranges per run; continuous repair |
| Divergence persists despite repair | Version resolution choosing wrongly | Version vectors rather than wall clocks |
| Repair costs more than a rebuild | Extensive divergence after a long outage | Replace the replica instead |
| Trees rebuilt from scratch each run | No incremental maintenance | Update tree nodes as data changes |
| Nobody notices it stopped running | No monitoring of completion | Alert on time since last successful run |

> **An anti-entropy process that quietly stopped is invisible until data is lost**  
> Nothing fails when repair stops running: reads succeed, writes succeed, and divergence simply grows unbounded in the background. The first symptom is typically a replica failure that exposes how far the remaining copies had drifted, or deleted records reappearing once tombstones expire. Time since the last successful full pass is the metric that has to be alerted on, because no other signal exists.

**Where it is used**

- **Cassandra clusters**, where scheduled repair is a documented operational requirement rather than an option.
- **Dynamo-style stores**, using Merkle trees exactly as the original design described.
- **Object storage systems**, scrubbing continuously to detect both divergence and silent data corruption.
- **Distributed filesystems**, verifying checksums and repairing from replicas or parity.
- **Backup verification**, comparing restored data against the source using the same tree technique.

**In the interview**

Show the comparison arithmetic and then raise the tombstone interaction; the second is what marks operational experience.

- **Explain why naive comparison is impossible** with a data-size figure, then introduce hash trees as the fix.
- **Say the cost scales with divergence, not dataset size**, which is the property that makes the technique viable.
- **State the guarantee as a bound**: running every interval means divergence lasts at most that long.
- **Raise the tombstone rule unprompted** — repair must run more often than tombstones expire, or deletes revert.
- **Insist on throttling and monitoring**, since repair competes with production traffic and fails silently when it stops.

**Practice drill**

Three replicas hold a billion keys each, and one missed an hour of writes during an outage. Explain why a direct comparison is infeasible, then design the hash-tree comparison and estimate the traffic needed to locate a thousand divergent keys. Choose a repair interval, state the guarantee it provides, and verify it against a tombstone lifetime you also choose. Finally, say how you would know the process had silently stopped running.

**Go deeper**

Anti-entropy compares replicas in the background and reconciles differences, providing convergence for data that reads never touch.

**It exists because read repair has a structural gap.** Convergence driven by access covers whatever is read and nothing else, so the cold majority of most datasets drifts indefinitely — and cold data held correctly on a single replica is exactly what is lost when that replica fails. A mechanism that scans independently of access is the only way to bound divergence across the whole keyspace.

**Hash trees are what make the comparison affordable.** Enumerating and exchanging every key is impossible at real data sizes, but a single hash covering a range proves equality over that range, so matching subtrees are pruned entirely. The cost then scales with how much divergence exists rather than with how much data exists, which turns an impossible comparison into one that usually finishes having transferred a few kilobytes.

**Running it on a schedule converts divergence from unbounded to bounded.** The interval is the guarantee: run daily and a replica can be wrong for at most a day. Choosing that number is a durability decision — how long an undetected inconsistency is tolerable — and it should be made explicitly, because the default of not running it at all provides no bound whatsoever.

**The tombstone interaction is the sharpest operational edge in replicated stores.** Deletions are recorded as markers that eventually expire, and a replica that missed a deletion still holds the value. If repair does not run before those markers expire elsewhere, the comparison sees a value on one side and an absence on the other with no evidence about which is newer, and restores the deleted record everywhere. Keeping the repair interval below the tombstone lifetime is an absolute constraint, not a tuning preference.

**It is expensive enough to need deliberate operational handling.** Building trees reads data, comparison uses the network, and reconciliation writes — all competing with production traffic on the same disks and links. Throttling is mandatory, and spreading the work continuously at a low rate is increasingly preferred over periodic large runs, because it shortens the divergence window and removes the temptation to skip a run during a busy period.

**Its failure mode is silence, which makes monitoring part of the design.** When repair stops running nothing breaks: reads and writes continue normally while divergence grows in the background, and the problem surfaces only when a replica is lost or deleted data reappears. Time since the last successful complete pass is the metric that reveals this, and a system that runs anti-entropy without alerting on that metric has a durability guarantee it cannot actually verify.

**Related patterns:** Read Repair · Quorum Replication · Gossip Membership · Geographic Replication

---

## Feeds

### Fanout On Write

*Push each new item into every follower's precomputed feed at write time, so reading a feed is a single cheap lookup.*

> **When you hear…** Feed reads vastly outnumber writes · feed latency must be minimal · followers per author are bounded

**Flow:** `User posts` → `Look up followers` → `Write to each inbox` → `Feed is prebuilt` → `Read is one query`

**The problem**

A feed assembled at read time must query every account the viewer follows, merge the results by time, and rank them. For someone following two thousand accounts that is two thousand lookups per feed open, and feeds are opened constantly.

The read path is the hot path. Most social products see reads outnumber writes by two or three orders of magnitude, so any work that can be moved off the read is worth a great deal of work on the write.

> **Do the work once per post rather than once per view**  
> A post is written once and read by all of its author's followers, many of them repeatedly. Computing each follower's feed at write time — inserting the post into a per-user list — means every subsequent read is a single sequential lookup of an already-assembled list. The total work is lower because the expensive assembly happens once instead of once per view.

**Mental model**

A materialised inbox per user. Writing a post appends its identifier to every follower's inbox; reading a feed is a range scan of one inbox.

1. **Post** — The author writes the item to a store of posts.
2. **Resolve** — The system fetches the author's follower list.
3. **Fan out** — The post identifier is appended to each follower's feed list.
4. **Read** — Opening a feed reads one list, already ordered.
5. **Hydrate** — Post identifiers are turned into full content, usually from a cache.

> **The write cost is proportional to follower count, without limit**  
> A post from an account with fifty million followers requires fifty million inserts. At any realistic write rate that is not a background task but a sustained load spike, and it arrives exactly when the account is most active. Pure fanout-on-write does not scale to accounts of that size, which is why no large platform uses it alone.

**How it works**

**The write amplification**

```text
TYPICAL USER, 200 followers
  1 post -> 200 inserts
  cheap, done in milliseconds

POPULAR USER, 1,000,000 followers
  1 post -> 1,000,000 inserts
  at 10,000 inserts/s -> 100 seconds
  -> the post appears in feeds over 100 seconds
  -> and the fanout job holds capacity the whole time

CELEBRITY, 50,000,000 followers
  1 post -> 50,000,000 inserts
  -> not a job, an incident

STORAGE
  average 500 followers, 1M posts/day
  -> 500,000,000 feed entries per day
  -> entries are small (post id + timestamp) but the
     volume is the cost, not the size

WHAT MAKES IT BEARABLE
  cap the stored feed length (e.g. 800 entries)
    older entries are dropped; deep scrolling falls
    back to a read-time query
  write asynchronously through a queue
    the post returns immediately; fanout catches up
  only fan out to ACTIVE followers
    an account dormant for a year does not need a
    prebuilt feed
```

1. **Fan out asynchronously through a queue** — The author's write must not wait for millions of inserts.
2. **Cap each stored feed at a few hundred entries** — Unbounded feeds cost storage for pages nobody scrolls to.
3. **Fan out only to active followers** — Dormant accounts are usually most of the follower list and none of the readers.
4. **Store identifiers, not content** — Copying full posts multiplies storage and makes edits impossible to propagate.
5. **Handle the backlog explicitly** — A large fanout must be throttled so it does not starve ordinary posts.
6. **Do not use it for very large accounts** — Above a threshold the cost is unbounded; those accounts need the read-time path.

**Where the design breaks**

```text
NEW FOLLOW
  a user follows someone
  -> their existing posts are not in the follower's
     feed
  -> backfill (expensive) or accept the gap

UNFOLLOW / BLOCK / DELETE
  the post is already in millions of feeds
  -> remove from all of them (expensive), or
  -> filter at read time (cheap, but every read pays
     a check)

EDIT
  storing ids rather than content means the edit is
  picked up on hydration automatically
  -> this alone justifies storing ids

RANKING
  a chronological inbox is easy
  a ranked feed needs scores that change over time
  -> the prebuilt list becomes a candidate set, and
     ranking happens at read time anyway

THE HONEST POSITION
  fanout-on-write optimises retrieval, not ranking
  modern feeds are ranked, so the pattern supplies
  candidates rather than the finished feed
```

| Metric | Value | Note |
|---|---|---|
| Read cost | one lookup | **very fast** |
| Write cost | × followers | unbounded |
| Storage | entries per follower | capped list |
| Breaks at | large accounts | needs a hybrid |

> **Deletions and blocks are expensive once a post is in millions of inboxes**  
> Removing a post that has already been fanned out means touching every feed that contains it, which costs as much as the original fanout. Most systems therefore filter at read time instead — checking blocks and deletions during hydration — which makes every read slightly more expensive in exchange for making deletion cheap. That trade has to be made deliberately, because discovering it after launch means choosing between slow deletes and stale feeds.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Feed storage | Redis lists or sorted sets | Fast, capped naturally, memory-bound |
| Feed storage | Wide-column stores | Cheaper per entry at very large scale |
| Fanout execution | Work queue with many workers | Must be throttled for large accounts |
| Follower lists | Graph or relational store | Read on every post; heavily cached |
| Hydration | Post cache keyed by id | Where the actual content is assembled |
| Alternative | Fanout on read | For accounts with very many followers |

Redis sorted sets are the common choice because capping a feed is a single trim operation and reads are already ordered, but memory becomes the binding constraint at large user counts — which is where teams typically move the long tail of inactive users to a cheaper store.

**Trade-offs**

**Feed construction strategies**

| Strategy | Read cost | Write cost | Fits |
|---|---|---|---|
| Fanout on write | One lookup | Follower count | Most users, bounded followings |
| Fanout on read | Following count | One insert | Celebrities, rarely read feeds |
| Hybrid | One lookup plus a few | Bounded | Real systems |
| No feed, search on demand | Expensive | None | Low-traffic products |

The hybrid row is where every large platform ends up, and arriving at it directly in a design discussion — rather than defending pure fanout-on-write against the celebrity case — is what demonstrates that the trade-off is understood.

> **Ask before choosing it**  
> What is the follower distribution? A product where the maximum following is a few hundred can use pure fanout-on-write happily; one with a long tail of accounts followed by millions cannot, and the threshold at which you switch strategies is the actual design decision.

**How it fails**

**How fanout on write goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Celebrity post stalls the fanout queue | Millions of inserts as one job | Threshold to the read-time path; throttle large fanouts |
| Posts appear minutes late | Fanout backlog behind a large job | Priority queues; cap per-job work |
| Storage grows faster than users | Uncapped feed length | Trim to a few hundred entries |
| Feeds full of content from dormant accounts | Fanning out to every follower | Only fan out to active users |
| Deleting a post takes minutes | Removal from every inbox | Filter at read time instead |
| Edits not reflected | Content copied into feeds | Store identifiers and hydrate |
| New follows show an empty feed | No backfill on follow | Backfill a page, or merge at read time |

> **A single large account can monopolise the fanout infrastructure**  
> One post to fifty million followers is fifty million writes that will occupy the fanout workers for a long time, delaying every ordinary user's post behind it. Without per-job limits and separate handling for large accounts, the platform's feed latency becomes a function of what one celebrity happened to do — which is both an availability problem and an unpleasant surprise during exactly the moments the product is busiest.

**Where it is used**

- **Early Twitter**, the canonical description of fanout-on-write and of the celebrity problem that broke it.
- **Instagram-style feeds**, where prebuilt inboxes serve the common case and large accounts are special-cased.
- **Activity and notification feeds**, which have the same shape and usually smaller fanouts.
- **Messaging group delivery**, fanning a message into per-member inboxes.
- **Email-style inbox models**, the original materialised-per-recipient design.

**In the interview**

Do the arithmetic for a celebrity early; it is what forces the hybrid conclusion naturally.

- **Justify it with the read/write ratio** — reads outnumber writes by orders of magnitude, so moving work to writes is profitable.
- **Compute the celebrity case out loud**: fifty million followers means fifty million inserts per post.
- **Cap the stored feed** and say what happens when a user scrolls beyond it.
- **Store identifiers, not content**, so edits propagate and storage stays small.
- **Raise deletion and blocking** as read-time filters, since removing a post from millions of inboxes is as expensive as writing it.

**Practice drill**

Design a feed for a platform where the median user has 200 followers and the largest account has 50 million. Compute the write amplification for both. Choose a feed length cap and justify it from scrolling behaviour. Then handle three operations: a user following someone new, a post being deleted, and a post being edited — saying for each whether you handle it at write time or read time and why.

**Go deeper**

Fanout on write precomputes each user's feed by inserting every new post into the inbox of every follower, so that reading a feed is a single lookup.

**It is an application of doing expensive work where it happens least often.** Feeds are read far more than they are written, so assembling a feed once per post rather than once per view reduces total work substantially, and it moves the remaining cost off the latency-critical path. This is the same reasoning behind any materialised view, applied to a workload where the read-to-write ratio is extreme.

**Its cost is proportional to follower count and therefore unbounded.** A typical user's post costs a few hundred inserts; a large account's post costs millions, arriving as a sustained burst that occupies the fanout infrastructure and delays everyone else's posts behind it. No tuning removes this, because the work is inherent to the approach, which is why the pattern must be bounded by a threshold rather than optimised.

**Storing identifiers rather than content is what keeps it viable.** Copying post bodies into millions of inboxes multiplies storage and makes editing impossible to propagate, whereas storing an identifier keeps entries tiny and lets hydration fetch the current version at read time. It also means a deleted post can be filtered during hydration instead of being chased through every inbox that contains it.

**Mutations after fanout are the awkward part of the design.** A new follow leaves the follower's feed missing history, deletions and blocks require either expensive removal or read-time filtering, and each of these decisions shifts cost between the two paths. The usual resolution is to filter at read time and accept a slightly more expensive hydration, because it keeps the write path bounded — but it is a choice that should be made explicitly rather than discovered when the first deletion takes minutes.

**Capping the stored feed matters more than it appears.** Users scroll a page or two, so storing thousands of entries per user buys almost no experience and multiplies storage across the entire user base. Trimming to a few hundred, with deeper scrolling falling back to a read-time query, keeps the storage bill proportional to what people actually read rather than to everything that was ever posted.

**Modern ranked feeds change what the pattern delivers.** A chronological inbox is a finished feed, but a ranked feed requires scores that depend on time, engagement and viewer context, none of which is known at write time. The prebuilt list therefore becomes a candidate set that ranking consumes at read time, which is still valuable — retrieval was the expensive part — but it means the pattern supplies input to the feed rather than the feed itself, and describing it as the latter overstates what it does.

**Related patterns:** Fanout On Read · Hybrid Fanout · Materialized View · Hot Key Mitigation

---

### Fanout On Read

*Store each post once and assemble a feed by querying the accounts a viewer follows at read time, keeping writes trivial at the cost of expensive reads.*

> **When you hear…** Authors with enormous follower counts · feeds read rarely relative to posting · write amplification unacceptable

**Flow:** `Post stored once` → `User opens feed` → `Query each followed` → `Merge by time` → `Return page`

**The problem**

Precomputing feeds means a post from an account with fifty million followers costs fifty million writes. The work is enormous, most of it lands in feeds nobody opens, and it arrives as a burst that delays everyone else's posts.

The wasted proportion is the striking part. Of those fifty million inboxes, perhaps a few million belong to people who will open the app today — the rest of the work is discarded without ever being read.

> **Do the work only for feeds someone actually opens**  
> A post read by nobody should cost nothing beyond storing it once. Assembling the feed when a viewer asks for it means the cost is paid exactly in proportion to consumption rather than to follower counts, which makes very large accounts free to post and makes inactive followers cost nothing at all.

**Mental model**

Posts stored once per author. A feed request queries the recent posts of every followed account, merges them and returns a page.

1. **Write** — The post is stored once against its author.
2. **Request** — A viewer opens their feed.
3. **Gather** — Recent posts are fetched from each followed account.
4. **Merge** — Results are combined and ordered.
5. **Return** — One page is served; the rest is discarded.

> **Read cost is proportional to how many accounts the viewer follows**  
> Someone following two thousand accounts triggers two thousand queries per feed open, and the response cannot be returned until the slowest of them completes. Because feeds are opened constantly, this cost is paid over and over for the same data — the exact inverse of the fanout-on-write problem, and equally unbounded.

**How it works**

**The read cost and how it is contained**

```text
NAIVE
  viewer follows 500 accounts
  -> 500 queries per feed open
  -> latency = the SLOWEST of 500
  -> at 10,000 feed opens/s that is 5,000,000
     queries/s

CONTAINMENT

1  PER-AUTHOR RECENT-POST CACHE
     each author's last ~50 posts cached
     -> the 500 lookups become cache hits
     -> this is the single most important measure

2  BATCH THE QUERIES
     one multi-get instead of 500 round trips
     -> latency becomes one slow shard, not 500

3  LIMIT WHAT IS FETCHED
     only posts newer than the viewer's last visit
     -> most followed accounts contribute nothing

4  CACHE THE ASSEMBLED FEED BRIEFLY
     30-60 s TTL on the merged result
     -> repeated opens and pagination are free
     -> this quietly converts the pattern into
        something close to fanout-on-write for
        active users

MERGE
  each source is already time-ordered
  -> k-way merge, stop once a page is filled
  -> do not fetch everything and sort
```

1. **Cache each author's recent posts** — It converts the fan-out from database queries into cache reads.
2. **Batch the lookups rather than iterating** — Latency should be one round trip, not the number of followed accounts.
3. **Merge lazily and stop at a page** — Fetching everything and sorting does far more work than is needed.
4. **Cache the assembled feed for a short period** — Pagination and repeated opens dominate real traffic.
5. **Bound how many accounts are queried** — Very large followings need sampling or prioritisation.
6. **Use it selectively for large accounts** — It is a complement to precomputation, not a replacement.

**Where it wins and where it fails**

```text
WINS
  celebrity posts       one write regardless of
                        follower count
  inactive users        cost nothing at all
  deletions and edits   nothing to clean up; the
                        feed is built fresh
  new follows           appear immediately, with
                        history, no backfill
  ranking changes       apply instantly, since
                        nothing is precomputed

FAILS
  large followings      read cost scales with them
  constant feed opens   the same work repeated
  deep pagination       each page costs another
                        assembly
  latency               bounded by the slowest source

THE SYMMETRY
  fanout-on-write  cost ∝ followers   (write side)
  fanout-on-read   cost ∝ following   (read side)
  -> neither distribution is bounded in a real social
     graph
  -> which is why production systems use both

WHAT MAKES IT PRACTICAL
  the per-author cache is doing most of the work; a
  pure database-backed fanout-on-read is rarely
  viable at scale
```

| Metric | Value | Note |
|---|---|---|
| Write cost | one insert | **regardless of reach** |
| Read cost | × following | unbounded |
| Deletes/edits | free | nothing precomputed |
| Depends on | author caches | not optional |

> **Deep pagination multiplies the cost of an already expensive read**  
> Each page requires another merge across the followed accounts, so a user scrolling through five pages pays the assembly cost five times. Cursor-based pagination over a short-lived cached merge is what makes scrolling affordable; recomputing from scratch per page turns an engaged user into the most expensive user on the platform.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Per-author post cache | Redis lists per author | The mechanism that makes this viable |
| Batched fetching | Multi-get or scatter-gather | Latency is one round trip, not many |
| Merge | K-way merge over ordered sources | Stop once the page is full |
| Short-lived feed cache | Cached merged result | Covers pagination and repeat opens |
| Following lists | Graph store | Read on every feed open; heavily cached |
| Complement | Fanout on write | For ordinary accounts |

The per-author cache is the component that decides whether this pattern is practical. With it, a five-hundred-source fanout is five hundred cache reads batched into a couple of round trips; without it, the same request is five hundred database queries and will not survive contact with real traffic.

**Trade-offs**

**Read-time against write-time assembly**

| Property | Fanout on read | Fanout on write |
|---|---|---|
| Write cost | Constant | Proportional to followers |
| Read cost | Proportional to following | Constant |
| Storage | One copy per post | One entry per follower |
| Deletes and edits | Free | Expensive or filtered |
| New follows | Immediate with history | Need backfill |
| Ranking changes | Apply instantly | Require recomputation |
| Scales badly when | Users follow many accounts | Accounts have many followers |

The final row states the symmetry plainly: each pattern is unbounded in the dimension the other handles, and real social graphs are unbounded in both directions — which is the argument for combining them rather than choosing.

> **Ask before choosing it**  
> How many accounts does a typical user follow, and how often do they open the feed? A product where people follow a handful of sources and check occasionally can use read-time assembly alone; one where users follow thousands and refresh constantly cannot.

**How it fails**

**How fanout on read goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Feed latency unacceptable | Querying sources serially | Batch the fetches into one round trip |
| Database overwhelmed by feed opens | No per-author caching | Cache each author's recent posts |
| Slowest source sets the latency | Waiting for every source | Timeout slow sources; serve what arrived |
| Scrolling is extremely expensive | Recomputing the merge per page | Cursor pagination over a cached merge |
| Users following thousands time out | Unbounded fan-out | Prioritise or sample sources |
| Memory exhausted during merge | Fetching everything before sorting | Lazy k-way merge, stop at a page |
| Cost dominated by repeat opens | No short-lived feed cache | Cache the assembled result briefly |

> **Waiting for every source makes the feed as slow as the worst one**  
> A merge across five hundred sources completes when the last responds, so a single slow shard determines the latency of the whole feed — the same tail-at-scale arithmetic that makes any large fan-out unpredictable. Timing out slow sources and serving a slightly incomplete feed is almost always better for the user than a complete feed that takes two seconds, and deciding that policy in advance is part of the design.

**Where it is used**

- **Twitter's celebrity path**, where posts from very large accounts are merged at read time rather than fanned out.
- **Feeds with small followings**, such as professional or interest-based products where read-time assembly is sufficient alone.
- **Search-driven and topic feeds**, which are assembled per request by construction.
- **Chronological timelines in smaller products**, where the simplicity outweighs the read cost.
- **Any feed where posts are frequently edited or removed**, since nothing precomputed has to be cleaned up.

**In the interview**

Present it as the mirror image of fanout-on-write, then show why neither alone is sufficient.

- **State the symmetry**: write-time cost scales with followers, read-time cost scales with following.
- **Say the per-author cache is what makes it viable**, since raw database fan-out at read time does not survive real traffic.
- **Batch the fetches** so latency is one round trip rather than the number of followed accounts.
- **Merge lazily and stop at a page**, rather than fetching everything and sorting.
- **Give the operational advantages** — free deletes and edits, immediate new follows, instant ranking changes — because they are real and often forgotten.

**Practice drill**

A user follows 500 accounts and opens their feed ten times a day; the platform serves 10,000 feed opens per second. Compute the naive query load and design the containment: caching, batching, merging and pagination. State the resulting per-request latency budget and where it is spent. Then explain what this design does well that fanout-on-write does badly, with three specific operations.

**Go deeper**

Fanout on read stores each post once and assembles a feed at request time by merging the recent posts of every followed account.

**It makes cost proportional to consumption rather than to reach.** A post costs one write regardless of whether its author has ten followers or fifty million, and feeds that are never opened cost nothing at all. In a social graph where a large fraction of accounts are dormant, that eliminates an enormous amount of work that precomputation performs and discards.

**Its cost reappears on the read side, scaled by following count.** A viewer following two thousand accounts triggers a two-thousand-way gather on every feed open, and because feeds are opened constantly, the same assembly is repeated endlessly for data that has not changed. This is the exact mirror of the write-side problem, and it is equally unbounded — which is why the two patterns are complements rather than alternatives.

**The per-author cache is what makes it practical rather than theoretical.** Caching each author's recent posts converts the gather from hundreds of database queries into hundreds of cache reads that batch into a couple of round trips. Without that layer the pattern is not viable at scale, and discussions that present read-time assembly as inherently cheap are usually assuming this component without saying so.

**Latency is governed by the slowest source, not the average.** Waiting for every followed account means one degraded shard sets the response time for the whole feed, so a policy of timing out stragglers and serving a marginally incomplete feed is almost always the better user experience. Making that decision explicitly, with a defined timeout and a defined degradation, is what separates a fan-out that performs from one that is merely correct.

**Mutations are free, and this is a real advantage.** Because nothing is precomputed, a deleted post disappears, an edit takes effect, a new follow brings history, and a ranking change applies — all immediately, with no cleanup and no backfill. Systems that precompute pay ongoing complexity for each of these, which is worth remembering when the read cost is being weighed.

**Short-lived caching of the assembled feed quietly blurs the distinction.** Caching a merged result for thirty seconds makes pagination and repeat opens free, and for an active user it means the feed is effectively precomputed — just triggered by their first request rather than by every author's post. That middle ground is where the hybrid approaches live, and recognising that the two patterns are ends of a spectrum rather than opposites is what makes choosing between them a matter of thresholds rather than doctrine.

**Related patterns:** Fanout On Write · Hybrid Fanout · Cache Aside · Scatter Gather

---

### Hybrid Fanout

*Precompute feeds for ordinary accounts and merge large accounts in at read time, so neither followers nor followings can make an operation unbounded.*

> **When you hear…** A follower distribution with a long tail · pure fanout failing on celebrities or on heavy followers · a production feed at scale

**Flow:** `Classify authors` → `Normal accounts fan out` → `Large accounts do not` → `Read merges both` → `Costs stay bounded`

**The problem**

Fanout on write costs one insert per follower, so an account with fifty million followers produces fifty million writes per post. Fanout on read costs one query per followed account, so a user following two thousand accounts pays two thousand lookups per feed open.

Both costs are unbounded in a real social graph, and they are unbounded in different directions. Choosing either one alone means accepting that some portion of the user base will have an experience that cannot be made acceptable.

> **Apply each strategy where it is bounded and avoid it where it is not**  
> The two approaches fail on opposite ends of the same distribution. Precomputing for accounts with ordinary follower counts is cheap and keeps reads fast; leaving the handful of very large accounts out of precomputation and merging them at read time keeps writes cheap, and because those accounts are few, the read-time merge stays small. Each mechanism is used only in the region where its cost is bounded.

**Mental model**

A threshold on follower count divides authors into two classes. Feeds are the union of a precomputed inbox and a small read-time merge over the large accounts the viewer follows.

1. **Classify** — Authors above a follower threshold are marked as non-fanout.
2. **Write** — Ordinary posts are fanned out to followers; large-account posts are stored only.
3. **Read** — The precomputed inbox is fetched.
4. **Merge** — Recent posts from the viewer's followed large accounts are merged in.
5. **Serve** — The combined, ordered result is returned.

> **The threshold creates two code paths that must produce identical results**  
> An account crossing the threshold changes strategy, and the transition must not duplicate posts, lose them, or reorder them relative to the other path. Getting this wrong produces feeds with missing or repeated items that appear only for users following an account that recently crossed the line — a bug that is difficult to reproduce and easy to dismiss as a client issue.

**How it works**

**The threshold and the read path**

```text
CLASSIFICATION
  followers < 100,000   -> fan out on write
  followers >= 100,000  -> do not fan out

WHY ~100,000
  fanout of 100,000 inserts is a few seconds of work
  -> acceptable as a background job
  and the number of accounts above it is small
  -> so the read-time merge stays small

WRITE PATH
  ordinary author  -> insert into every follower's
                      inbox (async, capped, active
                      followers only)
  large author     -> store the post; append to the
                      author's own recent-post cache

READ PATH
  1  fetch the viewer's precomputed inbox  (1 read)
  2  find which large accounts they follow (cached
     list, usually 0-20)
  3  fetch those authors' recent posts    (batched)
  4  merge by time or rank, take a page

TYPICAL COST
  1 inbox read + 1 batched multi-get
  -> two round trips regardless of following size
  -> because the number of LARGE accounts a person
     follows is small even when their total following
     is huge
```

1. **Set the threshold from what a fanout job can absorb** — It should be the largest fanout that runs comfortably in the background.
2. **Cache each viewer's list of followed large accounts** — Computing it per feed open reintroduces the read-time cost.
3. **Deduplicate carefully at the merge** — A post can appear in both paths while an account is crossing the threshold.
4. **Handle threshold crossings explicitly** — Backfill or exclude deliberately, rather than leaving it to chance.
5. **Keep both paths producing the same ordering** — Feeds that reorder depending on which path supplied an item look broken.
6. **Monitor the two paths separately** — The failure modes are different and an aggregate metric hides both.

**Why the read-time merge stays small**

```text
THE FOLLOWER DISTRIBUTION
  99.9% of accounts   < 10,000 followers
  ~0.1%               100,000 to millions
  -> almost every post is fanned out cheaply

THE FOLLOWING DISTRIBUTION
  a user follows 2,000 accounts
  of those, how many have > 100,000 followers?
  typically 5-30
  -> the read-time merge is over tens of sources,
     not thousands

THIS IS THE WHOLE TRICK
  large accounts are rare, so few of them appear in
  any one following list
  -> the expensive path is applied to a small,
     naturally bounded set

COST COMPARISON
  pure write fanout   50,000,000 inserts per
                      celebrity post
  pure read fanout    2,000 queries per feed open
  hybrid              a few hundred inserts per
                      ordinary post
                      + ~20 cached lookups per read

TUNING THE THRESHOLD
  lower  -> cheaper writes, larger read merges
  higher -> cheaper reads, heavier fanout jobs
  -> measure both and move it; it is the main dial
```

| Metric | Value | Note |
|---|---|---|
| Write path | bounded | by threshold |
| Read path | bounded | **few large accounts** |
| Round trips | two | regardless of following |
| Main dial | the threshold | measure and adjust |

> **Duplicates appear when an account crosses the threshold**  
> An account that grows past the limit has posts already sitting in followers' inboxes, and its new posts will arrive through the read-time path. If the merge does not deduplicate by post identifier, followers see items twice during the transition; if the transition is handled by removing the old entries, that removal is itself a large fanout. Deduplicating at the merge is the cheaper and more robust choice.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Precomputed inboxes | Redis sorted sets or wide-column store | Capped per user |
| Large-account posts | Per-author recent-post cache | The read-time source |
| Classification | Follower count with hysteresis | Avoid flapping at the boundary |
| Followed-large-accounts list | Cached per viewer | Must not be computed per request |
| Merge and ranking | Application layer | Where the two paths become one feed |
| Ranking service | Scores the merged candidate set | Modern feeds rank after merging |

Hysteresis on the classification matters more than it sounds: an account hovering at the threshold that flips strategy repeatedly produces inconsistent feeds and repeated backfills, so the promote and demote thresholds should be different values.

**Trade-offs**

**The three strategies**

| Strategy | Write cost | Read cost | Complexity |
|---|---|---|---|
| Fanout on write | Unbounded by followers | One lookup | Low |
| Fanout on read | One insert | Unbounded by following | Low |
| Hybrid | Bounded by threshold | Bounded by rare large accounts | Two paths to keep consistent |
| Hybrid with ranking | Same | Same plus scoring | Highest |

The complexity column is the honest cost. A hybrid has two write paths, two read sources and a merge that must reconcile them, which is meaningfully more code and more failure modes than either pure approach — justified only once one of those approaches has actually stopped working.

> **Ask before choosing it**  
> Does the follower distribution actually have a long tail? A product where no account exceeds a few thousand followers gains nothing from a hybrid and pays its complexity, so the pattern should be introduced when the distribution demands it rather than by default.

**How it fails**

**How hybrid fanout goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicate posts in a feed | Both paths supplying the same post | Deduplicate by post identifier at the merge |
| Posts missing during a transition | Account crossed the threshold with no handling | Explicit promote and demote procedures |
| Read cost creeping upward | Threshold too low, many accounts on the read path | Raise it; measure the merge size |
| Fanout jobs backing up | Threshold too high for the worker pool | Lower it, or add capacity |
| Inconsistent ordering between paths | Different sort keys or precision | Single ordering rule applied after merging |
| Accounts flapping across the boundary | Single threshold with no hysteresis | Separate promote and demote values |
| Followed-large-account list computed per read | Not cached | Cache it and invalidate on follow changes |

> **Two paths diverging is the characteristic failure of this pattern**  
> Everything that makes a hybrid valuable comes from having two mechanisms, and everything that makes it fragile comes from needing them to agree. Different ordering, different filtering rules for blocks, different handling of deletions, or a post appearing in both — each produces feeds that are subtly wrong for a subset of users, and each is invisible in aggregate metrics. The merge must be the single place where ordering, filtering and deduplication are decided, for items from both sources.

**Where it is used**

- **Twitter's timeline**, the best-documented hybrid and the origin of the celebrity threshold idea.
- **Instagram and comparable feeds**, precomputing for ordinary accounts and merging large ones at read time.
- **Professional networks**, where following distributions are skewed but less extreme.
- **Notification systems**, which apply the same split for high-volume sources.
- **Any feed with a long-tailed follower distribution**, which is nearly all of them at scale.

**In the interview**

Arrive here by showing both pure approaches failing, then explain why the read-time merge stays small.

- **Show both failures with numbers** before proposing the hybrid; the arithmetic is the argument.
- **Give a concrete threshold** and justify it from what a fanout job can absorb comfortably.
- **Explain why the merge is small**: large accounts are rare, so a viewer follows only a few of them.
- **Raise deduplication and threshold crossings**, which are the specific bugs this design introduces.
- **Acknowledge the complexity cost** and say it should be adopted when a pure approach breaks, not by default.

**Practice drill**

Design a feed for a platform whose median account has 200 followers and whose largest has 50 million, with users following up to 2,000 accounts. Choose a threshold and justify it from both directions. Compute the write cost for an ordinary post and a celebrity post, and the read cost for a heavy follower. Then handle an account crossing the threshold in both directions, saying exactly how you avoid duplicates and gaps.

**Go deeper**

Hybrid fanout precomputes feeds for ordinary accounts and merges large accounts at read time, bounding both the write and read paths.

**It exists because the two pure strategies fail on opposite ends of the same distribution.** Precomputation is unbounded in followers, read-time assembly is unbounded in followings, and a real social graph is unbounded in both. Choosing either alone means a segment of users — celebrities or heavy followers — has an experience that cannot be fixed by tuning, which is why every large platform converges on a combination.

**The reason it works is a property of the graph rather than of the code.** Accounts with enormous follower counts are rare, so even a user following thousands of accounts follows only a few dozen of them. The expensive read-time merge is therefore over tens of sources rather than thousands, and stays small automatically as the platform grows — the bound comes from the distribution, not from a limit anyone has to enforce.

**The threshold is the primary control and should be measured, not guessed.** Lowering it makes writes cheaper and read merges larger; raising it does the reverse. The right value is roughly the largest fanout that the background workers absorb comfortably, and it moves as the platform's capacity and follower distribution change — which means it should be a configurable number with dashboards on both sides, not a constant chosen once.

**Its characteristic fragility is the two paths disagreeing.** Ordering, block filtering, deletion handling and deduplication must produce identical results whichever path an item came from, or feeds become subtly wrong for the users who happen to follow an account near the boundary. Concentrating all of those decisions in the merge — one place where items from both sources are ordered, filtered and deduplicated — is what keeps the divergence from accumulating.

**Threshold crossings need explicit handling in both directions.** An account growing past the limit has posts already in inboxes and new posts arriving through the read path, which produces duplicates unless the merge deduplicates by identifier. An account shrinking below it stops appearing in the read path and has no history in inboxes, which produces a gap unless something backfills. Hysteresis — different values for promotion and demotion — prevents an account at the boundary from oscillating between the two and triggering this repeatedly.

**It is a pattern to adopt under pressure rather than by default.** Two write paths, two read sources and a reconciliation layer are genuinely more code and more failure modes than either pure approach, and a product whose accounts all have comparable follower counts gains nothing from the split. The signal to introduce it is a measured failure — fanout jobs backing up behind large accounts, or feed latency scaling with following size — and arriving at it from that evidence produces a better threshold than adopting it because large platforms do.

**Related patterns:** Fanout On Write · Fanout On Read · Hot Key Mitigation · Materialized View

---

## Traffic Control

### Token Bucket

*Accumulate tokens at a fixed rate up to a cap, spending one per request, so a steady rate is enforced while short bursts are still allowed.*

> **When you hear…** A sustained rate limit that must still permit bursts · API quotas · protecting a backend from spikes

**Flow:** `Tokens refill steadily` → `Bucket has a cap` → `Request takes a token` → `Empty means reject` → `Burst absorbed`

**The problem**

A rate limit of one hundred requests per second, enforced strictly, rejects a client that sends five requests in the same millisecond and then nothing for a second — even though their average is far below the limit. Real clients batch, retry and page, so strict smoothing rejects legitimate traffic constantly.

Allowing bursts without any bound is equally wrong: a client that saves up an hour of quota and spends it in one second delivers a spike the backend was never sized for.

> **Separate the sustained rate from the permitted burst**  
> A bucket that fills at the allowed rate and holds a maximum number of tokens encodes both properties in two numbers. The refill rate bounds long-run throughput exactly; the capacity bounds how much unused allowance can be saved and spent at once. A client that has been idle can burst up to the capacity and no further, and one that is continuously active is limited precisely to the refill rate.

**Mental model**

A bucket refilled continuously at a fixed rate, capped at a maximum. Each request removes a token; a request arriving at an empty bucket is refused.

1. **Refill** — Tokens accrue at the rate limit, continuously rather than in steps.
2. **Cap** — The bucket never holds more than its capacity.
3. **Spend** — Each request consumes one token, or more for expensive operations.
4. **Refuse** — An empty bucket means rejection, usually with a retry hint.
5. **Recover** — Idle time refills the bucket, restoring burst capacity.

> **Capacity is the burst the backend must survive**  
> The bucket size is not a generosity setting; it is a promise that the system can absorb that many requests arriving simultaneously. A capacity of a thousand with ten thousand clients means ten million requests can arrive in the same instant while every client remains within its limit. Capacity must be chosen from what the backend can take, not from what feels fair.

**How it works**

**The algorithm and why it needs no timer**

```text
STATE PER KEY
  tokens        current count
  last_refill   timestamp

ON REQUEST
  now = current time
  elapsed = now - last_refill
  tokens = min(capacity, tokens + elapsed * rate)
  last_refill = now
  if tokens >= 1:
    tokens -= 1
    ALLOW
  else:
    retry_after = (1 - tokens) / rate
    REJECT with that hint

-> refill is COMPUTED on access, not by a background
   job
-> so idle keys cost nothing and there is no timer
   per client

TWO NUMBERS, TWO PROPERTIES
  rate = 100/s        long-run throughput
  capacity = 500      maximum instantaneous burst
  -> idle for 5 s, then 500 requests at once: allowed
  -> then 100/s sustained thereafter

VARIABLE COST
  cheap read      1 token
  expensive query 10 tokens
  -> the limit becomes about work, not request count
  -> far more useful than counting requests equally
```

1. **Compute refill lazily on access** — A background timer per client does not scale and is unnecessary.
2. **Choose capacity from backend burst tolerance** — It is the size of the spike you are promising to absorb.
3. **Charge expensive operations more tokens** — Counting all requests equally limits the wrong thing.
4. **Return a retry hint on rejection** — Clients that know when to return stop hammering.
5. **Decide the scope deliberately** — Per user, per key, per IP and global limits solve different problems.
6. **Account for multiple instances** — Per-instance buckets multiply the effective limit by the fleet size.

**Distributed enforcement**

```text
PER-INSTANCE BUCKETS
  limit 100/s, 10 instances
  -> each enforces 100/s
  -> the client can actually do 1000/s
  -> divide by instance count? then a client routed
     unevenly is throttled below their limit

SHARED COUNTER  (accurate)
  bucket state in Redis
  atomic script: refill, check, decrement
  -> exact enforcement
  -> a network round trip on EVERY request
  -> the limiter becomes a dependency and a
     bottleneck

LOCAL WITH PERIODIC SYNC  (practical)
  each instance holds a local allowance
  syncs with the shared view every second
  -> allows brief overshoot
  -> no per-request round trip
  -> the usual production choice

WHEN EXACTNESS MATTERS
  billing quotas      -> shared counter
  abuse prevention    -> local is fine; approximate
                         limits still stop abuse
  backend protection  -> local is fine; the goal is
                         a bound, not a number
```

| Metric | Value | Note |
|---|---|---|
| Rate | sustained limit | long-run exact |
| Capacity | burst allowed | **backend must absorb** |
| Refill | lazy on access | no timers |
| Distributed | sync periodically | slight overshoot |

> **Per-instance limiters multiply the limit by the fleet size**  
> Ten instances each enforcing a hundred per second permit a thousand per second from one client, and the limit written in the documentation is simply not the limit in effect. Dividing by the instance count fixes the arithmetic and breaks fairness when routing is uneven, which is why production systems usually keep a shared view synchronised periodically and accept brief overshoot as the price of not adding a round trip to every request.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| API gateways | Built-in token bucket limiting | Usually the right place; no application code |
| Redis with an atomic script | Shared, accurate buckets | A round trip per request |
| In-process libraries | Per-instance buckets | Fast; multiplies by fleet size |
| Service meshes | Per-route limits | Applied without touching the application |
| Cloud provider quotas | Managed, per-account | The model most public APIs expose |
| Leaky bucket | Smooths instead of bursting | Different shape, different purpose |

The gateway is usually the correct layer: it sees every request before any backend work happens, its state is already shared across the fleet, and rejecting there costs almost nothing compared with rejecting after a service has already begun processing.

**Trade-offs**

**Rate limiting algorithms**

| Algorithm | Allows bursts | Memory per key | Accuracy |
|---|---|---|---|
| Token bucket | Yes, up to capacity | Two values | Exact over the long run |
| Leaky bucket | No, smooths output | Queue or counter | Exact output rate |
| Fixed window | Yes, at boundaries | One counter | Poor; 2× at edges |
| Sliding window log | No | One entry per request | Exact |
| Sliding window counter | Slightly | Two counters | Good approximation |

Token bucket is the usual default because it matches how clients actually behave — idle periods followed by short bursts — and because its two parameters map directly onto the two things the operator cares about: sustained load and peak instantaneous load.

> **Ask before choosing it**  
> What is the limit protecting? A backend that cannot absorb bursts needs smoothing rather than a large capacity; a billing quota needs exactness and a shared counter; abuse prevention needs neither and can happily run locally with approximate numbers.

**How it fails**

**How token buckets go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Effective limit is far higher than documented | Per-instance buckets | Shared state or periodic synchronisation |
| Backend overwhelmed despite limits | Capacity larger than burst tolerance | Size capacity from what the backend absorbs |
| Clients retry immediately on rejection | No retry hint returned | Return the wait time with the rejection |
| Expensive queries exhaust the backend | All requests cost one token | Variable token cost by operation |
| The limiter becomes the bottleneck | Shared counter round trip per request | Local allowance with periodic sync |
| Legitimate bursts rejected | Capacity too small for real client behaviour | Measure actual burst patterns |
| One client starves others | Global limit with no per-client scope | Per-client buckets beneath a global cap |

> **A rate limiter that fails closed takes the service down with it**  
> If the shared store holding bucket state becomes unavailable and the limiter rejects everything it cannot verify, an outage in the limiter becomes a total outage of the service it was protecting. Failing open risks a brief period of unlimited traffic; failing closed guarantees zero traffic. For most systems the first is clearly preferable, and the decision must be made explicitly rather than inherited from whatever the client library does on error.

**Where it is used**

- **Public API quotas**, where burst allowance plus sustained rate is the standard published model.
- **API gateways**, enforcing per-key limits before any backend work is done.
- **Network traffic shaping**, the origin of both bucket algorithms.
- **Internal service protection**, bounding what one caller can demand of a shared backend.
- **Cost control on expensive operations**, where variable token costs limit work rather than requests.

**In the interview**

The two parameters and what each one means to the backend is the substance here.

- **Name the two properties**: refill rate bounds sustained throughput, capacity bounds the instantaneous burst.
- **Say capacity is a promise about burst absorption**, not a fairness setting, and size it from the backend.
- **Describe lazy refill** — computed on access, no timers — which is what makes it cheap at scale.
- **Raise distributed enforcement** and the fleet-multiplication problem, then choose local with sync and justify the overshoot.
- **Decide fail-open versus fail-closed explicitly**, because a limiter that fails closed converts its own outage into a total one.

**Practice drill**

An API offers 1,000 requests per minute per key. Choose the refill rate and capacity, and state what burst behaviour each permits. Then deploy it across 20 instances: describe three enforcement options, pick one, and quantify the overshoot it allows. Finally, decide what happens when the shared state is unavailable, and justify your choice in terms of what the limit is protecting.

**Go deeper**

A token bucket enforces a sustained rate while permitting bounded bursts, by accruing tokens at the limit rate up to a maximum.

**Its two parameters map cleanly onto the two things that matter.** The refill rate determines long-run throughput exactly, and the capacity determines how much unused allowance can be spent at once. That separation is what makes the algorithm fit real clients, which are rarely smooth — they batch, retry, paginate and idle — and which a strictly smoothed limiter rejects while they remain well under their average allowance.

**Capacity is a statement about the backend, not about generosity.** Whatever the bucket size permits can arrive simultaneously, multiplied by the number of clients, so a large capacity chosen to be accommodating is a commitment to absorb a proportionally large spike. Sizing it from measured burst tolerance rather than from what seems reasonable is what keeps the limiter protective rather than merely nominal.

**Lazy refill is what makes it cheap.** Computing accrued tokens from elapsed time on each access means there are no timers, no background jobs and no cost for idle keys — state is two numbers per client, updated only when that client is active. This is why token buckets scale to millions of keys where implementations based on scheduled refills do not.

**Distributed enforcement forces a choice between accuracy and latency.** Per-instance buckets multiply the effective limit by the fleet size, a shared counter adds a round trip to every request and makes the limiter a dependency of everything it protects, and local allowances synchronised periodically permit brief overshoot while costing nothing per request. The third is the usual answer, and it is correct because most limits exist to bound load rather than to be exact — billing quotas being the notable exception.

**Charging by cost rather than by request makes the limit meaningful.** Treating a cheap lookup and an expensive aggregation as one unit each limits the wrong quantity: a client within its request limit can still exhaust the backend by choosing expensive operations. Variable token costs turn the bucket into a budget for work, which is what the backend actually has a finite amount of.

**Its failure behaviour deserves an explicit decision.** When the state store is unreachable, a limiter that rejects what it cannot verify converts its own outage into a total outage of the protected service, while one that allows traffic through risks a window of unlimited load. For nearly all systems the second risk is smaller, but the choice is frequently left to a client library's default — which means the behaviour under the limiter's own failure has never actually been chosen.

**Related patterns:** Leaky Bucket · Sliding Window Rate Limit · Load Shedding · Backpressure

---

### Leaky Bucket

*Queue incoming requests and release them at a constant rate, converting bursty arrivals into a perfectly smooth output stream.*

> **When you hear…** A downstream system that cannot absorb bursts · a strict output rate required · smoothing matters more than latency

**Flow:** `Requests arrive bursty` → `Enter a bounded queue` → `Drain at fixed rate` → `Output is smooth` → `Overflow rejected`

**The problem**

A backend can process exactly two hundred operations per second and degrades badly above that. A token bucket with any meaningful capacity permits a burst that exceeds it, and the backend's failure under overload is worse than the delay that queueing would cause.

Some downstreams have hard rate constraints that are not about capacity at all — a third-party API with a contractual limit, a device with a fixed processing rate, a system whose behaviour beyond a threshold is undefined.

> **Smooth the output rather than policing the input**  
> If requests are placed in a queue that drains at a constant rate, the arrival pattern stops mattering: whatever shape the traffic has going in, the output is a steady stream at exactly the configured rate. The cost is latency for requests that wait, and rejection only when the queue itself fills — which converts burstiness into delay rather than into failure.

**Mental model**

A bounded queue with a constant-rate drain. Arrivals join the queue if there is room; the drain releases work at a fixed interval regardless of how much is waiting.

1. **Arrive** — A request joins the queue if space remains.
2. **Overflow** — A full queue means immediate rejection.
3. **Drain** — Work is released at the configured rate, no faster.
4. **Smooth** — Output is constant regardless of the arrival pattern.
5. **Wait** — Queued requests experience latency proportional to their position.

> **Queued requests may have already been abandoned**  
> A request waiting three seconds in a smoothing queue behind a burst may face a client that timed out after two. Processing it does work for nobody, and in a sustained burst the queue can fill entirely with requests whose callers have gone. Bounding the queue by time — dropping anything older than the client's timeout — is what prevents the smoother from spending all its capacity on abandoned work.

**How it works**

**Smoothing versus bursting**

```text
ARRIVALS  (bursty)
  t=0.0  50 requests
  t=0.5   0
  t=1.0  50 requests
  t=1.5   0

TOKEN BUCKET  (rate 100/s, capacity 100)
  t=0.0  all 50 pass immediately
  t=1.0  all 50 pass immediately
  -> output is as bursty as the input
  -> the backend sees 50 at once, twice

LEAKY BUCKET  (rate 100/s, queue 100)
  t=0.0  50 queued
  drain releases one every 10 ms
  -> output: a steady 100/s
  -> the backend never sees more than one at a time
  -> the 50th request waits 500 ms

THE TRADE
  token bucket   low latency, bursty output
  leaky bucket   smooth output, added latency

QUEUE SIZING
  queue = rate x acceptable_wait
  100/s, 1 s acceptable wait -> queue of 100
  -> a larger queue does not add throughput; it only
     adds waiting
```

1. **Size the queue from acceptable latency, not from memory** — Queue depth converts directly into waiting time at the drain rate.
2. **Drop requests older than the client timeout** — Processing abandoned work consumes the capacity live requests need.
3. **Reject immediately when the queue is full** — Fast rejection is better than an unbounded wait.
4. **Use it only where smoothing is the requirement** — Where bursts are harmless, the added latency buys nothing.
5. **Consider priority within the queue** — Not all waiting work is equally worth the remaining capacity.
6. **Monitor queue depth and wait time** — Depth alone does not say whether anything is still being served usefully.

**Where each bucket belongs**

```text
USE LEAKY BUCKET WHEN
  the downstream has a HARD rate limit
    third-party API with a contractual cap
  bursts cause disproportionate damage
    a backend that collapses above its rate
  the output rate itself is the requirement
    device control, media streaming, egress shaping
  delay is preferable to rejection
    background jobs, uploads, notifications

USE TOKEN BUCKET WHEN
  clients are interactive and latency matters
  bursts are normal and absorbable
  rejecting is better than delaying
  you are limiting a caller rather than protecting a
  fragile downstream

COMBINE THEM
  token bucket at the edge   (limit each client)
  leaky bucket before a
  fragile dependency         (smooth the aggregate)
  -> clients get burst tolerance; the dependency
     gets a steady stream

THE QUEUE IS THE DIFFERENCE
  token bucket has no queue: allowed or rejected
  leaky bucket has a queue: delayed or rejected
  -> which is acceptable depends entirely on whether
     the caller is waiting
```

| Metric | Value | Note |
|---|---|---|
| Output | perfectly smooth | **constant rate** |
| Cost | added latency | proportional to depth |
| Queue size | rate × wait | not memory-driven |
| Best for | fragile downstream | or hard caps |

> **A larger queue adds waiting, not throughput**  
> The drain rate is fixed, so doubling the queue does not process more work — it only allows twice as many requests to wait twice as long. Teams that respond to overflow rejections by increasing the queue convert a fast, honest failure into slow, invisible timeouts, and the arrival rate that exceeded the drain rate still exceeds it. The only real remedies are raising the drain rate or reducing arrivals.

**Technologies**

| Context | Implementation | Note |
|---|---|---|
| Network shaping | Traffic shapers and queue disciplines | The original application |
| Message consumers | Fixed-rate polling from a queue | The queue is already there |
| Outbound API calls | Scheduler releasing at the allowed rate | Respects a third party's contractual limit |
| Job processing | Worker pool with a fixed dispatch rate | Smooths load on shared databases |
| Token bucket | The bursty alternative | Different shape, different purpose |
| Backpressure | Slow the producer instead | Better where the producer can be slowed |

Backpressure deserves the comparison: where the producer is under your control, telling it to slow down is superior to queueing its output, because it moves the problem to a place that can actually reduce demand rather than storing it.

**Trade-offs**

**Shaping against limiting**

| Property | Leaky bucket | Token bucket |
|---|---|---|
| Output shape | Constant rate | As bursty as the input |
| Excess handling | Queued, then rejected | Rejected immediately |
| Latency | Increases with queue depth | Unaffected |
| Memory | Queue of pending work | Two numbers |
| Best for | Protecting fragile downstreams | Limiting client behaviour |
| Client experience | Slow but served | Fast success or fast failure |

The last row is usually the deciding one. An interactive user prefers a fast rejection they can retry over a request that eventually succeeds after four seconds; a background job prefers the opposite, which is why the two algorithms tend to appear at different layers of the same system.

> **Ask before choosing it**  
> Is anyone waiting for this request? Smoothing suits work whose latency nobody observes — uploads, notifications, batch jobs — and suits interactive requests badly, because the queue converts their burst into a wait they experience directly.

**How it fails**

**How leaky buckets go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Capacity spent on abandoned requests | No expiry on queued work | Drop entries older than the client timeout |
| Timeouts instead of rejections | Queue enlarged to avoid overflow | Size from acceptable wait; reject beyond it |
| Memory exhausted | Unbounded queue | A bound is mandatory |
| Interactive requests feel broken | Smoothing applied to user-facing traffic | Token bucket at the user-facing edge |
| Throughput lower than expected | Drain rate below actual capacity | Measure the downstream's real rate |
| Important work stuck behind bulk | Single undifferentiated queue | Priority classes within the queue |
| Problem hidden until collapse | Only depth monitored | Track wait time and drop rate as well |

> **Enlarging the queue to stop rejections converts failure into latency**  
> Overflow rejections are a signal that arrivals exceed the drain rate, and a bigger queue does not change that arithmetic — it only lets more requests accumulate before the same rejection happens, with every one of them waiting longer first. The visible symptom shifts from honest errors to client timeouts, which are harder to diagnose and worse for the user, while the underlying imbalance is untouched.

**Where it is used**

- **Network traffic shaping**, where constant-rate output is the defining requirement.
- **Outbound calls to third-party APIs**, releasing requests at exactly the contractual rate.
- **Job dispatch to shared databases**, smoothing bursts that would otherwise spike load.
- **Media streaming and egress control**, where the output rate is the product requirement.
- **Notification delivery**, spreading a large batch over time rather than sending it all at once.

**In the interview**

Define it against the token bucket; the contrast is what makes each one's purpose clear.

- **Give the distinguishing property**: output is smooth regardless of input, which a token bucket does not provide.
- **Say the queue is the difference** — delayed rather than rejected — and that this is only acceptable when nobody is waiting.
- **Size the queue from acceptable wait time**, and state that a bigger queue adds latency rather than throughput.
- **Raise abandoned requests.** Work queued past the client's timeout consumes capacity for nobody.
- **Place both in one system**: token bucket at the client edge, leaky bucket in front of a fragile dependency.

**Practice drill**

A third-party API allows exactly 100 calls per second and your system generates bursts of 500. Design the smoothing layer: drain rate, queue size, overflow behaviour and expiry policy. Compute how long the last request in a 500-item burst waits. Then decide whether this design suits a user-facing request path, and if not, say what you would use there instead and why.

**Go deeper**

A leaky bucket converts bursty arrivals into a constant-rate output stream by queueing work and releasing it at a fixed interval.

**Its defining property is output shape, which is what distinguishes it from a token bucket.** A token bucket decides whether each request is allowed and passes it through immediately, so the downstream sees whatever burst pattern the client produced. A leaky bucket decides when each request happens, so the downstream sees a steady stream no matter what arrived. Where a dependency's problem is burstiness rather than volume, only the second addresses it.

**It trades rejection for delay, which is only acceptable when nobody is waiting.** Queued work is eventually served rather than refused, which suits uploads, notifications and background jobs where latency is unobserved. Applied to an interactive path the same behaviour is worse than rejecting: the user experiences the queue directly as an unresponsive application, and a fast error they can retry would have been preferable.

**Queue depth is a latency parameter disguised as a capacity parameter.** At a fixed drain rate, depth converts directly into waiting time, so sizing it should start from how long a request may acceptably wait and work backwards. Enlarging the queue in response to overflow rejections is the characteristic mistake: it does not increase throughput, it converts honest fast failures into client timeouts, and the imbalance between arrival and drain rates remains exactly as it was.

**Abandoned work is a real and underestimated cost.** During a sustained burst the queue fills with requests whose clients have already timed out, and the drain spends its entire fixed capacity producing results nobody will receive — while newly arriving live requests are rejected for lack of space. Expiring entries older than the client timeout keeps capacity pointed at work that still matters, and it is the difference between a smoother that degrades gracefully and one that stops being useful precisely under load.

**It complements rather than competes with the token bucket, and mature systems use both.** A token bucket at the client edge gives each caller burst tolerance and rejects excess quickly, which is right for interactive traffic. A leaky bucket in front of a fragile dependency smooths the aggregate of all callers into the steady stream that dependency requires. They sit at different layers because they solve different halves of the problem: one bounds what a caller may demand, the other bounds what the dependency must survive.

**Where the producer can be slowed, backpressure is better than smoothing.** A queue stores excess demand; backpressure removes it, by making the producer wait rather than accumulating its output. Smoothing is the right answer when the producer is outside your control — an external client, a third-party integration — and the second-best answer when it is not, because storing demand always risks storing more of it than the system can eventually serve.

**Related patterns:** Token Bucket · Backpressure · Work Queue · Load Shedding

---

### Fixed Window Rate Limit

*Count requests within aligned time windows and reject beyond the limit, the simplest possible limiter and the one with a known boundary flaw.*

> **When you hear…** A rate limit needed quickly · minimal state acceptable · approximate enforcement sufficient

**Flow:** `Window starts` → `Counter increments` → `Limit reached` → `Reject until reset` → `Counter clears`

**The problem**

A limit has to be enforced with as little state and complexity as possible. Tracking individual request times per client costs memory proportional to traffic, and maintaining continuously refilling state is more machinery than a simple quota seems to warrant.

The straightforward answer — count requests in each minute and reset the count when the minute changes — is trivially cheap and has a specific, well-known defect that must be understood before it is chosen.

> **One counter per window is the cheapest limiter that works at all**  
> A single integer and an expiry per client is the minimum viable state for rate limiting, and it enforces the limit exactly within each window. The flaw is at the boundaries: because the counter resets instantly, a client can spend a full window's quota just before the reset and another immediately after, achieving twice the limit across a span shorter than one window.

**Mental model**

Time divided into aligned windows. Each client has one counter per window, incremented per request and discarded when the window ends.

1. **Bucket** — Requests are attributed to the current aligned window.
2. **Count** — Each request increments that window's counter.
3. **Compare** — Beyond the limit, requests are rejected.
4. **Reset** — The counter disappears when the window rolls over.
5. **Repeat** — The next window starts from zero, with no memory of the last.

> **Twice the limit can pass across a window boundary**  
> A client sending its full quota in the last moments of one window and again in the first moments of the next has sent double the limit within a span far shorter than the window. The limiter records two compliant windows and the backend experiences one spike of twice the intended size — which is the entire reason more elaborate algorithms exist.

**How it works**

**The counter and the boundary problem**

```text
IMPLEMENTATION
  key = client_id + ":" + floor(now / window)
  count = INCR key
  if count == 1: EXPIRE key window
  if count > limit: REJECT
  -> two Redis operations, one integer of state
  -> keys expire themselves; no cleanup needed

THE BOUNDARY
  limit 100 per minute

  10:00:59  client sends 100 requests  -> allowed
            (window 10:00 count = 100)
  10:01:00  window rolls over, count = 0
  10:01:00  client sends 100 requests  -> allowed
            (window 10:01 count = 100)

  -> 200 requests within ~1 second
  -> both windows appear compliant
  -> the backend saw 2x the limit instantaneously

HOW BAD IS IT
  worst case is exactly 2x the limit, for a duration
  approaching zero
  -> if the backend can absorb 2x briefly, the flaw
     is acceptable
  -> if it cannot, use a sliding window

WHY IT PERSISTS
  one integer per client per window
  self-expiring keys
  two operations per request
  -> nothing else is this cheap
```

1. **Use aligned windows derived from the timestamp** — It avoids storing a window start time per client.
2. **Let the key expire itself** — Self-expiring counters remove all cleanup work.
3. **Account for the doubling in capacity planning** — Size the backend for twice the limit, or choose a different algorithm.
4. **Keep windows short where the doubling matters** — A shorter window reduces the absolute size of the boundary burst.
5. **Return the time until reset** — Clients can then wait correctly rather than retrying blindly.
6. **Reserve it for cases where approximation is fine** — Billing and hard caps need something exact.

**When it is good enough**

```text
ACCEPTABLE
  abuse prevention
    a scraper limited to 2x instead of 1x is still
    effectively stopped
  coarse API quotas
    documented as approximate
  internal services with generous headroom
  very high volume where per-request state matters
    millions of keys: memory per key is the
    binding constraint

NOT ACCEPTABLE
  billing quotas
    charging for 200 when the plan allows 100
  hard third-party limits
    the provider will reject or ban regardless of
    your accounting
  fragile backends
    a 2x spike causes real damage

MITIGATION WITHOUT CHANGING ALGORITHM
  window 1 minute, limit 100   -> burst up to 200
  window 1 second, limit ~2    -> burst up to 4
  -> shorter windows shrink the absolute burst
  -> but reject legitimate bursty clients more often

THE UPGRADE PATH
  sliding window counter costs one extra counter and
  removes most of the flaw
  -> usually the right move once the flaw matters
```

| Metric | Value | Note |
|---|---|---|
| State | one integer | **per key per window** |
| Worst case | 2× the limit | at a boundary |
| Cost | two operations | cheapest available |
| Upgrade | sliding counter | one extra counter |

> **Aligned windows synchronise every client's reset**  
> When all windows begin on the minute, every client's quota refreshes simultaneously, and clients that were being throttled all resume at once. The result is a traffic spike at each window boundary across the whole user base — the boundary problem multiplied by client count. Offsetting each client's window by a hash of their identifier spreads the resets and removes the synchronised surge.

**Technologies**

| Layer | Implementation | Note |
|---|---|---|
| Redis | INCR with EXPIRE | Two commands; the standard approach |
| API gateways | Built-in fixed-window limiting | Often the default offering |
| In-memory per instance | A map of counters | Multiplies the limit by the fleet size |
| Sliding window counter | Two windows weighted | The natural upgrade |
| Token bucket | Continuous refill | No boundary flaw; slightly more state |
| CDN and edge limits | Applied before origin | Cheapest place to reject |

The sliding window counter is worth knowing as the upgrade because it costs almost nothing extra — one additional counter and a weighted calculation — and removes the doubling, which makes the fixed window's simplicity advantage much narrower than it first appears.

**Trade-offs**

**Limiter algorithms by cost and accuracy**

| Algorithm | State per key | Boundary flaw | Operations per request |
|---|---|---|---|
| Fixed window | One counter | Up to 2× | Two |
| Sliding window counter | Two counters | Small | Two to three |
| Sliding window log | One entry per request | None | Several |
| Token bucket | Two values | None | Two to three |
| Leaky bucket | A queue | None; smooths output | Queue operations |

Ranked by what they cost and what they give, the fixed window is only the right choice when either the boundary flaw genuinely does not matter or the per-key memory difference is decisive — which usually means tens of millions of keys.

> **Ask before choosing it**  
> Can the protected system absorb twice the limit briefly? If yes, the simplicity is free and the flaw is theoretical. If no, the sliding window counter costs one more integer and removes the problem, which is a trade almost always worth making.

**How it fails**

**How fixed window limiting goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Backend sees twice the limit | Boundary burst | Sliding window, or plan for 2× |
| Fleet-wide spike every minute | Aligned windows across all clients | Offset each client's window by a hash |
| Billing disputes | Approximate counting used for charging | Exact algorithm for anything billable |
| Third party rejects traffic | Their limit enforced differently from yours | Match their algorithm, or stay well below |
| Effective limit multiplied | Per-instance counters | Shared counter store |
| Clients retry immediately | No reset time returned | Return seconds until the window rolls over |
| Legitimate clients throttled | Window too short for real usage patterns | Longer window, or token bucket for burst tolerance |

> **Two systems counting differently will disagree about compliance**  
> If a provider enforces a sliding window and you account with a fixed one, your records will show compliance at moments when theirs show a violation — and their view is the one that results in rejected traffic or a suspended key. Matching the upstream's algorithm, or keeping a margin large enough that the discrepancy cannot matter, is the only way to make local accounting mean anything.

**Where it is used**

- **Simple API rate limits**, where a documented approximate quota is sufficient.
- **Abuse and scraping prevention**, where twice the limit is still an effective bound.
- **Gateway default configurations**, which very often implement exactly this algorithm.
- **Very high-cardinality limiting**, where per-key memory is the binding constraint.
- **Internal service quotas**, where headroom makes the boundary flaw immaterial.

**In the interview**

Name the boundary flaw immediately and quantify it; describing the algorithm without it reads as unfamiliarity.

- **State the implementation in one line** — a counter keyed by client and window, self-expiring — to show why it is chosen.
- **Give the boundary example with times and numbers**, and say the worst case is exactly twice the limit.
- **Judge whether that matters** by asking what the backend can absorb, rather than dismissing the algorithm outright.
- **Mention synchronised resets** across clients, which multiplies the boundary spike fleet-wide.
- **Offer the sliding window counter as a cheap upgrade**, since it removes the flaw for one additional integer.

**Practice drill**

Implement a limit of 100 requests per minute using fixed windows, writing out the key structure and the operations per request. Construct the worst-case boundary scenario with explicit timestamps and say how many requests reach the backend and over what interval. Then decide whether to keep it for an endpoint that can absorb 300 requests per second, and describe the smallest change that would remove the flaw.

**Go deeper**

A fixed window rate limiter counts requests per aligned time window and rejects once the count exceeds the limit.

**It is the minimum viable limiter, and that is its entire argument.** One integer per client per window, two operations per request, and keys that expire themselves with no cleanup — nothing else provides enforcement this cheaply. At very high cardinality, where per-key memory across millions of clients is the binding constraint, that economy can be decisive in a way it is not at smaller scale.

**Its flaw is precise and bounded, which makes it possible to reason about.** Because the counter resets instantly at the boundary, a client can spend a full quota immediately before and another immediately after, delivering twice the limit in an arbitrarily short span. The worst case is exactly double, never more, so the question becomes concrete: can the protected system absorb twice the configured rate briefly? If it can, the flaw is theoretical.

**Aligned windows compound the problem across clients.** When every window begins on the same second, every throttled client resumes simultaneously and the boundary spike is multiplied by the number of clients waiting. Offsetting each client's window by a hash of their identifier spreads the resets across the window and removes the synchronised surge at no cost, and it is omitted far more often than it should be.

**Approximate counting is unsuitable wherever the number has consequences.** Billing against a quota that can be exceeded by half is indefensible, and accounting locally with a fixed window against a provider enforcing a sliding one produces records that show compliance at the moments they are rejecting your traffic. Approximation is acceptable for bounding load and preventing abuse, and not for anything where the count is the product.

**The upgrade is cheap enough to narrow its advantage considerably.** A sliding window counter keeps one extra integer and weights the previous window by how much of it remains in view, which removes nearly all of the boundary error for one more value and one more arithmetic step. Once the doubling matters at all, that trade is almost always worth making — which confines the fixed window to cases where the flaw is genuinely irrelevant.

**Choosing it should be a decision, not a default.** It is frequently what a gateway implements out of the box, so systems end up with it without anyone having considered the boundary behaviour or the synchronised resets. Knowing that both exist, and having concluded that the protected system tolerates them, is the difference between an appropriate choice and an accidental one that will be discovered during a traffic spike.

**Related patterns:** Sliding Window Rate Limit · Token Bucket · Leaky Bucket · Load Shedding

---

### Sliding Window Rate Limit

*Evaluate the limit over a window that moves with the present moment, removing the boundary burst that fixed windows permit.*

> **When you hear…** Fixed-window boundary bursts causing real load · limits that must be accurate at every instant · third-party limits enforced this way

**Flow:** `Window ends now` → `Count recent requests` → `Old ones age out` → `No reset boundary` → `Smooth enforcement`

**The problem**

A fixed window lets a client send its whole quota just before the reset and again just after, producing twice the limit in a fraction of a second. For a backend that cannot absorb that spike, the limiter is not providing the bound it claims.

The underlying reason is that a fixed window asks the wrong question. It asks how many requests occurred since an arbitrary aligned instant, when the meaningful question is how many occurred in the last minute counted from now.

> **Let the window end at the present moment, not at a clock boundary**  
> A window anchored to now has no reset, so there is no instant at which allowance suddenly reappears. Requests age out continuously as they pass beyond the window's trailing edge, which means the limit holds over every possible interval of that length rather than only over the aligned ones — and the boundary burst becomes impossible by construction.

**Mental model**

Two implementations of one idea: an exact log of recent request times, or an approximation that weights the previous fixed window by how much of it remains in view.

1. **Anchor** — The window always ends at the current instant.
2. **Include** — Requests within the trailing window count toward the limit.
3. **Expire** — Requests older than the window stop counting, continuously.
4. **Decide** — The count over that moving window determines allow or reject.
5. **Approximate** — Where exactness is too costly, weight the previous window instead.

> **The exact version stores one entry per request**  
> Keeping a timestamp for every request gives perfect accuracy and memory proportional to the limit multiplied by the number of clients. A limit of a thousand per minute across a million clients is a billion timestamps — accuracy that is unaffordable at scale, which is why the weighted approximation exists and why it is what most systems actually run.

**How it works**

**Exact log and weighted counter**

```text
SLIDING WINDOW LOG  (exact)
  store a sorted set of request timestamps per client
  on request:
    remove entries older than now - window
    count remaining
    if count < limit: add now, ALLOW
    else: REJECT
  + exact at every instant
  - memory = limit x clients
  - several operations per request

SLIDING WINDOW COUNTER  (approximate, common)
  keep counts for the current and previous fixed
  windows
  weight the previous one by how much of it is still
  within view:

    now is 30% into the current minute
    estimate = current + previous x 0.7

    previous minute: 90 requests
    current minute:  20 requests
    estimate = 20 + 90 x 0.7 = 83
    limit 100 -> ALLOW

  + two counters per client
  + error is small and bounded
  - assumes requests were evenly spread in the
    previous window

THE ERROR
  if the previous window's requests were all at its
  very start, the estimate overcounts
  if all at its end, it undercounts
  -> in practice within a few percent
  -> vastly better than 2x
```

1. **Use the weighted counter unless exactness is required** — Two integers per client instead of one entry per request.
2. **Reserve the exact log for low-cardinality, high-value limits** — Billing and hard external caps justify the memory.
3. **Understand the approximation's assumption** — It presumes even distribution within the previous window.
4. **Match the algorithm a third party uses** — Accounting differently from the enforcer produces surprises.
5. **Return a precise retry time** — The sliding window can compute exactly when capacity returns.
6. **Keep the state shared across instances** — Per-instance windows multiply the limit as with any limiter.

**Cost comparison at scale**

```text
1,000,000 CLIENTS, limit 1,000/minute

SLIDING WINDOW LOG
  worst case 1,000 timestamps x 1,000,000 clients
  = 1,000,000,000 entries
  at ~16 B with overhead -> tens of GB
  -> and several sorted-set operations per request

SLIDING WINDOW COUNTER
  2 integers x 1,000,000 clients
  = 2,000,000 values, a few hundred MB
  -> two reads and one write per request

FIXED WINDOW
  1 integer x 1,000,000 = 1,000,000 values
  -> cheapest, with the 2x flaw

-> the counter is close to the fixed window in cost
   and close to the log in accuracy
-> which is why it is the usual production choice

WHEN THE LOG EARNS ITS COST
  few clients, expensive requests
  billing where every unit is charged
  regulatory or contractual exactness
```

| Metric | Value | Note |
|---|---|---|
| Accuracy | exact or near | **no 2× burst** |
| Log cost | entry per request | tens of GB |
| Counter cost | two integers | few hundred MB |
| Usual choice | weighted counter | best trade |

> **The weighted estimate assumes the previous window was evenly distributed**  
> If a client sent all of the previous window's requests in its final second, the weighting underestimates their recent activity and allows more than intended; if they arrived at the very start, it overestimates and throttles too early. The error is bounded and small in practice, but it exists — and describing the counter as a sliding window without mentioning that it is an approximation misrepresents what it guarantees.

**Technologies**

| Approach | Implementation | Note |
|---|---|---|
| Sliding window log | Redis sorted set of timestamps | Exact; memory proportional to the limit |
| Sliding window counter | Two Redis counters plus weighting | The common production choice |
| Gateway limiters | Often sliding by default | Check which variant is in use |
| Token bucket | Continuous refill | Also has no boundary flaw; allows bursts by design |
| Fixed window | Single counter | Cheapest; 2× burst at boundaries |
| Cloud API limits | Usually sliding | Match their model in your own accounting |

The token bucket row is worth pausing on: it also avoids the boundary problem, but it deliberately permits bursts up to its capacity, whereas a sliding window enforces the limit uniformly over every interval. Which behaviour is wanted depends on whether client bursts are welcome or hazardous.

**Trade-offs**

**Window strategies**

| Approach | Accuracy | Memory | Burst behaviour |
|---|---|---|---|
| Fixed window | Poor at boundaries | One counter | Up to 2× the limit |
| Sliding window log | Exact | One entry per request | None |
| Sliding window counter | Within a few percent | Two counters | Very small |
| Token bucket | Exact over the long run | Two values | Up to capacity, by design |
| Leaky bucket | Exact output rate | A queue | None; smoothed |

Between the sliding counter and the token bucket the question is intent rather than accuracy: the sliding window says no interval of this length may exceed the limit, while the token bucket says a client may save allowance and spend it in a burst. Both are defensible; they encode different policies.

> **Ask before choosing it**  
> How is the limit you are mirroring actually enforced? When an upstream provider uses a sliding window, accounting locally with a fixed one guarantees that your records and their enforcement disagree — and theirs is the one that rejects your traffic.

**How it fails**

**How sliding window limiting goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Memory exhausted | Exact log at high cardinality | Use the weighted counter |
| Latency added to every request | Several sorted-set operations per call | Counter variant, or limit at the gateway |
| Slightly over the limit | Uneven distribution in the previous window | Accept it, or use the exact log |
| Limit multiplied by instance count | Per-instance state | Shared store for window state |
| Clients retry too early | Generic retry advice | Compute the exact time capacity returns |
| Disagreement with an upstream limiter | Different algorithm from the provider | Match theirs, or keep a margin |
| Old entries never removed | Log not pruned on access | Prune before counting, every time |

> **Exact sliding windows scale with traffic, not with clients**  
> The log holds an entry per request within the window, so its memory grows with how much traffic the system receives rather than with how many clients exist. A busy period therefore increases the limiter's memory consumption at precisely the moment the system is under load — which is a failure mode that does not exist in any counter-based algorithm and is the practical reason the exact variant is rarely deployed at scale.

**Where it is used**

- **Cloud provider API limits**, which are commonly enforced with sliding windows.
- **Payment and financial APIs**, where exactness justifies the cost of a log.
- **Gateway rate limiting**, where the counter variant is a standard offering.
- **Abuse prevention at high cardinality**, using the weighted counter for millions of keys.
- **Any system mirroring a third party's limit**, where matching their algorithm avoids disagreement.

**In the interview**

Present both variants and choose between them with numbers; naming only one suggests a partial picture.

- **Explain why the window must end at now** — there is no reset, so no instant when allowance reappears.
- **Give both implementations** and their costs, then choose the weighted counter for scale.
- **Work the weighting arithmetic aloud** with a concrete example; it is short and demonstrates real familiarity.
- **State the approximation's assumption** about even distribution, and that the error is a few percent rather than 2×.
- **Contrast with the token bucket** on policy: uniform enforcement against deliberate burst allowance.

**Practice drill**

Enforce 100 requests per minute with a sliding window. Write out the weighted counter calculation for a client 30 seconds into the current window with 90 requests in the previous one and 20 so far. Then compute the memory for the exact log and for the counter across a million clients, and choose one. Finally, construct the traffic pattern that produces the largest error in the approximation, and say how far off it is.

**Go deeper**

A sliding window rate limiter evaluates the limit over an interval ending at the present moment, eliminating the reset boundary that fixed windows create.

**Moving the window's end to now changes what the limit means.** A fixed window bounds requests since an arbitrary aligned instant; a sliding window bounds requests over every interval of that length. Because allowance is never restored all at once, the doubling that fixed windows permit becomes impossible, and the limit describes the traffic the backend actually experiences rather than an accounting artefact.

**The exact implementation is correct and rarely affordable.** Storing a timestamp per request gives perfect enforcement at every instant, at a memory cost proportional to the limit times the client count — and, importantly, proportional to traffic rather than to clients, so consumption rises exactly when the system is busiest. That combination confines it to low-cardinality, high-value limits where every unit matters.

**The weighted counter captures most of the benefit for almost none of the cost.** Keeping the current and previous window counts and weighting the previous by how much of it remains in view costs two integers per client and one extra arithmetic step. The error is a few percent rather than a factor of two, which is close enough to exact for every purpose except billing — and that is why it, rather than either pure form, is what production systems typically run.

**The approximation makes an assumption worth stating.** Weighting presumes the previous window's requests were spread evenly through it, so a client that concentrated them at one end is measured slightly wrongly in one direction or the other. Describing the counter as a sliding window without that caveat overstates the guarantee, and knowing which direction the error runs under a given traffic shape is what lets an operator judge whether it matters.

**Matching the enforcing party's algorithm matters more than the algorithm's abstract quality.** When a provider enforces a sliding window and a client accounts with a fixed one, the two views disagree precisely at the boundaries, and the client's records will show compliance while their traffic is being rejected. Local accounting is only useful if it models the same thing the enforcer models, or keeps a margin wide enough that the difference cannot surface.

**It encodes a different policy from the token bucket, and the choice should follow intent.** Both avoid boundary bursts, but a sliding window enforces the limit uniformly across every interval, while a token bucket deliberately lets an idle client accumulate allowance and spend it at once. A backend that cannot survive bursts wants the first; an API designed to accommodate clients that batch and page wants the second — and picking on accuracy grounds alone misses that they are answering different questions.

**Related patterns:** Fixed Window Rate Limit · Token Bucket · Leaky Bucket · Load Shedding

---

### Backpressure

*Signal upstream to slow down when a system cannot keep up, so demand is reduced at the source instead of accumulating in queues.*

> **When you hear…** Queues growing without bound · memory pressure from buffered work · a producer faster than its consumer

**Flow:** `Consumer saturated` → `Signal upstream` → `Producer slows` → `Queues stay bounded` → `System stays stable`

**The problem**

A producer generates work faster than the consumer can process it. The difference accumulates in a buffer, and if nothing intervenes the buffer grows until memory is exhausted — at which point the process dies and everything buffered is lost.

Adding a bigger buffer postpones this and makes it worse. Latency grows with queue depth, so by the time the buffer is large enough to matter, every item in it has been waiting longer than anyone is prepared to accept, and much of it is work whose requester has already given up.

> **Storing excess demand is not the same as handling it**  
> A queue converts a rate mismatch into a growing backlog, not into extra capacity. The only stable responses are to increase throughput or reduce arrivals, and where the producer can be influenced, reducing arrivals is available immediately. Backpressure is the mechanism that carries that information upstream, so the mismatch is resolved at the source rather than stored in the middle.

**Mental model**

A feedback signal travelling opposite to the data flow. The consumer communicates its capacity, and the producer's rate is bounded by what has been granted.

1. **Observe** — The consumer tracks its own saturation — queue depth, latency, or outstanding work.
2. **Signal** — That state is communicated upstream, explicitly or by blocking.
3. **Slow** — The producer reduces its rate or stops.
4. **Stabilise** — Arrivals match capacity and the queue stops growing.
5. **Resume** — Capacity is granted again as the consumer catches up.

> **Backpressure propagates until it reaches something that cannot be slowed**  
> Slowing a consumer slows its producer, which slows that producer's producer, all the way to the origin. If the origin is an internal job it simply waits, which is the desired outcome; if it is a user clicking a button, the pressure surfaces as a slow application, and if it is an external system that cannot be slowed, the chain ends in rejection. Knowing where the chain terminates is what decides whether backpressure is the right mechanism at all.

**How it works**

**Mechanisms, from explicit to implicit**

```text
EXPLICIT CREDIT  (strongest)
  consumer grants the producer N items of credit
  producer sends at most N, waits for more
  -> arrivals can never exceed granted capacity
  -> used by TCP windows, HTTP/2 flow control,
     reactive streams

BLOCKING
  producer calls a bounded queue that blocks when
  full
  -> simple, effective in-process
  -> the producer's thread is held, which may be
     unacceptable

REJECTION AS A SIGNAL
  return 429 or 503 with a retry hint
  -> works across service boundaries where blocking
     does not
  -> depends on clients honouring it

PULL INSTEAD OF PUSH
  the consumer fetches when ready, rather than being
  pushed to
  -> backpressure is inherent; nothing arrives
     unbidden
  -> message queue consumers work this way

LAG AS A SIGNAL
  consumer lag rising -> reduce production
  -> indirect but works across systems that share
     nothing else

THE BEST OPTION IS USUALLY PULL
  it requires no protocol, no signalling and no
  cooperation: the consumer simply does not ask for
  more than it can handle
```

1. **Prefer pull-based consumption** — It provides backpressure structurally, with nothing to implement or honour.
2. **Bound every buffer** — An unbounded queue is backpressure that was designed out of the system.
3. **Signal before saturation, not at it** — By the time the buffer is full, latency is already unacceptable.
4. **Know where the chain terminates** — Pressure reaching a user is a slow application; reaching an uncontrollable source means rejection.
5. **Never block a request thread on a backpressured path** — Held threads convert one slow dependency into total exhaustion.
6. **Shed load where backpressure cannot reach** — External producers must be refused rather than slowed.

**Why buffering is not a solution**

```text
PRODUCER 1,000/s   CONSUMER 800/s

WITH A LARGE BUFFER
  surplus 200/s accumulates
  after 60 s   12,000 queued -> 15 s of latency
  after 300 s  60,000 queued -> 75 s of latency
  -> every request has timed out long before it is
     processed
  -> the system is doing work for nobody
  -> eventually memory is exhausted and everything
     buffered is lost

WITH BACKPRESSURE
  buffer reaches its bound
  producer is slowed to 800/s
  -> latency stays at the buffer's depth, constant
  -> throughput is 800/s either way
  -> but now the surplus is visible at the source,
     where it can be handled deliberately

THE POINT
  buffering NEVER increases throughput
  it only decides where the surplus waits and how
  long each item sits before being served

SURPLUS MUST GO SOMEWHERE
  slow the producer     (backpressure)
  reject the excess     (load shedding)
  add capacity          (scaling)
  -> there is no fourth option; storing it is just
     deferring the choice
```

| Metric | Value | Note |
|---|---|---|
| Buffering | adds latency | not throughput |
| Backpressure | bounds latency | **at the source** |
| Best form | pull-based | structural |
| Terminates at | user or rejection | know which |

> **Blocking a request-handling thread to apply backpressure exhausts the pool**  
> Backpressure implemented by making the caller wait works within a process and fails badly across a service boundary, because the waiting caller is holding a thread that other requests need. One slow consumer then consumes the entire thread pool of everything upstream of it, which is how a local rate mismatch becomes a fleet-wide outage. Across service boundaries the signal has to be a response, not a delay.

**Technologies**

| Mechanism | Where | Note |
|---|---|---|
| TCP flow control | Network transport | The original credit-based scheme |
| HTTP/2 flow control | Per-stream windows | Backpressure inside a connection |
| Reactive streams | In-process pipelines | Explicit demand signalling |
| Bounded queues | Within a process | Blocking provides the signal |
| Message consumers | Pull-based by design | The consumer sets the pace |
| 429 and 503 responses | Across service boundaries | Requires clients to honour them |

Pull-based message consumption is worth emphasising as the practical default: a consumer that fetches only what it can process needs no protocol, no cooperation and no signalling, because it never receives work it did not ask for.

**Trade-offs**

**Responses to a rate mismatch**

| Response | Latency | Work lost | Requires |
|---|---|---|---|
| Unbounded buffering | Grows without limit | Everything, on crash | Nothing |
| Bounded buffer, then reject | Bounded | The excess | A bound |
| Backpressure | Bounded | None | A controllable producer |
| Load shedding | Bounded | The shed requests | A rejection policy |
| Scale the consumer | Bounded | None | Capacity and time |

Backpressure and load shedding are the two coherent answers and they apply in different circumstances: slow the producer where you can reach it, refuse the work where you cannot. Systems generally need both, because some producers are internal and some are the open internet.

> **Ask before choosing it**  
> Can the producer actually be slowed, and what happens when it is? Backpressure applied to an internal pipeline is ideal; applied to user-facing traffic it manifests as an unresponsive product, and applied to an external system that ignores the signal it does nothing at all.

**How it fails**

**How backpressure goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Memory exhausted | Unbounded buffer with no signal | Bound every queue |
| Thread pools exhausted upstream | Blocking across a service boundary | Respond with rejection instead of waiting |
| Pressure reaches users as unresponsiveness | Chain terminates at the request path | Shed load at the edge instead |
| Signal ignored | External clients not honouring 429 | Enforce with rate limits, not only signals |
| Oscillation between full speed and stopped | On/off signalling with no gradation | Gradual rate adjustment with hysteresis |
| Latency already unacceptable when the signal fires | Threshold set at full saturation | Signal earlier, at partial saturation |
| Work processed after clients gave up | No expiry on queued items | Drop entries older than the client timeout |

> **An unbounded queue is a decision to fail later and worse**  
> Every buffer without a limit is a promise that the surplus will be stored indefinitely, which ends in memory exhaustion and the loss of everything accumulated. Before that, latency has already grown past every client's timeout, so the system spends its final minutes doing work whose results nobody will receive. Bounding the queue converts that into a visible, survivable rejection at the moment the mismatch begins.

**Where it is used**

- **TCP**, whose receive window is credit-based backpressure and the model everything else borrows from.
- **Stream processing frameworks**, propagating demand upstream through the topology.
- **Message queue consumers**, which pull at their own pace and therefore have backpressure inherently.
- **Reactive libraries**, where explicit demand signalling is the core abstraction.
- **Ingestion pipelines**, slowing collection when downstream storage cannot keep up.

**In the interview**

The sentence worth saying is that buffering never adds throughput; everything else follows from it.

- **State that surplus demand must be slowed, rejected or absorbed by capacity** — storing it only defers the choice.
- **Do the arithmetic** of a 200-per-second mismatch to show how quickly latency exceeds every timeout.
- **Prefer pull-based consumption**, since it provides backpressure with nothing to implement or honour.
- **Warn against blocking across service boundaries**, which converts one slow consumer into upstream thread exhaustion.
- **Trace where the chain terminates** and say what happens there, because that determines whether backpressure or shedding is appropriate.

**Practice drill**

A pipeline stage produces 1,000 items per second and the next stage processes 800. Compute the queue depth and added latency after one minute and after five, assuming an unbounded buffer. Then design backpressure for it: the mechanism, the threshold and the signal. Say what the producer does when slowed, and follow the chain upstream to identify where the pressure ultimately lands and what a user would observe.

**Go deeper**

Backpressure is a signal travelling against the flow of data, telling a producer to slow down when the consumer cannot keep up.

**It rests on the fact that buffering never creates capacity.** A queue between a fast producer and a slow consumer stores the difference, and the difference accumulates without limit until memory runs out. Throughput is identical with or without the buffer; all the buffer decides is how long each item waits and how much is lost when the process finally fails. Recognising this makes the response set small and clear: slow the producer, reject the excess, or add capacity.

**Latency degrades long before memory does.** A surplus of a couple of hundred items per second produces a queue deep enough to exceed every client timeout within a minute or two, so the system spends the remainder of the incident processing work nobody is waiting for any more. The failure is therefore not a sudden crash but a long period of expensive uselessness, which is why bounded buffers with early signalling are better than large ones.

**Pull-based consumption provides the mechanism structurally.** A consumer that fetches work when it is ready never receives more than it asked for, so no protocol, signal or cooperation is required — the absence of a request is the backpressure. Where a pipeline can be arranged this way, it is strictly better than any push-based design with a signalling layer bolted on, because there is nothing to implement incorrectly and nothing for a producer to ignore.

**Across service boundaries the signal must be a response, not a delay.** Applying backpressure by making the caller wait works inside a process and is destructive between services, because the waiting caller holds a thread that other work needs. One saturated consumer then drains the thread pools of everything upstream, turning a local rate mismatch into a cascading outage — which is why rejection with a retry hint, rather than a slow response, is the correct cross-service form.

**Pressure propagates until it reaches something that cannot be slowed, and that endpoint determines the design.** An internal batch job simply waits, which is exactly right. A user pressing a button experiences an unresponsive product, which is usually worse than a clear refusal. An external system that ignores the signal absorbs nothing at all. Tracing the chain to its origin before choosing backpressure is what reveals whether it will work or whether load shedding is the mechanism actually needed.

**Signalling should begin before saturation and change gradually.** A threshold set at a full buffer fires when latency is already unacceptable, and an on/off signal produces oscillation between full speed and a complete stop. Signalling at partial saturation, and adjusting the rate rather than toggling it, keeps the system near its sustainable throughput instead of swinging around it — which is the same reason congestion control algorithms adjust continuously rather than in steps.

**Related patterns:** Load Shedding · Leaky Bucket · Work Queue · Bulkhead

---

### Load Shedding

*Deliberately reject a portion of traffic when overloaded, so the remainder is served properly instead of everything failing together.*

> **When you hear…** Demand exceeding capacity with no way to slow the source · degradation affecting all requests · protecting a service from collapse

**Flow:** `Load exceeds capacity` → `Classify requests` → `Reject low priority` → `Serve the rest well` → `Recover`

**The problem**

Demand reaches twice what a service can handle. Without intervention every request is accepted, each one takes twice as long, queues grow, timeouts fire, and clients retry — which adds more load. The system ends up serving almost nothing while consuming all of its capacity.

Attempting to serve everyone is what produces the total failure. Capacity is finite, and admitting work beyond it does not create more; it simply degrades every request equally until none of them completes usefully.

> **Refusing some work is how the rest gets served**  
> If a service admits only what it can handle and rejects the remainder immediately, the admitted requests receive normal service and the rejected ones receive a fast, honest error they can act on. Total useful throughput is higher than in the case where everything is accepted and everything times out — deliberate partial failure is strictly better than accidental total failure.

**Mental model**

An admission decision at the entrance. When a saturation signal exceeds a threshold, a portion of incoming work is refused before any resources are committed to it.

1. **Measure** — Track a signal that reflects real saturation — latency, queue depth, concurrency.
2. **Decide** — Determine how much must be rejected to stay within capacity.
3. **Classify** — Rank requests by importance so the right ones are dropped.
4. **Reject** — Refuse immediately, before doing any work.
5. **Recover** — Admit more as the signal improves, gradually.

> **Rejecting must be far cheaper than serving, or shedding cannot help**  
> If a refusal costs a database lookup to check a quota, a policy evaluation and a log write, then at twice capacity the rejections alone can consume everything the service has. The decision must happen at the edge, on information already available in memory, and cost orders of magnitude less than processing — otherwise the protective mechanism becomes part of the overload.

**How it works**

**Choosing the signal and what to drop**

```text
SIGNALS, WORST TO BEST
  CPU utilisation     lags; already saturated when
                      it looks high
  request rate        says nothing about cost per
                      request
  queue depth         better; shows accumulation
  concurrency         good; work in flight now
  LATENCY vs target   best; directly measures whether
                      the service is meeting its
                      obligations

PRIORITY CLASSES
  1  payments, checkout       never shed
  2  logged-in interactive    shed last
  3  anonymous browsing       shed next
  4  prefetch, analytics,
     background refresh       shed first
  -> at 2x load, dropping classes 3 and 4 may fully
     restore classes 1 and 2

HOW MUCH TO SHED
  target: keep latency at the objective
  if p99 > target: increase the shed fraction
  if p99 < target: decrease it
  -> a control loop, not a fixed threshold

COST OF REJECTION
  must be microseconds
  -> in-memory decision at the edge
  -> no database, no downstream call, no per-request
     policy fetch
```

1. **Shed at the edge, before resources are committed** — Rejecting after a service has begun work wastes the capacity shedding was meant to protect.
2. **Use latency against the objective as the signal** — Utilisation lags; latency states directly whether obligations are being met.
3. **Classify requests by importance in advance** — Undifferentiated shedding drops checkout as readily as a prefetch.
4. **Adjust the shed fraction continuously** — A fixed threshold either over-rejects or under-protects as load moves.
5. **Return a retry hint with the rejection** — Rejections that trigger immediate retries add to the load they relieve.
6. **Recover gradually** — Admitting everything the moment the signal improves re-triggers the overload.

**What shedding buys**

```text
CAPACITY 1,000/s     DEMAND 2,000/s

NO SHEDDING
  all 2,000 accepted
  each takes ~2x as long, queues grow
  clients time out at 1 s and retry
  retries add load -> effective demand 3,000/s
  -> useful throughput approaches ZERO
  -> everyone fails

WITH SHEDDING
  admit 1,000/s, reject 1,000/s immediately
  admitted: normal latency, all succeed
  rejected: fast 503 with Retry-After
  -> useful throughput 1,000/s
  -> 50% of users served perfectly, 50% told to wait

WITH PRIORITY SHEDDING
  classes 1 and 2 total 900/s -> all admitted
  classes 3 and 4 total 1,100/s -> 900 rejected
  -> every checkout succeeds
  -> analytics and prefetch absorb the loss
  -> the business impact is far smaller than 50%

THE RETRY AMPLIFIER
  rejections without a retry hint produce immediate
  retries
  -> shedding 1,000/s creates 1,000/s of retries
  -> always return Retry-After, and honour it
```

| Metric | Value | Note |
|---|---|---|
| No shedding | ≈0 useful | **everyone fails** |
| Flat shedding | 50% served | randomly chosen |
| Priority shedding | all critical served | loss absorbed elsewhere |
| Rejection cost | microseconds | or it fails too |

> **Rejections without a retry hint manufacture their own load**  
> A client refused with no guidance retries immediately, so shedding a thousand requests per second creates a thousand retries per second — the mechanism relieving the overload is simultaneously feeding it. Returning a concrete wait time, and having clients honour it with jitter, is what makes the rejection actually reduce demand rather than merely redistributing it in time.

**Technologies**

| Layer | Mechanism | Note |
|---|---|---|
| Load balancers | Queue limits and rejection | Cheapest place; never reaches the application |
| Service meshes | Adaptive concurrency limits | Applied without application changes |
| Application admission control | Priority-aware shedding | The only layer that knows request importance |
| CDN and edge | Reject before origin | Stops traffic furthest from the backend |
| Rate limiters | Per-client bounds | Prevents one client causing the overload |
| Backpressure | Slow the producer | Better where the producer is controllable |

Adaptive concurrency limiting is the most practical modern form: rather than configuring a fixed threshold, the system continuously probes for the concurrency at which latency begins to degrade and admits up to that level, which tracks real capacity as it changes with deploys and dependencies.

**Trade-offs**

**Overload responses**

| Response | Useful throughput | Who suffers | Complexity |
|---|---|---|---|
| Accept everything | Approaches zero | Everyone | None |
| Random shedding | Near capacity | A random half | Low |
| Priority shedding | Near capacity | Low-value traffic | Classification needed |
| Backpressure | Near capacity | The producer waits | Controllable producer |
| Autoscaling | Eventually higher | Everyone, during the delay | Provisioning time |
| Queue everything | Approaches zero | Everyone, slowly | Low |

Autoscaling belongs in the table but not as a substitute: new capacity takes minutes to arrive and a spike does damage in seconds, so shedding is what keeps the service alive during the interval in which scaling is still happening.

> **Ask before choosing it**  
> Which requests can be dropped with the least harm? If every request is equally important the mechanism degrades to random rejection, which is still far better than total failure but leaves most of its value unclaimed — and the classification is usually a product conversation rather than a technical one.

**How it fails**

**How load shedding goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Shedding consumes the remaining capacity | Expensive rejection path | Decide in memory at the edge |
| Critical requests dropped | No prioritisation | Classify traffic in advance |
| Rejections cause more load | No retry hint, clients retry instantly | Return Retry-After; require jitter |
| Overload not detected until collapse | Utilisation used as the signal | Use latency against the objective |
| Oscillation between shedding and overload | Immediate full recovery | Ramp admission gradually |
| Wasted work | Requests rejected after processing started | Shed at admission, not mid-request |
| Retry storms after recovery | All clients returning at once | Jittered retry advice |

> **Accepting everything during overload serves nobody at all**  
> Admitting twice the capacity does not halve the service quality — it collapses it, because queues grow, every request exceeds its timeout, clients retry, and the added load compounds until useful throughput approaches zero while the machines run at full utilisation. Deliberately refusing half the traffic feels worse and produces a dramatically better outcome, which is why shedding has to be designed in before the incident rather than argued about during it.

**Where it is used**

- **Large-scale web services**, which shed at the edge as standard practice during traffic spikes.
- **Service meshes with adaptive concurrency limits**, discovering the admission threshold automatically.
- **Payment and checkout paths**, protected by shedding everything of lower priority first.
- **Search and recommendation backends**, dropping optional enrichment when saturated.
- **Electrical grids**, the origin of the term and the same reasoning applied to a physical network.

**In the interview**

Make the throughput argument explicitly; it converts an uncomfortable idea into an obviously correct one.

- **Show that accepting everything yields near-zero useful throughput**, with the retry amplification included.
- **Choose latency against the objective as the signal** and explain why utilisation is too slow to be useful.
- **Introduce priority classes** with concrete examples, since that is where most of the value comes from.
- **Insist the rejection path is cheap**, because an expensive refusal at twice capacity is part of the overload.
- **Return a retry hint and recover gradually**, otherwise the mechanism generates the load it is relieving.

**Practice drill**

A service handles 1,000 requests per second and receives 2,000. Compute what happens with no shedding, including the effect of client retries after a one-second timeout. Then design shedding: the signal, the threshold, the priority classes and the rejection response. State the useful throughput your design achieves and who is refused. Finally, describe the recovery behaviour and why admitting everything as soon as latency improves would be wrong.

**Go deeper**

Load shedding rejects a portion of incoming work during overload so that the remainder receives proper service.

**Its justification is arithmetic rather than philosophical.** Admitting more work than capacity does not distribute the shortfall evenly; it collapses throughput, because queues grow past every timeout, clients retry, and the retries add load. Refusing the excess immediately keeps admitted requests within their latency objective and produces far more completed work than the alternative — deliberate partial failure genuinely beats accidental total failure.

**The choice of signal decides whether shedding engages in time.** Utilisation lags badly: by the time it looks high the queues are already deep, and it says nothing about whether the service is meeting its obligations. Latency measured against the objective states directly whether the system is keeping its promises, and concurrency or queue depth are reasonable proxies. Feeding that signal into a continuous control loop, rather than a fixed threshold, keeps admission tracking real capacity as it changes.

**Prioritisation is where most of the value lies.** Random shedding at twice capacity serves half of everyone; priority shedding can serve every checkout and every logged-in session while dropping prefetches, analytics and background refreshes that nobody will notice. The classification is a product decision about what matters, and doing it in advance is what converts a blunt survival mechanism into one whose business impact is small.

**The rejection path must be dramatically cheaper than the serving path.** A refusal requiring a database lookup, a policy fetch or a downstream call costs a meaningful fraction of a real request, and at twice capacity the rejections alone can saturate the service. Deciding in memory, at the edge, before any resources are committed, is what makes the mechanism protective — and shedding after work has already begun wastes precisely the capacity it was protecting.

**Rejections without guidance recreate the load they relieve.** A client refused with no retry information returns immediately, so shed traffic reappears within milliseconds and the system is refusing the same requests repeatedly. Returning a concrete wait time, and clients honouring it with jitter, is what turns a rejection into an actual reduction in demand rather than a redistribution of it across the next second.

**Recovery has to be gradual or the system oscillates.** Admitting everything the instant latency improves sends the full suppressed demand back at once, which re-triggers the overload and produces a cycle of collapse and recovery. Ramping admission upward while watching the signal keeps the system near its sustainable point, and it is the same reasoning that makes congestion control increase slowly and back off sharply — the asymmetry exists because overload is much more expensive than under-utilisation.

**Related patterns:** Backpressure · Circuit Breaker · Token Bucket · Bulkhead

---

## Service Architecture

### API Gateway

*Put one entry point in front of many services to handle authentication, routing, limiting and observability once instead of everywhere.*

> **When you hear…** Many services exposed directly to clients · cross-cutting concerns duplicated in each · clients coupled to internal topology

**Flow:** `Clients call one host` → `Gateway authenticates` → `Applies limits` → `Routes to services` → `Services stay simple`

**The problem**

Forty services are exposed directly to clients. Each implements its own authentication, rate limiting, request logging and CORS handling, and each implements them slightly differently — so a security fix must be applied forty times and verified forty times.

Clients are also bound to the internal structure. Splitting a service, renaming it or moving an endpoint requires every client, including mobile applications that take weeks to update, to change with it.

> **Concerns that are identical everywhere belong in one place**  
> Authentication, rate limiting, request logging and TLS termination do not vary by service; only their configuration does. Handling them once at a single entry point removes forty implementations and forty opportunities to get it wrong, and simultaneously decouples clients from internal topology — because the address they call is the gateway rather than whichever service currently answers.

**Mental model**

A reverse proxy with policy. Every external request passes through it, is validated and shaped, and is then forwarded to whichever internal service handles it.

1. **Terminate** — TLS ends here; internal traffic is simpler.
2. **Authenticate** — Identity is established once and passed onward as a trusted claim.
3. **Enforce** — Rate limits, quotas and request validation are applied before any service is involved.
4. **Route** — The request is directed to the appropriate service.
5. **Observe** — Every request is logged and traced from one consistent place.

> **The gateway is in the path of every request, so its failure is total**  
> A component every request must traverse is by definition a single point of failure, and a deployment mistake, a bad configuration or a memory leak there takes down services that are themselves perfectly healthy. It must be redundant, deployed with more care than the services behind it, and kept simple enough that changes to it are low risk.

**How it works**

**What belongs in the gateway and what does not**

```text
BELONGS
  TLS termination
  authentication (verify the token, reject invalid)
  rate limiting and quotas
  request and response logging, tracing headers
  routing by path or host
  request size limits, timeouts
  CORS, compression
  -> identical for every service, configuration only

DOES NOT BELONG
  business logic of any kind
  response transformation specific to one service
  authorisation decisions needing domain knowledge
  data aggregation across services
  -> these make the gateway a shared service that
     every team must change, and it becomes the
     bottleneck for all delivery

THE TEST
  if adding a feature to a service requires changing
  the gateway, the boundary is wrong

AUTHENTICATION VERSUS AUTHORISATION
  gateway: is this a valid token? who is it?
  service: may this user perform this action on this
           resource?
  -> the second needs domain knowledge the gateway
     must not have
```

1. **Keep cross-cutting concerns only** — Business logic in the gateway turns it into a shared bottleneck for every team.
2. **Authenticate at the gateway, authorise in the service** — Identity is generic; permission decisions need domain context.
3. **Run it redundantly across zones** — Every request passes through it, so its availability is the system's availability.
4. **Pass a trusted identity downstream** — Services should not re-verify tokens on every internal call.
5. **Keep configuration declarative and reviewed** — Most gateway outages are caused by configuration, not code.
6. **Do not let it become a deployment bottleneck** — If every team must queue for gateway changes, it has absorbed too much.

**Operational shape**

```text
CAPACITY
  it handles 100% of traffic
  -> size it for peak, with headroom
  -> a single instance is never acceptable

LATENCY
  adds one network hop plus policy evaluation
  typical 1-5 ms
  -> negligible against most backend work
  -> becomes significant only for very fast endpoints

FAILURE MODES
  configuration error       most common by far
  certificate expiry        total outage, entirely
                            preventable
  connection pool exhausted downstream slowness
                            consuming gateway
                            resources
  -> bulkhead per upstream so one slow service does
     not consume the gateway

TRUST BOUNDARY
  internal services must not accept external traffic
  directly, or the gateway can be bypassed and every
  policy with it
  -> network policy, not just convention

WHAT IT IS NOT
  a service mesh: that handles service-to-service
  traffic; the gateway handles north-south traffic
  from outside
```

| Metric | Value | Note |
|---|---|---|
| Scope | cross-cutting | **not business logic** |
| Availability | must be redundant | total failure otherwise |
| Latency | 1-5 ms | usually negligible |
| Common outage | configuration | not code |

> **A gateway that accumulates business logic becomes every team's bottleneck**  
> Each small transformation added for one service is reasonable in isolation, and the accumulation turns the gateway into a shared codebase that every team must modify and every deployment must coordinate. Changes to it become high-risk because it is in the path of everything, so they queue and slow down — and the component introduced to decouple teams becomes the thing that couples them.

**Technologies**

| Option | Strength | Watch for |
|---|---|---|
| Managed cloud gateways | Operated, integrated, scalable | Provider-specific features and limits |
| Kong, Envoy, Traefik | Flexible, plugin ecosystems | You operate and scale them |
| NGINX | Ubiquitous, fast, simple | Policy features are more limited |
| Service mesh ingress | Consistent with internal routing | Couples ingress to mesh adoption |
| Backend for frontend | Per-client aggregation | A complement, not a replacement |
| No gateway | One less hop | Cross-cutting concerns duplicated everywhere |

The distinction between gateway and mesh is worth keeping clear: the gateway governs traffic entering from outside, while the mesh governs traffic between internal services. They overlap in mechanism and differ in purpose, and conflating them produces designs where internal calls traverse an external entry point unnecessarily.

**Trade-offs**

**Handling cross-cutting concerns**

| Approach | Duplication | Coupling | Failure impact |
|---|---|---|---|
| Each service implements its own | High | None | Isolated |
| Shared library in each service | Low | Version coupling | Isolated |
| API gateway | None | Clients to the gateway | Total if it fails |
| Service mesh sidecars | None | Platform coupling | Per-instance |
| Gateway plus mesh | None | Both | Split by direction |

Shared libraries are a reasonable middle ground for smaller systems and have one persistent weakness: upgrading them requires every service to redeploy, so a security fix propagates at the speed of the slowest team rather than at the speed of a configuration change.

> **Ask before choosing it**  
> How many services are exposed and how much genuinely varies between them? For three services with similar needs, a shared library is simpler and has no single point of failure; the gateway earns its place when the number of services and the number of policies both grow.

**How it fails**

**How API gateways go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Total outage from a configuration change | Untested routing or policy update | Staged rollout with automated validation |
| Everything down on a certificate expiry | Manual renewal | Automated rotation with alerting |
| Gateway becomes a delivery bottleneck | Business logic accumulating in it | Keep it to cross-cutting concerns |
| One slow service degrades all traffic | Shared connection pools | Bulkhead per upstream |
| Policies bypassed | Services reachable directly | Enforce with network policy |
| Latency on fast endpoints | Heavy policy evaluation per request | Simplify the hot path; cache decisions |
| Identity re-verified everywhere | Downstream services not trusting the gateway | Signed internal identity headers |

> **Services reachable without passing through the gateway make every policy optional**  
> If internal services accept traffic directly, then authentication, rate limiting and logging apply only to callers who happen to use the front door. An attacker who finds an internal address bypasses all of it, and so does any internal caller taking a shortcut. The gateway's guarantees are only as strong as the network policy that forces traffic through it — convention is not enforcement.

**Where it is used**

- **Public API platforms**, where quotas, keys and versioning are enforced centrally.
- **Microservice architectures**, presenting one coherent surface over many internal services.
- **Mobile backends**, where clients cannot be updated quickly and must be decoupled from internal changes.
- **Partner and third-party integrations**, needing distinct authentication and quota policies.
- **Migration and strangler patterns**, routing individual paths from a legacy system to new services.

**In the interview**

Define the boundary precisely; the interesting content is what the gateway must not do.

- **List the cross-cutting concerns** and say they are identical everywhere, which is the argument for centralising them.
- **Draw the authentication/authorisation line** — identity at the gateway, permission in the service — and explain why.
- **Name it as a single point of failure** and describe redundancy and staged configuration rollout as the mitigations.
- **Warn against business logic accumulating**, since that converts it into a shared bottleneck for every team.
- **Require network enforcement** so services cannot be reached directly and the policies bypassed.

**Practice drill**

Forty services are exposed directly to web and mobile clients, each implementing its own authentication and rate limiting. Design the gateway: what it handles, what stays in the services, and where the authorisation boundary sits. State how you make it highly available and how a configuration change is rolled out safely. Then identify two requests that would tempt you to add service-specific logic to it, and say what you would do instead.

**Go deeper**

An API gateway is a single entry point that applies cross-cutting concerns once for many services and decouples clients from internal structure.

**Its value comes from the concerns being genuinely identical.** Token verification, rate limiting, request logging and TLS termination do not vary by domain — only their configuration does — so implementing them once removes both the duplication and the divergence that accumulates when many teams implement the same thing separately. A security fix becomes one change rather than forty, applied at the speed of a deployment rather than the speed of the slowest team.

**The decoupling it provides is as valuable as the deduplication.** Clients address the gateway rather than individual services, so services can be split, merged, renamed or moved without any client changing. That matters most for clients that cannot be updated on demand — mobile applications in particular — where internal refactoring would otherwise be constrained by release cycles outside the team's control.

**Drawing the authorisation line correctly is what keeps it maintainable.** Establishing who the caller is requires no domain knowledge and belongs at the entrance; deciding whether that caller may perform a specific action on a specific resource requires knowing what those things mean. Pushing the second into the gateway forces domain logic into shared infrastructure, which is the first step towards the component becoming everyone's bottleneck.

**Accumulated business logic is the characteristic decay of this pattern.** Each small service-specific transformation is defensible individually, and together they turn the gateway into a codebase every team must change and every deployment must coordinate around. Because it sits in the path of all traffic, those changes are high-risk and slow — so the component adopted to decouple teams ends up serialising their delivery, and the test for whether a change belongs is whether a service feature should require touching it at all.

**It is a single point of failure, which is acceptable only if treated as one.** Redundancy across zones, staged configuration rollout with automated validation, automated certificate rotation and per-upstream bulkheads are not refinements but the minimum for a component whose failure is total. In practice the overwhelming majority of gateway outages come from configuration rather than code, which means the review and rollout process for configuration deserves the same rigour as for software.

**Its guarantees depend on traffic having no alternative route.** If internal services accept direct connections, every policy the gateway enforces becomes optional for anyone who knows an internal address, including internal callers taking shortcuts. Network policy that makes the gateway the only reachable path is what turns its rules into actual constraints, and a system relying on convention has documentation rather than enforcement.

**Related patterns:** Backend For Frontend · Service Discovery · Sidecar · Token Bucket

---

### Backend For Frontend

*Give each client type its own backend that shapes data specifically for it, so no single API has to compromise between very different consumers.*

> **When you hear…** One API serving clients with conflicting needs · mobile making many round trips · over-fetching on constrained connections

**Flow:** `Mobile BFF` → `Web BFF` → `Each shapes data` → `Same services behind` → `Clients get exact payloads`

**The problem**

A mobile application needs a compact payload containing exactly eight fields, assembled from four services. A web application on the same endpoint wants forty fields and does not mind the size. One shared API either sends everything, wasting bandwidth on the connection least able to afford it, or sends the minimum and forces the web client to make several more calls.

Mobile suffers most from the compromise. Each additional round trip on a mobile network costs hundreds of milliseconds, so an API shaped for a browser turns one screen into six sequential requests.

> **Let the client's constraints shape its own backend**  
> The conflict exists because one interface is serving consumers with genuinely different requirements — payload size, round-trip cost, update cadence, and what the screen actually displays. Giving each client type a backend owned by the team that builds that client removes the negotiation entirely: each API can be exactly what its consumer needs, because it has only one consumer.

**Mental model**

A thin, client-specific layer that calls the same downstream services and assembles responses tailored to one client's screens.

1. **Own** — The client team owns its backend and can change it freely.
2. **Call** — It fans out to the same shared services everyone uses.
3. **Shape** — Responses match what the client actually renders.
4. **Aggregate** — Several service calls collapse into one client request.
5. **Evolve** — It changes at the pace of that client, not of every client.

> **Business logic duplicated across backends will drift**  
> Once there are three of these, a pricing rule or an eligibility check implemented in each will eventually be implemented differently, and the clients will disagree about what a user is allowed to do. These layers must aggregate and shape, never decide — any rule that matters belongs in a shared service that all of them call.

**How it works**

**What it does and what it must not**

```text
WITHOUT  (one shared API)
  mobile home screen
    GET /user         120 fields, needs 3
    GET /orders       full objects, needs id + status
    GET /recommended  needs 5 items, gets 50
    GET /notifications
  -> 4 sequential round trips, ~800 ms on mobile
  -> payload ~180 KB, of which ~8 KB is used

WITH A MOBILE BFF
  GET /mobile/home
    -> BFF calls 4 services in parallel
    -> returns exactly what the screen renders
  -> 1 round trip, ~200 ms
  -> payload ~8 KB

RESPONSIBILITIES
  aggregate calls        yes
  reshape and rename     yes
  filter fields          yes
  client-specific
  caching and defaults   yes
  business rules         NO
  authorisation logic    NO
  data ownership         NO

OWNERSHIP
  the mobile team owns the mobile backend
  -> they change it with their release
  -> no cross-team coordination for a screen change
  -> this is the main organisational benefit
```

1. **Give each client type its own backend** — Web, mobile and partner integrations have genuinely different constraints.
2. **Let the client team own it** — The coordination saving is most of the value.
3. **Keep business rules in shared services** — Duplicated logic across backends will diverge.
4. **Call downstream services in parallel** — Aggregation only helps if the fan-out is concurrent.
5. **Keep it thin enough to be disposable** — A backend that accumulates state stops being a shaping layer.
6. **Consider GraphQL as an alternative** — One flexible endpoint can serve the same purpose with one deployment.

**Cost, and when GraphQL is the better answer**

```text
COST OF THE PATTERN
  3 client types = 3 services to build, deploy,
  monitor and secure
  + shared aggregation code duplicated or extracted
  + one more hop in every request path
  -> justified when client needs genuinely differ
  -> not justified for cosmetic differences

GRAPHQL INSTEAD
  one endpoint, clients request the fields they need
  + no per-client backend
  + client-driven shaping without a deployment
  - query complexity and cost control become real
    problems
  - caching is harder than with fixed endpoints
  -> often the right answer when the difference is
     mostly about field selection

BFF IS BETTER WHEN
  clients differ in more than field selection
    different auth, different caching, different
    protocols, different update cadence
  teams want independent deployment
  payload shaping requires real logic

FAILURE ISOLATION BONUS
  a broken mobile backend does not affect web
  -> an underrated benefit of the split
```

| Metric | Value | Note |
|---|---|---|
| Round trips | 4 → 1 | **~600 ms saved** |
| Payload | 180 KB → 8 KB | on mobile |
| Cost | one service per client | plus a hop |
| Rule | shape, never decide | logic stays shared |

> **Aggregation is only a win if the downstream calls run concurrently**  
> A backend that calls four services one after another has replaced four client round trips with four server round trips plus one more hop, which on a fast internal network is still an improvement but a much smaller one than expected. The saving comes from issuing the calls in parallel and from the internal network being faster than the client's — and sequential implementation quietly discards most of it.

**Technologies**

| Approach | Strength | Watch for |
|---|---|---|
| Per-client backend services | Full control per client | More services to operate |
| GraphQL | Client-driven field selection | Query cost control and caching |
| API gateway aggregation | No new services | Business logic creeping into the gateway |
| Edge compute functions | Backends close to users | Cold starts and limited runtimes |
| Shared API with field selection | Simple | Compromises on payload shape and round trips |
| No separation | Least infrastructure | Every client gets the same compromise |

GraphQL and the per-client backend are not really competitors: several organisations run client-specific backends that each expose a tailored schema, using GraphQL as the shaping mechanism while keeping the ownership and isolation benefits of separate deployments.

**Trade-offs**

**Serving multiple client types**

| Approach | Payload fit | Team autonomy | Services to run |
|---|---|---|---|
| One shared API | Compromised | Low | One |
| Shared API with field selection | Better | Low | One |
| GraphQL | Excellent | Medium | One |
| Backend per client | Excellent | High | One per client |
| Gateway aggregation | Good | Low | One |

Team autonomy is the column that usually decides it. The technical shaping can be achieved several ways, but only separate deployments let the mobile team change their backend alongside their app release without coordinating with anyone.

> **Ask before choosing it**  
> Do the clients differ in more than which fields they need? If the difference is purely field selection, GraphQL or a field parameter achieves it with one service; the pattern earns its extra deployments when caching, protocols, authentication or release cadence differ as well.

**How it fails**

**How backends for frontends go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Clients disagree about business rules | Logic duplicated across backends | Rules live in shared services only |
| Aggregation saves little | Downstream calls made sequentially | Fan out concurrently |
| Backends drift into full services | Accumulating state and ownership | Keep them thin and disposable |
| More deployments than the benefit warrants | Split by cosmetic differences | Consolidate; consider GraphQL |
| One slow service breaks a screen | No timeouts or fallbacks in the aggregation | Per-call timeouts with partial responses |
| Duplicated aggregation code | No shared client libraries | Extract common access code, not logic |
| Security policies inconsistent | Each backend implementing its own | Authenticate at the gateway, uniformly |

> **Rules implemented in several backends will eventually disagree**  
> A discount calculation or an eligibility check written once per client starts identical and diverges with the first bug fix applied in one place. The result is a user who can complete an action on the web and not on mobile, which is confusing to report and difficult to diagnose because both implementations look correct in isolation. These layers must shape data and never decide anything.

**Where it is used**

- **Mobile-heavy consumer products**, where round trips and payload size dominate perceived performance.
- **Organisations with separate web and mobile teams**, where independent deployment is the main benefit.
- **Partner-facing APIs**, shaped and versioned differently from internal clients.
- **Streaming and media platforms**, where device classes have very different capabilities.
- **GraphQL deployments**, achieving the shaping benefit through a schema rather than separate services.

**In the interview**

Give the round-trip and payload numbers; they make the case far better than an abstract description.

- **Quantify the mobile cost** — four sequential round trips against one, and a payload mostly discarded.
- **Emphasise team ownership**, since independent deployment is usually the larger benefit in practice.
- **State the hard boundary**: shape and aggregate, never decide, because duplicated rules drift.
- **Insist the fan-out is parallel**, or the aggregation saves far less than claimed.
- **Offer GraphQL as the alternative** and say when it is preferable — when the difference is only field selection.

**Practice drill**

A mobile home screen currently makes four sequential calls returning 180 KB, of which 8 KB is displayed. Design the backend for it: the endpoint, the fan-out, and the response shape. Estimate the latency improvement and state what assumption it depends on. Then decide what happens when one of the four downstream services is slow, and explain why a pricing rule needed by both web and mobile must not live in this layer.

**Go deeper**

A backend for frontend is a client-specific layer that aggregates and shapes data for exactly one kind of consumer.

**It exists because a shared API must compromise between consumers with incompatible constraints.** A mobile client on a high-latency connection needs few round trips and small payloads; a web client can afford more of both and wants richer data. One interface serving both either over-fetches for mobile or under-serves the browser, and no amount of parameterisation removes the tension when the differences extend beyond field selection.

**The organisational benefit usually exceeds the technical one.** When the team building a client also owns its backend, a screen change ships with the client release and requires no negotiation with a platform team. That removes a coordination point from every feature, which compounds over time in a way that the latency improvement does not — and it is why the pattern persists in organisations where the raw payload savings could have been achieved another way.

**The boundary must be shaping, never deciding.** Business rules implemented separately in each backend begin identical and diverge at the first divergent fix, producing clients that disagree about what a user may do. Because each implementation looks correct on its own, the resulting inconsistency is genuinely difficult to diagnose. Keeping every rule in a shared service that all the backends call is what makes multiple layers safe to have.

**Aggregation only pays when the fan-out is concurrent.** Replacing four client round trips with four sequential server calls plus an extra hop captures very little of the available saving; issuing them in parallel and returning one assembled response is where the improvement comes from. The pattern is frequently described in terms of fewer requests, when the actual mechanism is parallelism plus a faster internal network.

**Partial failure has to be handled deliberately at this layer.** A screen assembled from four services will sometimes have one of them slow or unavailable, and the choice between failing the screen and returning it with a section missing is a product decision the shaping layer is well positioned to make. Per-call timeouts with defined fallbacks turn one degraded dependency into a slightly reduced screen rather than a blank one.

**GraphQL addresses the same problem differently, and the choice depends on where the differences lie.** Where clients want different fields from the same data, a flexible schema serves them all from one deployment and avoids the additional services. Where they differ in caching, protocols, authentication or release cadence, separate backends are the better fit — and the two combine happily, with each client's backend exposing a tailored schema while retaining independent ownership and failure isolation.

**Related patterns:** API Gateway · Aggregator · Scatter Gather · Service Discovery

---

### Service Discovery

*Let services find each other through a registry of currently healthy instances, so addresses can change without reconfiguring callers.*

> **When you hear…** Instances that come and go · autoscaling and rolling deploys · hardcoded addresses breaking on every change

**Flow:** `Instance registers` → `Health checked` → `Client queries registry` → `Gets healthy addresses` → `Unhealthy removed`

**The problem**

Service addresses are listed in configuration files. Every deploy replaces instances with new addresses, every autoscaling event adds or removes them, and every one of those changes requires updating and redeploying the services that call them.

Between the change and the reconfiguration, callers are sending traffic to addresses that no longer answer. With instances rotating several times a day, the window in which the configuration is accurate becomes shorter than the time it takes to update it.

> **Membership is the thing that changes; addresses should be looked up, not configured**  
> Instances know when they start and stop, and health checks know which of them are actually working. If that information is kept in one place that callers consult, the configuration stops needing to describe a constantly changing set — callers ask for whatever is healthy now, and the answer is correct at the moment they need it.

**Mental model**

A registry of service instances, kept current by registration and health checking, consulted by callers at connection time.

1. **Register** — An instance announces itself on startup, or the platform does it.
2. **Check** — Health checks continuously confirm it is serving.
3. **Discover** — A caller asks for the healthy instances of a service.
4. **Balance** — The caller, or a proxy, chooses among them.
5. **Deregister** — An instance leaves cleanly, or is removed when checks fail.

> **The registry can be wrong in both directions, and both are harmful**  
> An instance that has died but has not yet failed enough health checks continues receiving traffic, which fails; an instance that is healthy but briefly unreachable from the checker is removed, reducing capacity. The registry always describes the recent past rather than the present, so callers must handle connecting to an instance that is no longer there.

**How it works**

**Client-side and server-side discovery**

```text
CLIENT-SIDE
  caller queries the registry
  caller picks an instance and connects directly
  + no extra hop
  + the caller can balance intelligently (least
    loaded, zone-aware)
  - discovery logic in every service and language

SERVER-SIDE
  caller connects to a stable address (a load
  balancer or virtual IP)
  that component consults the registry
  + callers need no discovery logic at all
  - one more hop
  - the balancer is in the request path

SIDECAR / MESH
  a local proxy handles discovery and balancing
  caller connects to localhost
  + no discovery logic in the application
  + rich balancing and retries
  - a proxy per instance to run

DNS-BASED
  service name resolves to healthy instances
  + works with everything
  - TTL caching means stale answers
  - no load or health awareness beyond membership
  -> simple and widely used; the TTL is the weakness
```

1. **Prefer platform-provided discovery where one exists** — A container platform already tracks instance lifecycle accurately.
2. **Health check what the service actually does** — A process that is running but cannot reach its database is not healthy.
3. **Deregister on shutdown before stopping** — Otherwise traffic arrives at an instance that has already closed.
4. **Cache discovery results briefly** — Querying per request makes the registry a hot dependency.
5. **Handle connections to dead instances** — The registry is always slightly behind reality.
6. **Keep the registry highly available** — It is consulted by everything; its failure prevents all new connections.

**Health checks and graceful shutdown**

```text
SHALLOW CHECK
  GET /health -> 200 if the process is up
  -> misses a service that cannot reach its database
  -> instance stays registered while failing every
     request

DEEP CHECK
  verifies critical dependencies
  -> accurate, but a shared dependency failing marks
     EVERY instance unhealthy at once
  -> the whole service disappears from the registry
  -> and then nothing can serve, including the
     requests that did not need that dependency

THE BALANCE
  readiness: can this instance serve requests now?
             (used for routing)
  liveness:  should this instance be restarted?
             (used for supervision)
  -> keep dependency checks shallow enough that a
     shared outage does not deregister everything

GRACEFUL SHUTDOWN SEQUENCE
  1  deregister
  2  wait for propagation (caches, DNS TTL)
  3  stop accepting new connections
  4  finish in-flight requests
  5  exit
  -> skipping step 2 causes errors on every deploy,
     and it is the step most often skipped
```

| Metric | Value | Note |
|---|---|---|
| Registry | current membership | **always slightly stale** |
| Check type | readiness vs liveness | different purposes |
| Shutdown | deregister first | then drain |
| Caller | must handle failures | registry is not truth |

> **A deep health check can deregister an entire service at once**  
> If every instance verifies a shared database and that database becomes briefly unreachable, every instance fails its check simultaneously and the service vanishes from the registry. Recovery then requires instances to pass checks again before any traffic flows, extending a short dependency blip into a full outage — and requests that did not need that dependency fail alongside the ones that did.

**Technologies**

| Option | Model | Note |
|---|---|---|
| Kubernetes services and endpoints | Platform-managed | Discovery is built in; usually the right answer |
| Consul | Registry with health checking | Works outside container platforms |
| etcd or ZooKeeper | Generic coordination store | Registry is one use among several |
| DNS-based discovery | Names resolve to instances | Universal; TTL caching is the weakness |
| Service mesh | Sidecar-based discovery and balancing | Rich features, platform commitment |
| Cloud load balancers | Server-side with health checks | Simple, managed, one extra hop |

When the workload already runs on a container platform, its built-in discovery is almost always preferable to an additional registry: the platform knows the instance lifecycle authoritatively because it controls it, rather than inferring it from registration and probes.

**Trade-offs**

**Discovery approaches**

| Approach | Extra hop | Client complexity | Balancing quality |
|---|---|---|---|
| Static configuration | No | None | None |
| DNS | No | None | Membership only |
| Client-side discovery | No | High | Excellent |
| Server-side load balancer | Yes | None | Good |
| Sidecar proxy | Local only | None | Excellent |
| Platform-managed | Varies | None | Good |

The sidecar row is why service meshes gained ground: it delivers client-side discovery quality without putting discovery logic into every application and every language, at the cost of running a proxy alongside every instance.

> **Ask before choosing it**  
> Does the platform already provide this? Adding a separate registry to an environment whose orchestrator already tracks instances creates two sources of truth that will disagree during exactly the events — deploys and scaling — that discovery exists to handle.

**How it fails**

**How service discovery goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Traffic to dead instances | Deregistration slower than shutdown | Deregister first, then drain |
| Errors on every deploy | No propagation delay before stopping | Wait for caches and TTLs to expire |
| Entire service deregistered | Deep health checks on a shared dependency | Keep readiness checks shallow |
| Stale routing for minutes | Long DNS TTLs | Short TTLs, or a non-DNS mechanism |
| Registry becomes a bottleneck | Queried on every request | Cache results with a short lifetime |
| All new connections fail | Registry unavailable | Highly available registry; cached fallback |
| Two sources of truth disagree | Separate registry alongside a platform | Use one mechanism |
| Traffic sent across zones | Discovery ignoring topology | Zone-aware balancing |

> **Deregistering after shutdown begins produces errors on every single deploy**  
> If an instance stops accepting connections before the registry has propagated its removal, callers holding the old address connect and fail — and because this happens during every rolling deploy, it becomes a steady background of errors that teams learn to ignore. The correct order is to deregister, wait long enough for every cache and TTL to expire, and only then stop serving, which turns deploys into non-events.

**Where it is used**

- **Container orchestration platforms**, where discovery is provided as a primitive.
- **Service meshes**, handling discovery and balancing through per-instance proxies.
- **Microservice estates on virtual machines**, using a dedicated registry with health checking.
- **Autoscaling groups behind load balancers**, the server-side form most systems start with.
- **Multi-region deployments**, where zone-aware discovery keeps traffic local.

**In the interview**

The health check and shutdown details are what separate a real answer from a description of a registry.

- **Contrast client-side and server-side discovery** with their respective costs, and mention the sidecar as the modern resolution.
- **Distinguish readiness from liveness**, and warn that deep dependency checks can deregister a whole service at once.
- **Give the graceful shutdown sequence** and stress the propagation wait, which is the step usually missed.
- **Say the registry is always slightly stale**, so callers must handle connecting to instances that have gone.
- **Use the platform's mechanism where one exists**, rather than adding a second source of truth.

**Practice drill**

A service scales between 5 and 50 instances and deploys several times a day. Design discovery for it: the registration mechanism, the health check, and how callers find instances. Write the exact shutdown sequence with timings, and explain which step prevents errors during rolling deploys. Then describe what happens if the health check verifies the database and the database has a 30-second outage.

**Go deeper**

Service discovery maintains a current view of which instances of a service are healthy, so callers can find them without configuration.

**It exists because instance membership changes faster than configuration can be updated.** Deploys, autoscaling and failures rotate instances continuously, and any static list is wrong within hours of being written. Moving membership into a registry that instances update and health checks verify means callers ask a question whose answer is current, rather than reading an answer that was current when someone last edited a file.

**The registry always describes the recent past.** Health checks take time to notice a failure and propagation takes time to reach callers, so there is always a window in which an instance is listed and not serving, or serving and not listed. Callers therefore cannot treat discovery as authoritative — they must handle connection failures and retry elsewhere — and designs that assume the registry is correct fail during precisely the events it exists to manage.

**Health check depth is a genuine trade rather than a detail.** A shallow check misses an instance that is running but unable to do useful work, so traffic continues to a failing instance; a deep check that verifies shared dependencies marks every instance unhealthy when that dependency blips, removing the entire service from rotation and turning a brief outage into a complete one. Keeping readiness checks shallow enough to remain independent, and using liveness separately for restart decisions, is what keeps both failure modes bounded.

**Graceful shutdown ordering determines whether deploys are visible to users.** Deregistering first, waiting long enough for every cache and DNS TTL to expire, and only then refusing connections means no caller ever holds an address that has stopped answering. Omitting the wait produces a small burst of errors during every rolling deploy — frequent enough to become background noise that teams stop investigating, and entirely avoidable.

**Where discovery logic lives has shifted over time for good reasons.** Client-side discovery balances best because the caller knows its own context, and it requires implementing and maintaining that logic in every service and language. Server-side load balancers remove the burden and add a hop. The sidecar proxy resolves the tension by putting client-side discovery in a process next to the application rather than inside it, which is much of why service meshes became attractive despite the operational weight of running a proxy everywhere.

**Two sources of truth are worse than either alone.** Adding a separate registry to a platform that already tracks instance lifecycle creates two views that disagree during deploys and scaling events, and callers then receive different answers depending on which they consult. Because the platform controls the lifecycle it observes it authoritatively, while an external registry infers it from registrations and probes — so where the platform provides discovery, using anything else introduces a disagreement that surfaces at the worst moment.

**Related patterns:** API Gateway · Sidecar · Heartbeat Failure Detection · Gossip Membership

---

### Sidecar

*Run shared infrastructure concerns in a separate process alongside each service instance, so every language gets the same behaviour without a library.*

> **When you hear…** Several languages needing the same infrastructure behaviour · libraries requiring redeploys to upgrade · a service mesh being adopted

**Flow:** `Service container` → `Sidecar alongside` → `Shares network and lifecycle` → `Handles cross-cutting work` → `Service stays focused`

**The problem**

Retries, circuit breaking, mutual TLS, tracing and metrics have to behave identically in services written in four languages. A shared library means four implementations that drift apart, and upgrading any of them requires every service to redeploy.

A security fix in that library therefore propagates at the speed of the slowest team's release cycle, which for infrequently changed services can be months.

> **Put shared behaviour in a process, not in a dependency**  
> A separate process beside the application, intercepting its network traffic, implements the behaviour once for every language — because it operates on connections rather than on function calls. It also upgrades independently: replacing the sidecar changes the behaviour without rebuilding or redeploying the application it serves.

**Mental model**

A companion process deployed with each instance, sharing its lifecycle and network namespace, handling concerns the application should not implement.

1. **Deploy** — The sidecar is scheduled alongside the instance and starts with it.
2. **Intercept** — Inbound and outbound traffic is routed through it.
3. **Handle** — Encryption, retries, discovery, limiting and telemetry happen there.
4. **Forward** — The application sees plain local traffic and implements none of it.
5. **Upgrade** — The sidecar is replaced without touching the application.

> **Every instance now has two processes that can fail**  
> A sidecar that crashes, is misconfigured or exhausts its memory takes the instance with it, and it doubles the resource footprint of the deployment. The failure modes are also less familiar — a proxy rejecting traffic due to a certificate problem looks to the application like the network being broken — which makes debugging harder until teams learn where to look.

**How it works**

**What it takes over, and what it costs**

```text
TYPICAL RESPONSIBILITIES
  mutual TLS between services
  service discovery and load balancing
  retries, timeouts, circuit breaking
  rate limiting
  metrics, distributed tracing, access logs
  traffic shifting for canaries
  -> all identical regardless of language

APPLICATION AFTER ADOPTION
  connects to localhost
  no TLS code, no retry code, no discovery code
  -> the application is genuinely simpler

RESOURCE COST
  ~50-100 MB memory and a fraction of a core per
  sidecar
  1,000 instances -> 50-100 GB of memory and
  substantial CPU just for proxies
  -> a real cost that must be justified by the
     consistency gained

LATENCY COST
  two extra hops (out through one proxy, in through
  another)
  typically 0.5-2 ms total
  -> negligible for most calls
  -> significant for very fast internal calls made
     in a tight loop

WHEN IT IS NOT WORTH IT
  one language, few services
  -> a library gives the same behaviour for far less
```

1. **Adopt it when several languages need identical behaviour** — That is the problem it solves better than any alternative.
2. **Share lifecycle and network namespace** — Otherwise the coupling is unreliable and the interception fragile.
3. **Keep the application unaware of it** — Configuration leaking into the application defeats the separation.
4. **Budget the resource overhead explicitly** — At a thousand instances the proxies are a substantial line item.
5. **Ensure it starts before and stops after the application** — Startup ordering causes failures that are difficult to diagnose.
6. **Consider a library for single-language estates** — The sidecar's advantage is polyglot consistency, which a uniform estate does not need.

**Startup ordering and debugging**

```text
STARTUP RACE
  application starts, makes a call
  sidecar not ready yet
  -> connection refused
  -> looks like a downstream outage, is not
  -> fix: the sidecar must be ready before the
     application starts

SHUTDOWN RACE
  sidecar exits first
  application still finishing in-flight requests
  -> those requests fail
  -> fix: the sidecar drains last

DEBUGGING SHIFT
  before: a failed call is the application's problem
  after:  it may be the application, the local
          sidecar, the remote sidecar, or the remote
          application
  -> four places instead of two
  -> mesh observability is not a luxury; it is what
     makes this tractable

CONFIGURATION SURFACE
  mesh policy is powerful and easy to get wrong
  a misapplied retry policy can amplify load
  a wrong mTLS setting blocks all traffic
  -> treat mesh configuration with the same care as
     production code
```

| Metric | Value | Note |
|---|---|---|
| Consistency | all languages | **one implementation** |
| Overhead | 50-100 MB | per instance |
| Latency | 0.5-2 ms | two hops |
| Debug surface | four components | instead of two |

> **Startup and shutdown ordering produces failures that look like network problems**  
> An application that starts before its sidecar is ready gets connection refusals that appear to be downstream outages, and a sidecar that exits before the application drains causes in-flight requests to fail during every deploy. Both are ordering bugs in the deployment definition rather than faults in either process, and both are commonly misdiagnosed for weeks because the symptoms point elsewhere.

**Technologies**

| Use | Examples | Note |
|---|---|---|
| Service mesh data plane | Envoy in Istio, Linkerd proxy | The dominant use of the pattern |
| Log and metric shipping | Collector sidecars | Simple, low-risk adoption |
| Secret management | Agents fetching and refreshing credentials | Keeps secret handling out of the application |
| Protocol adaptation | Translating between protocols | Legacy integration without changing the service |
| Shared library | In-process equivalent | Cheaper for a single-language estate |
| Node agent | One agent per host instead of per pod | Lower overhead, weaker isolation |

The per-host agent is a meaningful middle ground: one proxy serving every instance on a machine costs far less memory than one per instance, at the price of weaker isolation between workloads and a larger blast radius when it fails.

**Trade-offs**

**Delivering cross-cutting behaviour**

| Approach | Language independence | Upgrade cost | Overhead |
|---|---|---|---|
| Per-service implementation | None | Very high | None |
| Shared library | Per language | Every service redeploys | None |
| Sidecar | Complete | Replace the sidecar | Memory and CPU per instance |
| Node agent | Complete | Replace the agent | Per host |
| Gateway only | Complete for north-south | One component | One hop |

The upgrade column is the argument that usually decides it: a library fix reaches production at the pace of the slowest team's release, while a sidecar fix is a rollout the platform team performs directly.

> **Ask before choosing it**  
> How many languages are actually in use, and how often does the shared behaviour change? A single-language estate with stable requirements gets the same result from a library with none of the overhead, and adopting a mesh for consistency you already have is a large cost for no gain.

**How it fails**

**How sidecars go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Connection refused at startup | Application starting before the sidecar is ready | Enforce startup ordering |
| Requests failing during deploys | Sidecar exiting before the application drains | Sidecar drains last |
| Memory cost unexpectedly large | Overhead multiplied across instances | Budget it; consider a node agent |
| Failures hard to attribute | Four components in every call path | Invest in mesh observability |
| Retry amplification | Retries configured in both mesh and application | Retry in one layer only |
| All traffic blocked | Misapplied mutual TLS policy | Stage policy changes; treat config as code |
| Latency on hot paths | Proxy hops on very fast calls | Bypass the mesh for latency-critical paths |

> **Retries configured in both the mesh and the application multiply silently**  
> A mesh retry policy applied to a service whose client library already retries produces the product of the two, so three attempts each becomes nine — and because neither configuration is visibly wrong, the amplification is only discovered during an incident when a struggling dependency receives many times the expected load. Adopting a mesh requires auditing and removing application-level retries, not merely adding a policy.

**Where it is used**

- **Service meshes**, where a proxy per instance is the data plane and the pattern's dominant use.
- **Kubernetes deployments**, using containers in the same pod for logging, secrets and proxying.
- **Polyglot estates**, obtaining identical mutual TLS and telemetry across four or five languages.
- **Legacy integration**, adapting protocols beside a service that cannot be modified.
- **Observability pipelines**, shipping logs and metrics without instrumenting every application.

**In the interview**

Make the polyglot argument explicitly, then be honest about the overhead and the debugging cost.

- **Frame it as library versus process**: a sidecar works on connections, so it serves every language identically.
- **Emphasise independent upgrades**, since a library fix propagates only as fast as the slowest team redeploys.
- **Quantify the overhead** — tens of megabytes per instance, multiplied across the fleet — rather than calling it small.
- **Raise startup and shutdown ordering**, which produce failures that masquerade as network problems.
- **Warn about duplicated retries** between the mesh and the application, because the multiplication is invisible until an incident.

**Practice drill**

Four services in three languages must all use mutual TLS, consistent retries and distributed tracing. Compare implementing this as a shared library against a sidecar, including the upgrade path for a security fix. Compute the memory overhead of the sidecar across 300 instances. Then describe the startup and shutdown ordering you require, and explain what a developer would see if either were wrong.

**Go deeper**

A sidecar runs infrastructure concerns in a process deployed alongside each service instance, so they are implemented once for every language.

**It converts a dependency problem into a deployment problem, which is easier to manage.** A library must be implemented per language and upgraded by redeploying every service, so a fix reaches production at the pace of the slowest team. A sidecar operates on network traffic rather than on function calls, so one implementation serves everything, and replacing it is a rollout the platform team performs without any application changing.

**Its advantage is specifically about heterogeneity.** In a single-language estate a library provides the same behaviour with no extra processes, no additional hops and no resource overhead, so adopting a sidecar there buys consistency that already existed. The pattern earns its cost when several languages must behave identically, or when the behaviour changes often enough that redeploying everything to update it is genuinely painful.

**The overhead is real and multiplies with fleet size.** Tens of megabytes and a fraction of a core per instance is unremarkable for one service and becomes a substantial infrastructure line item across a thousand, which is why per-host agents exist as a middle ground — lower total overhead, weaker isolation, larger blast radius. Treating the cost as negligible is the mistake; treating it as a considered trade against consistency is the correct framing.

**It widens the debugging surface in a way teams underestimate.** A failed call now involves the application, its local proxy, the remote proxy and the remote application, and a proxy rejecting traffic because of a certificate problem presents to the application as an unreachable network. Without strong mesh observability the additional indirection makes ordinary incidents considerably harder to diagnose, which is why the telemetry the sidecar provides is part of what makes it tolerable.

**Lifecycle ordering is the most common source of confusing failures.** An application starting before its proxy is ready receives connection refusals that look like downstream outages; a proxy exiting before the application drains fails in-flight requests during every deploy. Both are properties of the deployment definition rather than of either process, and both typically persist for a long time because the symptoms point at something else entirely.

**Policy in the mesh must be reconciled with policy in the application.** Retries configured in both layers multiply, so an application retrying three times behind a mesh retrying three times sends nine requests to a struggling dependency — an amplification invisible in either configuration and discovered during an incident. Adopting a mesh therefore includes auditing and removing the application-level behaviour it replaces, which is a migration rather than an addition.

**Related patterns:** Service Discovery · API Gateway · Circuit Breaker · Retry With Jitter

---

### Aggregator

*Have one service call several others and combine their responses, so a client makes one request instead of coordinating many.*

> **When you hear…** A screen needing data from several services · clients orchestrating service calls · chatty client-server interaction

**Flow:** `One client request` → `Fan out in parallel` → `Collect responses` → `Merge and shape` → `Single response`

**The problem**

An order details page needs the order, the customer, the shipment status and the payment record, each owned by a different service. If the client fetches them itself, it makes four calls, handles four failures, and needs to know which services exist and how they relate.

That knowledge belongs in the system, not the client. It also means every change to the service topology requires a client change, and clients that cannot be updated quickly — mobile applications in particular — hold the architecture still.

> **Composition is a server-side responsibility**  
> The services are close to one another and far from the client, so combining their responses server-side turns four slow round trips into one, plus four fast internal ones issued in parallel. The client is then insulated from the topology, and the decisions about partial failure — what to do when one of the four is unavailable — are made in a place that has the context to make them well.

**Mental model**

A service whose job is calling other services concurrently, merging their results, and returning one coherent response.

1. **Receive** — One request arrives describing what the client needs.
2. **Fan out** — Calls to each required service are issued in parallel.
3. **Collect** — Responses are gathered within a deadline.
4. **Handle gaps** — Missing pieces are filled with defaults or omitted.
5. **Merge** — One shaped response is returned.

> **Availability is the product of every dependency unless partial failure is handled**  
> Four services at 99.9% each give 99.6% if all must succeed — roughly three hours of monthly failure created purely by composition. Returning something useful when one dependency is missing is not a refinement; it is what stops the aggregator being less reliable than any of the services behind it.

**How it works**

**Parallel fan-out with partial results**

```text
SEQUENTIAL  (the common mistake)
  order    80 ms
  customer 60 ms
  shipment 90 ms
  payment  70 ms
  total = 300 ms

PARALLEL
  all four issued at once
  total = max(80, 60, 90, 70) = 90 ms
  -> the whole benefit of the pattern lives here

WITH DEPENDENCIES
  order first (needed for the others' ids)
  then customer, shipment, payment in parallel
  total = 80 + 90 = 170 ms
  -> parallelise every stage that can be

DEADLINE AND PARTIAL RESULTS
  overall budget 200 ms
  each call gets a timeout inside it
  at the deadline, return what arrived

  order    ok      required -> without it, fail
  customer ok      required
  shipment TIMEOUT optional -> omit the section
  payment  ok      optional

  -> return the page with shipment status missing
  -> mark it as unavailable rather than absent

REQUIRED VERSUS OPTIONAL
  classify every dependency before writing the code
  -> it determines both the failure behaviour and
     the real availability
```

1. **Issue independent calls concurrently** — Sequential fan-out captures almost none of the benefit.
2. **Classify each dependency as required or optional** — It decides what a missing response does to the response.
3. **Give the whole request a deadline and each call a share** — Without one, the slowest dependency defines the latency.
4. **Return partial results rather than failing** — Composed availability is otherwise the product of every dependency.
5. **Mark missing sections explicitly** — A silently absent section is indistinguishable from empty data.
6. **Keep it stateless and free of business rules** — An aggregator that decides things becomes a service with hidden ownership.

**Availability arithmetic**

```text
ALL REQUIRED
  4 dependencies at 99.9%
  0.999^4 = 99.60%
  -> ~3 hours unavailable per month from composition
     alone

TWO REQUIRED, TWO OPTIONAL
  0.999^2 = 99.80% for a complete response
  but a USEFUL response requires only the two
  -> the page renders through most failures
  -> availability as experienced is much higher than
     the arithmetic suggests

CACHING OPTIONAL DEPENDENCIES
  serve the last known value when a service is down
  -> the section is stale rather than missing
  -> usually better for the user, and must be
     labelled

WHAT THE AGGREGATOR MUST NOT BECOME
  a place where business rules live
  -> it has no data of its own and no clear owner
  -> rules there are invisible to the teams who own
     the domains
  -> keep it to calling, waiting, merging, shaping
```

| Metric | Value | Note |
|---|---|---|
| Sequential | 300 ms | avoidable |
| Parallel | 90 ms | **the point** |
| All required | 99.6% | from 99.9% parts |
| Partial results | much higher | as experienced |

> **An aggregator without partial-failure handling is less available than anything it calls**  
> Requiring every dependency multiplies their failure rates, so a composition of four highly available services is measurably less available than each of them. The component added to simplify the client has then become the least reliable thing in the path, and the only remedy is deciding, per dependency, what the response looks like without it.

**Technologies**

| Approach | Strength | Watch for |
|---|---|---|
| A dedicated aggregation service | Full control over merging and failure | Another service to operate |
| Backend for frontend | Aggregation shaped per client | One per client type |
| GraphQL server | Client-driven composition | Query cost and caching |
| Gateway aggregation | No new service | Logic accumulating in the gateway |
| Client-side composition | No server component | Round trips and topology coupling |
| Materialised view | Precomputed joins | Staleness; a write-side commitment |

A materialised view is the alternative worth remembering: where the same combination is requested constantly and tolerates slight staleness, precomputing the joined result removes the fan-out entirely and replaces it with a single read.

**Trade-offs**

**Where composition happens**

| Location | Client round trips | Topology coupling | Failure handling |
|---|---|---|---|
| Client-side | One per service | High | In the client |
| Aggregator service | One | None | Server-side, with context |
| Backend for frontend | One | None | Per client type |
| GraphQL | One | None | Per field resolution |
| Materialised view | One | None | Precomputed; staleness instead |

Client-side composition is not always wrong: a web application on a fast connection making three parallel calls is perfectly reasonable, and the pattern earns its place where round trips are expensive or where failure handling needs context the client does not have.

> **Ask before choosing it**  
> Which of these dependencies must succeed for the response to be useful? Answering that first produces both the failure design and an honest availability figure, and teams that skip it usually discover the composed availability during an incident rather than during design.

**How it fails**

**How aggregators go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Latency equals the sum of dependencies | Sequential calls | Fan out concurrently |
| Less available than any dependency | All dependencies treated as required | Classify and return partial results |
| One slow service delays everything | No per-call timeout | Deadline budget split across calls |
| Missing data looks like empty data | Sections omitted silently | Explicitly mark unavailable sections |
| Thread pool exhausted | Blocking calls to slow dependencies | Asynchronous calls with bounded concurrency |
| Business rules hidden in the aggregator | Logic accumulating in a layer with no owner | Rules belong in domain services |
| Cascading retries | Aggregator retrying while clients also retry | Retry at one layer |

> **Sequential fan-out discards the entire benefit while keeping all the costs**  
> An aggregator calling four services one after another produces the same total latency the client would have experienced, plus an additional hop, plus a new service to operate. It looks correct in code review and in a single-service trace, and it is only visible as a problem when someone compares the response time against the sum of the dependency latencies — at which point the pattern has been paying for itself in nothing but complexity.

**Where it is used**

- **Product and order detail pages**, combining data from several domain services.
- **Mobile backends**, where one aggregated call replaces several expensive round trips.
- **Search result enrichment**, merging results with inventory, pricing and personalisation.
- **Dashboards**, gathering metrics from many sources into one response.
- **GraphQL servers**, which are aggregators driven by a client-supplied query.

**In the interview**

Two things carry the answer: parallel fan-out and the availability arithmetic.

- **Show the latency difference** between sequential and parallel fan-out with concrete numbers.
- **Do the availability multiplication** — four services at three nines give 99.6% — to motivate partial results.
- **Classify dependencies as required or optional** and describe the response in each failure case.
- **Give the whole request a deadline** and split it across calls, so one slow service cannot define the latency.
- **Keep business rules out of it**, because a layer owning no data and no domain is the wrong place for decisions.

**Practice drill**

An order page needs data from four services with latencies of 80, 60, 90 and 70 milliseconds, each 99.9% available. Compute the response time for sequential and parallel fan-out, and the availability if all are required. Classify which are genuinely required, design the partial-response behaviour, and state the resulting user-visible availability. Then describe what happens when one service is slow rather than down, and what the client receives.

**Go deeper**

An aggregator composes responses from several services into one, so clients make a single request and remain unaware of the topology behind it.

**Its benefit comes from where the network is fast.** Services sit close to each other and far from the client, so moving composition to the server replaces several expensive round trips with one, plus several cheap internal ones. That advantage is entirely dependent on issuing the internal calls concurrently — a sequential implementation produces the same latency as the client would have had, with an extra hop and an extra service to run.

**Composition multiplies failure unless partial results are designed in.** Four dependencies at three nines yield 99.6% when all must succeed, which is worse than any individual component and amounts to hours of monthly unavailability produced by the composition itself. Classifying each dependency as required or optional, and defining what the response looks like without each optional one, is what keeps the aggregate at least as available as its most critical part.

**A deadline budget is what makes slow dependencies survivable.** Without one, the response time is dictated by whichever service is slowest at that moment, and a single degraded dependency makes every request slow rather than making one section missing. Allocating an overall budget and giving each call a share means the aggregator returns on time with whatever arrived, which is almost always more useful than a complete response that came too late to matter.

**Missing data must be distinguishable from empty data.** A section omitted because a service timed out looks identical to a section that is genuinely empty, so a client cannot tell whether to show nothing or to show an error. Marking unavailable portions explicitly lets the interface say that shipment status could not be loaded rather than implying that no shipment exists, which is a materially different statement to the user.

**It must remain a composition layer with no opinions.** An aggregator owns no data and belongs to no domain, so business rules placed there are invisible to the teams responsible for the concepts they govern, and they drift from the rules those teams maintain. Keeping the layer to calling, waiting, merging and shaping is what prevents it from becoming a service with hidden ownership and unclear authority.

**Where the same composition is requested constantly, precomputing it is often better.** A materialised view of the joined result turns a fan-out into a single read and removes the availability multiplication entirely, at the cost of staleness and a write-side pipeline to maintain. That trade is frequently worthwhile for high-traffic read paths, and considering it before building an aggregator is what distinguishes choosing the pattern from reaching for it.

**Related patterns:** Scatter Gather · Backend For Frontend · API Gateway · Materialized View

---

### Scatter Gather

*Send the same request to many nodes in parallel and combine their partial answers, so work that spans a dataset completes in the time of one node.*

> **When you hear…** A query that must consult every shard · search across a partitioned index · parallel computation over distributed data

**Flow:** `Query arrives` → `Broadcast to shards` → `Each searches locally` → `Partial results returned` → `Merged and ranked`

**The problem**

A search index is spread across sixty-four shards because it does not fit on one machine. A query has no way to know which shard holds the best matches, so consulting one is not an option — the answer depends on all of them.

Consulting them one after another takes sixty-four times as long as consulting one, which for an interactive search is hopeless. The data was partitioned to make it fit, and the query must now be made to work across that partitioning.

> **Broadcast the query, and let the fan-out cost be latency rather than throughput**  
> Because each shard holds a disjoint portion, all of them can search simultaneously and the elapsed time is that of a single shard rather than their sum. The query is duplicated across the fleet — total work is unchanged — but the user waits only as long as the slowest response, which turns an impossible sequential cost into a tractable parallel one.

**Mental model**

One coordinator broadcasting an identical request to every partition, then merging the partial results into a final answer.

1. **Broadcast** — The same query is sent to every shard.
2. **Search locally** — Each shard produces its best partial answer.
3. **Return** — Partial results, usually with scores, come back.
4. **Merge** — The coordinator combines and ranks them.
5. **Bound** — A deadline decides what to do about shards that have not replied.

> **Latency is set by the slowest shard, so component tails become the system median**  
> With sixty-four shards, a 1% chance of any one being slow means roughly a 47% chance that some shard is slow on every query. The 99th percentile of a shard therefore becomes something close to the typical experience of the query, which is why large fan-out systems must attack tail latency directly rather than average latency.

**How it works**

**Tail latency and the merge**

```text
THE ARITHMETIC
  P(a shard is slow) = 1%
  64 shards
  P(all fast) = 0.99^64 = 0.526
  -> ~47% of queries hit at least one slow shard
  -> the shard p99 becomes near the query median

MITIGATIONS
  hedged requests   send a duplicate to a replica
                    after the usual latency elapses
  deadline          return at the budget with what
                    arrived
  partial results   report coverage: "62 of 64 shards"
  -> most search systems use all three

THE MERGE
  each shard returns its local top K with scores
  coordinator merges and takes the global top K
  -> scores must be COMPARABLE across shards
  -> local relevance normalisation breaks this
     silently, producing subtly wrong rankings

HOW MANY TO REQUEST FROM EACH SHARD
  want top 10 globally
  request top 10 from each shard, not top 1
  -> all 10 could be on one shard
  -> requesting top 1 each gives a wrong answer

DEEP PAGINATION
  page 100 of 10 results
  each shard must return its top 1000 to guarantee
  correctness
  -> cost grows linearly with the page number
  -> which is why deep pagination is usually capped
```

1. **Request the full result count from every shard** — Asking each for a fraction produces incorrect global rankings.
2. **Ensure scores are comparable across shards** — Locally normalised scores make the merge quietly wrong.
3. **Set a deadline and return partial results** — Waiting for every shard means the slowest always wins.
4. **Report coverage with partial responses** — The consumer needs to know the answer is incomplete.
5. **Hedge against slow shards** — Duplicating to a replica after the usual latency cuts the tail sharply.
6. **Cap pagination depth** — Cost per page grows linearly and deep pages are rarely used.

**Where the cost lands**

```text
THROUGHPUT COST
  one query = 64 shard queries
  1,000 queries/s = 64,000 shard queries/s
  -> the cluster must be sized for the fan-out, not
     for the query rate

LATENCY BENEFIT
  sequential: 64 x 20 ms = 1,280 ms
  parallel:   max over 64 ≈ 20-60 ms
  -> the entire reason the pattern exists

REDUCING THE FAN-OUT
  route by a partition key where the query has one
    "orders for customer X" -> one shard
  filter shards by metadata
    date-partitioned index, query has a date range
    -> touch only the relevant shards
  -> the best optimisation is not broadcasting

WHEN NOT TO USE IT
  the query has a natural partition key
    -> route directly, do not broadcast
  results needed from only one partition
  the fan-out exceeds what the cluster can sustain
    at the query rate
```

| Metric | Value | Note |
|---|---|---|
| Latency | one shard | **not the sum** |
| Throughput | × shard count | size for fan-out |
| 64 shards | 47% hit a straggler | tail dominates |
| Best fix | do not broadcast | route by key |

> **Requesting fewer results per shard than the global count returns the wrong answer**  
> If ten results are wanted overall and each shard is asked for one, the correct answer is lost whenever several of the best matches sit on the same shard — which is common, because relevance and partitioning are unrelated. Each shard must return the full count so the merge has enough candidates, and this is the reason deep pagination becomes expensive rather than merely slow.

**Technologies**

| System | Use | Note |
|---|---|---|
| Elasticsearch and OpenSearch | Distributed search across shards | The canonical implementation |
| Distributed SQL engines | Parallel scans across partitions | Aggregation pushed down where possible |
| MapReduce-style frameworks | Batch scatter-gather | The same idea at a much coarser grain |
| Sharded databases | Queries without a shard key | Works, and should be the exception |
| Hedged requests | Tail latency control | Essential at large fan-out |
| Routing by key | Avoiding the fan-out entirely | The best optimisation available |

Pushing aggregation down to the shards matters more than it appears: having each shard compute a local count or sum and returning only those values, rather than raw rows, reduces the data crossing the network by orders of magnitude and moves the work to where the data already is.

**Trade-offs**

**Query routing strategies**

| Strategy | Shards touched | Latency | Correctness |
|---|---|---|---|
| Route by partition key | One | One shard | Exact |
| Filter shards by metadata | A subset | Slowest of the subset | Exact |
| Scatter-gather all shards | All | Slowest shard | Exact |
| Scatter-gather with a deadline | All attempted | Bounded | Partial |
| Sample a subset of shards | Some | Fast | Approximate |

Sampling is a legitimate strategy for analytics and trend queries where an approximate answer computed quickly is more valuable than an exact one computed slowly — and it is completely wrong for anything a user expects to be complete.

> **Ask before choosing it**  
> Does the query contain something that identifies a partition? If it does, routing directly is faster, cheaper and more reliable than broadcasting, and the fan-out should be reserved for the queries that genuinely cannot be narrowed.

**How it fails**

**How scatter-gather goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Latency dominated by stragglers | Waiting for every shard | Deadlines, partial results, hedging |
| Wrong results returned | Too few results requested per shard | Request the full count from each |
| Rankings subtly incorrect | Scores not comparable across shards | Global scoring or normalised statistics |
| Cluster saturated | Fan-out multiplied by query rate | Route by key where possible; add replicas |
| Deep pagination times out | Cost growing with page depth | Cap depth; use cursors |
| Partial results presented as complete | Coverage not reported | Return which shards answered |
| Memory exhausted in the coordinator | All raw results merged in memory | Push aggregation down; stream the merge |

> **Fan-out multiplies every query against the entire cluster**  
> A thousand queries per second across sixty-four shards is sixty-four thousand shard queries per second, so the cluster must be provisioned for the product rather than for the query rate. This is why reducing the fan-out — routing by key, filtering shards by metadata, denormalising so the common query touches one partition — is worth far more than optimising the individual shard query, and why systems that broadcast everything hit a capacity wall that more shards do not fix.

**Where it is used**

- **Distributed search engines**, the defining use, broadcasting queries across an index.
- **Analytics engines**, scanning partitions in parallel and merging aggregates.
- **Sharded databases**, for queries that lack a shard key and must consult everything.
- **Batch processing frameworks**, applying the same idea with much longer time scales.
- **Federated queries**, fanning out across independent systems and combining their answers.

**In the interview**

The tail arithmetic is the centre of this answer; without it the pattern sounds straightforward and is not.

- **Compute the straggler probability** — a 1% shard tail across 64 shards affects nearly half of queries.
- **Say latency equals the slowest shard**, and propose deadlines, partial results and hedging as the mitigations.
- **Explain the top-K merge correctly**: each shard returns the full count, not a share, or the answer is wrong.
- **Size for the fan-out**, since one query becomes as many shard queries as there are shards.
- **Propose avoiding the broadcast** by routing on a partition key wherever the query permits it.

**Practice drill**

A search index has 64 shards, each answering in 20 ms at the median and 200 ms at the 99th percentile. Compute the probability that a query touches a slow shard and estimate the query's p99. Design the mitigations: deadline, hedging and partial-result reporting. Then explain why asking each shard for the top 1 when 10 results are wanted is incorrect, and what deep pagination does to the cost.

**Go deeper**

Scatter-gather broadcasts one request to every partition in parallel and merges the partial answers into a complete result.

**It converts a sequential cost into a parallel one.** Consulting sixty-four shards one after another takes sixty-four times a single shard's latency, which is unusable interactively; consulting them simultaneously takes as long as the slowest. Total work is unchanged — the cluster performs the same number of shard queries either way — so what the pattern buys is latency, paid for in throughput capacity.

**Tail latency dominates, and this is the defining characteristic at scale.** The chance that at least one shard is slow rises quickly with fan-out, so a component's 99th percentile becomes close to the query's typical experience. Improving each shard's median does almost nothing for this; only compressing tails helps, which is why deadlines with partial results and hedged requests to replicas are standard rather than optional in large fan-out systems.

**The merge has a correctness requirement that is easy to violate.** Each shard must return the full desired result count rather than a share of it, because the best matches may be concentrated on one partition — and the scores those shards return must be comparable, or the coordinator ranks apples against oranges. Locally normalised relevance scores break this quietly, producing rankings that look plausible and are wrong, which is among the harder bugs to notice in a search system.

**Deep pagination degrades in a way that surprises people.** Returning the hundredth page of ten results requires every shard to produce its top thousand so the merge can be correct, so cost grows linearly with page depth while the value of those pages approaches zero. Capping depth or switching to cursor-based navigation is the standard resolution, and it is a product decision as much as a technical one.

**Throughput must be provisioned for the product of query rate and fan-out.** A thousand queries per second across sixty-four shards is sixty-four thousand shard queries, so the cluster is sized by the broadcast rather than by the external rate. Adding shards to handle more data also multiplies this, which means fan-out systems have a capacity relationship that does not improve as they grow — a reason to take shard-count decisions seriously rather than treating them as a storage detail.

**The strongest optimisation is to avoid broadcasting at all.** A query carrying a partition key can be routed to one shard, and metadata filters — time ranges against a time-partitioned index, for instance — can exclude most shards before the request is sent. Denormalising so that the highest-volume queries have a routable key is frequently worth considerable effort, because it moves those queries from an expensive fan-out to a single lookup and leaves the broadcast for the queries that genuinely need it.

**Related patterns:** Aggregator · Hedged Requests · Sharding · Inverted Index

---

## Data Processing

### Batch Processing

*Process a bounded set of data as one job on a schedule, trading freshness for throughput, simplicity and the ability to reprocess.*

> **When you hear…** Large volumes to process efficiently · results that can be hours old · computations that must be reproducible

**Flow:** `Data accumulates` → `Job starts on schedule` → `Reads a bounded set` → `Computes results` → `Writes output atomically`

**The problem**

Billing needs to aggregate a month of usage across millions of accounts. Doing this incrementally as events arrive means maintaining running state everywhere, handling late and corrected events in that state, and having no straightforward way to recompute if a rule changes.

It also processes each record individually, which is the least efficient way to touch large volumes — losing every advantage of sequential reads, bulk writes and columnar formats.

> **A bounded input makes correctness tractable**  
> When a job knows exactly which records it is processing, it can read them efficiently, compute a complete answer, and write the result atomically. There is no partial state to maintain between runs, no ambiguity about what has been included, and rerunning the job with corrected logic produces a corrected answer — because the input is still there and the computation is a pure function of it.

**Mental model**

A scheduled job over a fixed window of data, producing an output that replaces rather than amends the previous one.

1. **Accumulate** — Data collects in storage until the job runs.
2. **Bound** — The job selects a definite input — a day, a partition, a range.
3. **Process** — The computation runs over that whole set, usually in parallel.
4. **Write** — Output is produced and published atomically.
5. **Repeat** — The next run takes the next window, independently.

> **Publishing output incrementally exposes a half-computed result**  
> A job writing directly into the table consumers read makes partially processed data visible for the duration of the run, and a failure mid-way leaves it permanently inconsistent. Writing to a new location and switching consumers to it atomically is what makes a batch job safe to fail — the previous output remains correct and complete until the new one is entirely ready.

**How it works**

**Idempotence, partitioning and late data**

```text
IDEMPOTENT BY PARTITION
  each run writes to a partition determined by its
  input window
    output/date=2026-09-14/
  rerunning replaces that partition entirely
  -> reruns are safe, which is the property that
     makes batch operable

  NOT idempotent: appending to a single output
  -> a rerun doubles the results, silently

ATOMIC PUBLISH
  write to output/_tmp/date=.../
  verify
  atomically move or swap the pointer
  -> consumers never observe a partial result

LATE DATA
  the job for 14 September runs at 02:00 on the 15th
  an event with timestamp 23:58 on the 14th arrives
  at 02:05
  -> it is not in the output

  options:
    wait longer before running (delays everything)
    reprocess affected days on a slower cadence
    accept the loss for low-value data
  -> this must be a decision, not a discovery

BACKFILL
  logic changes -> rerun historical partitions
  -> only possible because the input is retained and
     the computation is deterministic
  -> this is batch processing's defining advantage
```

1. **Make each run idempotent by partition** — Reruns are routine, and appending makes them destructive.
2. **Publish output atomically** — Consumers must never see a partially computed result.
3. **Keep the raw input** — Reprocessing is the main advantage, and it requires the source data.
4. **Decide the late-data policy explicitly** — Every batch boundary excludes something; the question is what happens to it.
5. **Partition input and output by time** — It makes both parallelism and selective reprocessing straightforward.
6. **Monitor duration against the schedule** — A job whose runtime approaches its interval is about to overlap with itself.

**Why batch is efficient, and where it stops working**

```text
EFFICIENCY
  sequential reads over columnar files
  bulk writes rather than per-record transactions
  compression across many rows
  parallelism across partitions
  -> often 10-100x cheaper per record than
     processing events individually

THE COST IS FRESHNESS
  hourly job  -> results up to 1 hour old
  daily job   -> results up to 24 hours old
  -> plus the job's own runtime

WHEN THE SCHEDULE BREAKS DOWN
  job takes 45 min, scheduled hourly
  volume grows 50%
  -> job takes 68 min
  -> runs overlap, resources contend, both slow
  -> a queue of delayed runs forms and never drains
  -> alert on duration / interval, not just failures

WHEN TO USE STREAMING INSTEAD
  results needed in seconds
  the value of an answer decays quickly
  -> but keep batch for correction and backfill;
     most mature systems run both
```

| Metric | Value | Note |
|---|---|---|
| Efficiency | 10-100× cheaper | **per record** |
| Freshness | interval + runtime | the trade |
| Reruns | safe, by partition | the key property |
| Watch | duration vs interval | overlap risk |

> **A job whose runtime approaches its interval will eventually overlap with itself**  
> Growing data makes jobs take longer, and a job scheduled hourly that now takes fifty minutes has almost no margin. When it crosses the boundary, two runs execute concurrently, compete for the same resources, and both slow down — producing a backlog that never drains without intervention. The ratio of runtime to interval is the metric to alert on, long before any job actually fails.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Processing engines | Spark and similar | Distributed computation over partitioned data |
| Warehouse SQL | Scheduled queries | Simplest option when the data is already there |
| Orchestration | Airflow, Dagster and similar | Dependencies, retries, backfills |
| Storage formats | Parquet, ORC | Columnar, compressed, partition-aware |
| Object storage | Partitioned by date | Cheap, durable, atomic swaps by prefix |
| Streaming | The low-latency alternative | Usually complementary rather than a replacement |

Orchestration is what turns a collection of jobs into a system: dependency ordering, retry policy, backfill tooling and alerting are most of the operational work, and they matter more than the choice of processing engine for all but the largest workloads.

**Trade-offs**

**Batch against streaming**

| Property | Batch | Streaming |
|---|---|---|
| Latency | Minutes to hours | Seconds or less |
| Cost per record | Low | Higher |
| Reprocessing | Natural; rerun the job | Requires replay and care |
| Late data | Handled by rerunning | Handled by windows and watermarks |
| Operational complexity | Lower | Higher; always running |
| Failure recovery | Rerun the partition | Resume from a checkpoint |

The reprocessing row is why batch persists even where streaming is available. A rule change or a bug fix applied to a batch job is a rerun over the affected partitions; the same change to a streaming pipeline requires replaying history through a system designed for the present.

> **Ask before choosing it**  
> How old may this answer be before it stops being useful? A dashboard refreshed hourly and a fraud decision needed in milliseconds are different problems, and the acceptable staleness — not the data volume — is what decides between batch and streaming.

**How it fails**

**How batch processing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Duplicated results after a rerun | Output appended rather than replaced | Idempotent partition overwrite |
| Consumers read partial output | Non-atomic publish | Write elsewhere, then swap atomically |
| Overlapping runs | Runtime approaching the interval | Alert on the ratio; scale or lengthen the interval |
| Late events silently dropped | No late-data policy | Reprocess affected windows on a slower cadence |
| Backfill impossible | Raw input discarded | Retain source data for the reprocessing window |
| One failure blocks everything downstream | Rigid dependency chain | Isolate failures; allow partial progress |
| Results wrong after a logic change | No reprocessing of historical partitions | Backfill as part of the change |

> **A batch job that cannot be safely rerun is not really a batch job**  
> The entire operational model assumes that any run can be repeated — after a failure, after a bug fix, after a logic change. A job that appends rather than replaces, or that mutates state incrementally, turns every rerun into a corruption risk, which means failures must be resolved by hand and improvements to the logic cannot be applied to history. Idempotence by partition is what makes the rest of the model work.

**Where it is used**

- **Billing and usage aggregation**, where exactness matters more than immediacy.
- **Analytics and reporting pipelines**, computing daily aggregates over large volumes.
- **Machine learning feature generation**, producing training data from historical events.
- **Search index rebuilds**, regenerating an index from the source of record.
- **Data warehouse loading**, the archetypal scheduled transformation.

**In the interview**

Lead with idempotence and atomic publishing; they are what make batch operationally sound.

- **Say each run replaces a partition** rather than appending, which is what makes reruns safe.
- **Describe atomic publication** so consumers never observe a half-computed result.
- **State the late-data policy explicitly**, since every batch boundary excludes something.
- **Raise runtime against interval** as the metric that predicts overlapping runs before they happen.
- **Give reprocessing as the reason batch survives** alongside streaming — a logic change is a rerun, not a replay.

**Practice drill**

Design a nightly job aggregating a day of usage events for billing. Specify the input bounds, the output partitioning, the publication mechanism and the rerun behaviour. State your policy for events that arrive after the job has run. Then the job's runtime grows from 40 to 55 minutes against an hourly schedule — say what you would alert on, what happens if nothing is done, and what your options are.

**Go deeper**

Batch processing computes results over a bounded set of data on a schedule, trading freshness for efficiency, simplicity and reproducibility.

**Bounded input is what makes the model tractable.** A job that knows exactly which records it covers produces a complete answer that is a pure function of that input, with no running state to maintain between executions and no ambiguity about what has been counted. Correctness becomes a property of one computation rather than of an ongoing process, which is a substantially easier thing to reason about and to test.

**Idempotence by partition is the property everything else rests on.** Runs fail, logic changes, and bugs are found after the fact, so every execution must be repeatable — which means writing to a partition determined by the input window and replacing its contents entirely. A job that appends instead turns each rerun into a silent duplication, and the operational model collapses into manual intervention for every failure.

**Publication must be atomic or consumers see incoherent data.** Writing directly into the tables consumers read exposes partial results for the duration of the run and leaves permanent inconsistency if the job fails midway. Producing output in a separate location and switching over once it is verified keeps the previous, complete answer available until the new one is entirely ready, which is what allows a job to fail without an incident.

**Its efficiency comes from doing everything in bulk.** Sequential reads over columnar files, compression across many rows, bulk writes instead of per-record transactions, and parallelism across partitions combine to make batch processing an order of magnitude or more cheaper per record than handling events individually. That advantage grows with volume, which is why batch remains the economical choice for large-scale aggregation regardless of what streaming infrastructure exists alongside it.

**Every boundary excludes something, so the late-data policy is part of the design.** An event arriving after its window has been processed is either lost, or captured by a later reprocessing pass, or forces the job to wait longer before running. All three are legitimate and they have different costs, but the policy has to be chosen deliberately — systems that never decide discover it as a discrepancy in a report months later, at which point the affected data may no longer be reprocessable.

**Reprocessing is why batch survives the availability of streaming.** A changed rule or a corrected bug is applied to history by rerunning the affected partitions, which is routine precisely because the input is retained and the computation is deterministic. Achieving the same in a streaming system means replaying history through infrastructure built for the present, with all the state and ordering concerns that implies — so mature architectures typically run both, using streaming for immediacy and batch for correctness and correction.

**Related patterns:** Stream Processing · Materialized View · Partitioned Log · Durable Workflow

---

### Stream Processing

*Compute continuously over unbounded event streams, maintaining state and emitting results within seconds of the events that caused them.*

> **When you hear…** Results needed in seconds · continuously updated aggregates · detection that must happen while it still matters

**Flow:** `Events arrive` → `Continuous operators` → `State per key` → `Windows close` → `Results emitted`

**The problem**

Fraud detection based on an hourly batch job identifies a fraudulent pattern up to an hour after it began, by which time the transactions have completed. The value of the answer decayed long before it was computed.

Running the batch job more frequently does not fix it: at minute-level intervals the fixed cost of starting jobs dominates, the same data is re-read repeatedly, and the latency floor remains the interval plus the runtime.

> **Keep the computation running and feed events through it**  
> If the operators are always live and hold their state in memory, each event updates the result as it arrives rather than waiting for a job to start. Latency becomes the processing time of one event instead of the interval between runs — and the price is that the system must maintain durable state continuously, handle events that arrive out of order, and decide when an answer is final.

**Mental model**

A standing dataflow of operators over an unbounded input, with keyed state, windows that bound aggregation, and checkpoints that make the state recoverable.

1. **Consume** — Events are read from a log, in partition order.
2. **Transform** — Operators filter, enrich and key the stream.
3. **Accumulate** — State per key is updated as events arrive.
4. **Window** — Aggregations are bounded by time or count.
5. **Emit** — Results are produced when a window closes or on every update.

> **Event time and arrival time are not the same, and the gap is where correctness lives**  
> A mobile device offline for ten minutes sends events timestamped ten minutes ago. Grouping by arrival time puts them in the wrong window and quietly produces wrong answers; grouping by event time requires deciding how long to wait for stragglers, which is a trade between latency and completeness that has no universally correct setting.

**How it works**

**Windows, watermarks and late events**

```text
WINDOW TYPES
  tumbling  fixed, non-overlapping
            [0-60) [60-120)
  sliding   fixed, overlapping
            every 10 s, covering the last 60 s
  session   grouped by inactivity gaps
            closes after 30 min of silence

WATERMARKS
  a claim: "no events older than T will arrive"
  -> windows ending before T can be closed
  -> derived from observed event times minus an
     allowance

  watermark lag 5 minutes
    -> results are 5 minutes behind real time
    -> and events later than 5 minutes are late

THE LATENCY / COMPLETENESS TRADE
  short watermark  fast results, more late events
  long watermark   slower results, fewer missed

LATE EVENT POLICIES
  drop            simplest, silently wrong
  side output     collect and handle separately
  update the
  emitted result  correct, but downstream must
                  handle retractions
  -> pick one deliberately; the default is usually
     drop

STATE
  keyed state lives in the operator
  checkpointed periodically to durable storage
  -> recovery restores state and resumes from the
     checkpointed offsets
  -> checkpoint interval bounds the replay on
     failure
```

1. **Use event time, not arrival time** — Arrival-time windows are wrong whenever events are delayed, which is always.
2. **Choose the watermark from observed lateness** — It is a direct trade between result latency and completeness.
3. **Define the late-event policy explicitly** — Dropping silently is a decision, and usually an unexamined one.
4. **Checkpoint state at an interval matched to acceptable replay** — Recovery replays from the last checkpoint.
5. **Keep state bounded** — Unbounded keyed state grows forever; expire keys deliberately.
6. **Retain the source log for replay** — Reprocessing after a logic change depends on it entirely.

**Operations and delivery semantics**

```text
EXACTLY-ONCE, PRECISELY
  exactly-once STATE: checkpointed offsets and state
  advance together, so recovery does not double-count
  internally
  exactly-once OUTPUT: only if the sink is
  transactional or idempotent
  -> "exactly-once" without a qualifying sink means
     at-least-once at the boundary

BACKPRESSURE
  a slow operator slows its upstream
  -> the source reads more slowly
  -> consumer lag grows
  -> lag is the primary health metric

RESTARTS ARE EXPENSIVE
  state must be restored from the checkpoint
  large state -> minutes of recovery
  -> deployments are not free, unlike a batch job
     that simply runs again

STATE SIZE
  1M keys x 1 KB = 1 GB per instance
  -> memory or embedded storage becomes the limit
  -> expiry policy is a capacity decision

WHAT MAKES IT HARDER THAN BATCH
  always running, so failures are incidents
  state to manage, restore and size
  out-of-order events
  no natural point at which an answer is final
```

| Metric | Value | Note |
|---|---|---|
| Latency | seconds | **per event** |
| Trade | watermark lag | completeness |
| State | checkpointed | recovery cost |
| Health metric | consumer lag | primary signal |

> **Exactly-once is a property of the pipeline, not of the output, unless the sink cooperates**  
> Frameworks provide exactly-once by advancing state and input offsets together, so internal recovery does not double-count. The moment results are written to an external system, the guarantee holds only if that write is transactional or idempotent — otherwise a recovery re-emits results the sink has already applied. Describing a pipeline as exactly-once without qualifying the sink overstates what it delivers.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Processing frameworks | Flink, Kafka Streams, Spark Structured Streaming | Differ in state and windowing models |
| Source | Partitioned log | Replay is what makes reprocessing possible |
| State backends | Embedded key-value stores with checkpoints | State size drives instance sizing |
| Sinks | Transactional or idempotent writes | Required for end-to-end guarantees |
| Streaming SQL | Declarative continuous queries | Lower barrier; less control |
| Batch | The complementary approach | Correction, backfill and reprocessing |

Streaming SQL has changed who can build these pipelines: windowed aggregations expressed declaratively remove most of the boilerplate, while leaving the hard decisions — watermarks, late data, state expiry — exactly where they were.

**Trade-offs**

**Streaming against batch**

| Property | Streaming | Batch |
|---|---|---|
| Latency | Seconds | Minutes to hours |
| Cost per record | Higher | Lower |
| State | Continuous, checkpointed | None between runs |
| Out-of-order handling | Watermarks and late policies | Naturally handled by rerunning |
| Reprocessing | Replay the log | Rerun the job |
| Operational burden | Always running | Fails between runs, not during |

The operational row understates the difference in practice: a failed batch job is rerun in the morning, while a failed streaming job is an incident with growing lag, which changes how the team must be organised around it.

> **Ask before choosing it**  
> Does the value of this answer decay in seconds? Fraud detection and live alerting genuinely need streaming; a dashboard someone opens twice a day does not, and choosing streaming for it buys continuous operational burden in exchange for freshness nobody observes.

**How it fails**

**How stream processing goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Aggregates silently wrong | Windows keyed on arrival time | Use event time with watermarks |
| Late events discarded unnoticed | Default drop policy | Side outputs and explicit handling |
| State grows without limit | No expiry on keyed state | TTL on keys; bound the keyspace |
| Slow recovery after a restart | Very large checkpointed state | Smaller state; incremental checkpoints |
| Duplicate results downstream | Non-idempotent sink | Transactional or idempotent writes |
| Lag growing unnoticed | No monitoring of consumer lag | Alert on lag per partition |
| Deployment causes an outage | State restore time underestimated | Plan deployments around recovery cost |
| Results never final | No watermark, continuous updates | Define when a window is closed |

> **Unbounded keyed state fails slowly and then all at once**  
> A pipeline keyed on something with growing cardinality — session identifiers, user agents, URLs — accumulates state that nothing removes, and the job runs correctly for weeks while memory climbs. When it finally exceeds capacity the restart must restore an enormous checkpoint, which takes far longer than usual, extending a memory problem into a prolonged outage. Expiry has to be designed in from the start rather than added when it becomes urgent.

**Where it is used**

- **Fraud and anomaly detection**, where a decision is worthless once the transaction completes.
- **Real-time analytics dashboards**, updating continuously rather than on a schedule.
- **Materialised view maintenance**, keeping a derived store current as events arrive.
- **Monitoring and alerting pipelines**, aggregating metrics into conditions within seconds.
- **Personalisation**, updating recommendations from behaviour as it happens.

**In the interview**

Event time against processing time is the distinguishing topic; everything else follows from it.

- **Separate event time from arrival time** and say that arrival-time windows are wrong whenever events are delayed.
- **Explain watermarks as the latency/completeness dial**, not as a technical detail.
- **State the late-event policy** you would choose and why, since the default is silent dropping.
- **Qualify exactly-once**: it covers pipeline state, and reaches the output only with a transactional or idempotent sink.
- **Raise state size and restart cost**, because they make deployments materially different from batch.

**Practice drill**

Detect more than five transactions per card in one minute, from a stream where mobile events can be delayed by up to ten minutes. Choose the window type, the time semantics and the watermark, and state the resulting alert latency. Define what happens to an event arriving twelve minutes late. Then estimate the state size for ten million cards and describe what a restart costs at that size.

**Go deeper**

Stream processing computes continuously over unbounded input, maintaining state so that results are available within seconds of the events that produced them.

**It exists for answers whose value decays quickly.** Fraud detection, live alerting and real-time personalisation are worth much less an hour later, and shortening a batch interval does not reach that latency — the floor is the interval plus the runtime, and per-run overheads dominate as the interval shrinks. Keeping the computation running removes the interval entirely, at the cost of a system that must be operated continuously.

**The distinction between event time and processing time is the core difficulty.** Events arrive late, out of order, and in bursts when devices reconnect, so grouping by arrival time produces aggregates that are quietly wrong. Event-time processing is correct and requires deciding how long to wait for stragglers — a watermark — which is a direct trade between how fresh results are and how complete they are, with no setting that optimises both.

**Late events need an explicit policy because the default is silent loss.** An event arriving after its window has closed is dropped unless something else is specified, which produces under-counts nobody notices. Collecting late events to a side output, or emitting corrections that downstream consumers must handle as retractions, are the alternatives — each with real cost, and each better than an unexamined default.

**State is what makes streaming powerful and what makes it operationally heavy.** Keyed state enables aggregation, joins and pattern detection, and it must be checkpointed durably so failures do not lose it. That makes restarts expensive in proportion to state size, so a deployment is not the trivial operation it is for a batch job, and a pipeline with very large state can take minutes to become healthy after any change.

**Unbounded state is the failure that arrives slowly.** Keying on something with growing cardinality accumulates entries that nothing removes, and the pipeline behaves perfectly while memory climbs over weeks. The eventual failure is worse than an ordinary one because recovery must restore an oversized checkpoint, turning a capacity problem into an extended outage — which is why expiry belongs in the initial design rather than in the incident review.

**Exactly-once is real but narrower than it sounds.** Frameworks advance state and input offsets together so that internal recovery does not double-count, which is a genuine and valuable guarantee. It extends to the outside world only when the sink is transactional or idempotent; otherwise a recovery re-emits results that were already applied. Stating the guarantee with its boundary is the difference between an accurate description and a marketing one, and it determines whether the consumer needs deduplication of its own.

**Related patterns:** Batch Processing · Partitioned Log · Materialized View · Change Data Capture

---

## Realtime Delivery

### WebSocket

*Hold a persistent bidirectional connection so either side can send at any time, with no polling and no per-message request overhead.*

> **When you hear…** Bidirectional messaging · sub-second updates in both directions · high message rates where per-request overhead matters

**Flow:** `HTTP upgrade` → `Connection persists` → `Either side sends` → `Low per-message cost` → `Reconnect on drop`

**The problem**

A chat application built on polling asks the server for new messages every two seconds. Most requests return nothing, each carries full request and response headers, and a message still takes up to two seconds to appear.

Reducing the interval makes the waste worse without making it fast. At half-second polling the empty-request rate quadruples and messages still arrive up to half a second late, while the server handles four times the load to deliver the same content.

> **Pay for the connection once instead of for every message**  
> A connection that stays open removes both problems at once: the server pushes the moment something happens, so there is no interval, and each message carries only a few bytes of framing rather than a full set of headers. The cost moves from per-request overhead to holding state for every connected client, which is a much better trade wherever messages are frequent or latency matters.

**Mental model**

An HTTP request that is upgraded into a long-lived duplex channel, after which both ends send framed messages independently.

1. **Upgrade** — An ordinary HTTP request negotiates the protocol switch.
2. **Persist** — The connection remains open, held by both ends.
3. **Exchange** — Either side sends messages at any time, with minimal framing.
4. **Keep alive** — Pings detect dead connections that would otherwise linger.
5. **Reconnect** — The client re-establishes and recovers whatever it missed.

> **Connections are state, and state is what makes scaling hard**  
> Each connected client occupies memory, a file descriptor and a place in the server's connection table, so a server holds a bounded number of them and a deploy disconnects all of them at once. Every property that makes the protocol efficient per message makes the fleet harder to operate, and the difficulties are about connection management rather than message delivery.

**How it works**

**Connection cost and reconnection**

```text
PER-MESSAGE COST
  polling:    ~500-800 B of headers per request,
              both ways, plus TCP and TLS work
  websocket:  2-14 B of framing per message
  -> at high message rates the difference is orders
     of magnitude

PER-CONNECTION COST
  ~10-50 KB of memory per connection
  1 socket, 1 file descriptor
  100,000 connections -> 1-5 GB, plus tuned limits
  -> connection count, not message rate, is usually
     the binding constraint

RECONNECTION IS THE HARD PART
  networks drop connections constantly: mobile
  handovers, proxies, timeouts, deploys

  client must:
    reconnect with exponential backoff and JITTER
      without jitter, a deploy reconnects everyone
      simultaneously and the fleet is overwhelmed
    resume from a known position
      send the last received message id; the server
      replays what was missed
    handle duplicates on resume

HEARTBEATS
  ping every 30 s, expect a pong
  -> detects half-open connections that TCP will not
  -> also keeps intermediaries from idling the
     connection out
```

1. **Always send heartbeats** — Half-open connections look alive and deliver nothing; only pings reveal them.
2. **Reconnect with backoff and jitter** — Every client reconnecting at once after a deploy is a self-inflicted overload.
3. **Support resumption from a message position** — Without it, every reconnect loses whatever arrived during the gap.
4. **Plan for connection count, not message rate** — Memory and descriptors bound the fleet long before throughput does.
5. **Externalise routing state** — Which server holds which client must be shared, or messages cannot be delivered.
6. **Drain connections gradually on deploy** — Disconnecting everyone simultaneously produces a reconnection storm.

**Routing messages to the right server**

```text
THE PROBLEM
  user A is connected to server 3
  user B on server 7 sends A a message
  -> server 7 has no connection to A

OPTIONS
  1  SHARED PUB/SUB
       every server subscribes to channels for its
       connected users
       sender publishes; the holding server delivers
       -> simple, scales well, one extra hop

  2  CONNECTION REGISTRY
       a store maps user -> server
       sender looks up and forwards directly
       -> fewer messages, registry must be accurate
          and fast

  3  BROADCAST TO ALL SERVERS
       each checks whether it holds the user
       -> only viable with few servers

DEPLOYS
  rolling restart disconnects every client on each
  instance
  -> stagger, and drain slowly
  -> clients reconnect with jitter
  -> otherwise every deploy is a traffic spike

SCALING SHAPE
  connections spread across many instances
  -> adding capacity means more instances, and
     existing connections do not move
```

| Metric | Value | Note |
|---|---|---|
| Per message | 2-14 B | **vs ~600 B** |
| Per connection | 10-50 KB | the real limit |
| Hard part | reconnection | not messaging |
| Deploys | drain slowly | or storm |

> **Reconnection without jitter turns every deploy into a self-inflicted outage**  
> When a server restarts, every client it held reconnects — and if they all use the same backoff schedule they arrive together, overwhelming the remaining instances and causing more disconnections. The storm is entirely self-sustaining. Randomised backoff spreads the return across seconds or minutes and is the single most important client-side behaviour in any persistent-connection system.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Server implementations | Event-loop servers and frameworks | Connection-per-thread models do not scale here |
| Managed services | Hosted WebSocket gateways | Connection management as a service |
| Message routing | Redis pub/sub, a broker | Connects servers that hold different clients |
| Load balancers | Must support upgrade and long-lived connections | Idle timeouts are a common trap |
| Server-sent events | One-way alternative | Far simpler when only the server sends |
| Long polling | Fallback | Works through hostile intermediaries |

Load balancer idle timeouts cause a distinctive failure: connections silently closed after a fixed period of inactivity, which looks like an application bug and is configuration. Heartbeats at an interval shorter than that timeout prevent it, which is one more reason they are not optional.

**Trade-offs**

**Real-time delivery mechanisms**

| Mechanism | Direction | Overhead per message | Complexity |
|---|---|---|---|
| Polling | Client pulls | Full headers | Lowest |
| Long polling | Server pushes on a held request | Full headers per message | Low |
| Server-sent events | Server to client only | Minimal | Low |
| WebSocket | Bidirectional | Minimal | High |
| WebRTC data channels | Peer to peer | Minimal | Highest |

Server-sent events deserve consideration before WebSockets in most designs: if only the server sends, they provide the same push latency over ordinary HTTP, with automatic reconnection and resumption built into the protocol rather than implemented by hand.

> **Ask before choosing it**  
> Does the client actually need to send over the same channel? Notifications, live prices and progress updates are one-directional, and server-sent events deliver them with considerably less operational weight — the bidirectional capability is worth its cost only when it is genuinely used.

**How it fails**

**How WebSocket systems go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Reconnection storm after a deploy | Synchronised backoff | Jittered exponential backoff |
| Silent delivery failure | Half-open connection undetected | Heartbeats with a pong timeout |
| Connections dropped periodically | Load balancer idle timeout | Ping more often than the timeout |
| Messages lost across a reconnect | No resumption mechanism | Client sends its last received id; server replays |
| Server memory exhausted | Connection count exceeding capacity | Capacity planning by connections, not requests |
| Messages not delivered across servers | No cross-instance routing | Shared pub/sub or a connection registry |
| Backpressure ignored | Writing faster than the client reads | Bound the per-connection send buffer; disconnect slow clients |

> **A client that reads slower than the server writes will exhaust the server's memory**  
> Messages queued for a connection accumulate in a send buffer, and a client on a poor connection or one that has stopped reading lets that buffer grow without limit. A few thousand such clients consume the server's memory even though nothing appears to be failing. Bounding the buffer and disconnecting clients that exceed it is the only stable policy, and it must be in place before it is needed.

**Where it is used**

- **Chat and messaging platforms**, the archetypal bidirectional real-time use.
- **Collaborative editors**, where both sides send continuously and latency is felt directly.
- **Trading and live-price interfaces**, needing sub-second updates with client actions on the same channel.
- **Multiplayer games**, exchanging state in both directions many times per second.
- **Live dashboards with controls**, where the client both receives updates and issues commands.

**In the interview**

Spend the time on connection management; the protocol itself is the easy half.

- **Contrast per-message and per-connection costs** with numbers to show what the protocol actually changes.
- **Say reconnection is the hard part**, and require jittered backoff plus resumption from a position.
- **Insist on heartbeats**, because half-open connections appear healthy while delivering nothing.
- **Solve cross-server routing explicitly** with pub/sub or a registry; it is the question most designs omit.
- **Offer server-sent events** when the flow is one-directional, since the operational saving is substantial.

**Practice drill**

Design the connection layer for a chat system with 500,000 concurrent users across multiple servers. Compute the memory required and the number of instances. Describe how a message reaches a user connected to a different server. Then walk through a rolling deploy: what happens to the connections, what clients do, and exactly what prevents the reconnection from overwhelming the remaining instances.

**Go deeper**

A WebSocket is a persistent bidirectional connection established by upgrading an HTTP request, allowing either side to send at any time.

**It changes where the cost sits rather than removing it.** Polling pays full request overhead for every check, most of which return nothing; a persistent connection pays almost nothing per message and instead holds memory, a socket and a descriptor for every connected client. That trade is strongly favourable when messages are frequent or latency matters, and unfavourable for a client that needs an update every few minutes.

**Connection count, not message throughput, is the binding constraint.** Tens of kilobytes per connection means a server holds tens or hundreds of thousands, and capacity planning is about how many clients can be connected rather than how many messages flow. This reverses the usual instinct: a system can be entirely idle in message terms and still be at its limit.

**Reconnection is where the real engineering lives.** Networks drop connections constantly — mobile handovers, proxy timeouts, deploys — so a persistent-connection system spends most of its complexity on re-establishing rather than on messaging. Jittered backoff prevents clients returning in unison, and resumption from a known message position prevents every reconnect from silently losing whatever arrived during the gap. Neither is optional at scale.

**Half-open connections are invisible without heartbeats.** A connection whose peer has vanished can remain open indefinitely from the server's perspective, consuming resources and appearing healthy while delivering nothing. Application-level pings are what reveal this, and they simultaneously prevent intermediaries from closing idle connections — which is a distinct failure that presents identically and is entirely a configuration problem.

**Cross-server routing is the design question most omissions leave out.** With clients spread across many instances, a message from one user to another must reach whichever server holds the recipient's connection, which requires either a shared pub/sub layer or a registry mapping users to servers. Neither is difficult, and a design that describes the protocol without addressing this has not yet described a system that works with more than one server.

**Slow clients threaten the server's memory, not their own experience.** Messages queued for a connection accumulate in a send buffer, and a client that reads slowly lets that buffer grow while nothing appears to be wrong. Bounding it and disconnecting clients that exceed the bound is the only stable policy — and because the symptom is server memory growth rather than a delivery error, it is usually discovered during an incident rather than during design.

**Related patterns:** Server Sent Events · Long Polling · Work Queue · Backpressure

---

### Long Polling

*Hold a request open until data is available or a timeout expires, giving push-like latency using only ordinary HTTP requests.*

> **When you hear…** Push needed through restrictive networks · infrastructure that only understands request-response · a fallback for persistent connections

**Flow:** `Client requests` → `Server holds it` → `Data arrives` → `Respond immediately` → `Client requests again`

**The problem**

Push is needed, but the environment does not cooperate: a corporate proxy blocks protocol upgrades, an old load balancer terminates long-lived connections, or the client is a system that only knows how to make requests.

Regular polling is the fallback everyone reaches for, and it forces a choice between latency and waste — short intervals produce mostly empty responses, long intervals produce stale data, and neither is satisfactory.

> **A request that waits is a push in disguise**  
> Instead of the server answering immediately with nothing, it holds the request until there is something to say. The client still made an ordinary request and receives an ordinary response, so every proxy, gateway and firewall treats it as normal traffic — but the data arrives the instant it exists, which is what push latency means.

**Mental model**

A request/response cycle where the response is deferred. The client asks, the server waits, and the loop repeats immediately after each answer.

1. **Request** — The client asks for anything newer than what it already has.
2. **Wait** — The server holds the request rather than answering emptily.
3. **Deliver** — It responds as soon as data appears.
4. **Time out** — If nothing arrives within the limit, it responds empty.
5. **Repeat** — The client immediately issues the next request.

> **The gap between responses is a window in which events can be missed**  
> Between receiving a response and issuing the next request, the client is not connected. Anything published in that window is lost unless the server tracks per-client position and the client tells it where it left off — which makes cursors part of the protocol rather than an enhancement.

**How it works**

**The loop and the gap**

```text
CLIENT
  loop:
    GET /poll?since=<cursor>
    on response:
      process events
      advance cursor
    immediately repeat

SERVER
  on request:
    if events newer than cursor exist:
      return them now
    else:
      hold the request
      wait up to 30 s for a new event
      return it, or return empty at the timeout

THE GAP
  response sent  -----> client processes ----->
  next request arrives
  events published in this window are not delivered
  by that connection
  -> the CURSOR is what recovers them: the next
     request asks for everything since the last
     known position
  -> without a cursor, long polling loses messages
     routinely

TIMEOUT CHOICE
  too long   intermediaries kill the connection
  too short  request overhead dominates
  20-30 s is the usual compromise, chosen to sit
  below typical proxy idle limits

HOLDING COSTS A CONNECTION
  same resource profile as a persistent connection
  plus a full request cycle per message
```

1. **Make cursors part of the protocol** — The gap between requests loses events without them.
2. **Choose a timeout below proxy idle limits** — Otherwise intermediaries close the request and the client sees an error.
3. **Do not block a thread per held request** — Asynchronous handling is required; a thread per waiting client does not scale.
4. **Return immediately when data already exists** — Waiting when there is something to say adds pointless latency.
5. **Back off after errors, with jitter** — Reconnection storms apply here exactly as to any persistent mechanism.
6. **Treat it as a fallback** — Where WebSockets or event streams work, they are better.

**Cost comparison**

```text
REGULAR POLLING, every 2 s
  1,800 requests/hour per client
  almost all empty
  latency: up to 2 s

LONG POLLING, 30 s timeout, 1 message/minute
  ~60 requests/hour per client (one per message,
  plus timeouts)
  latency: near zero
  -> 30x fewer requests AND better latency

LONG POLLING, high message rate (10/s)
  each message ends a request and starts a new one
  -> 36,000 requests/hour per client
  -> full request overhead per message
  -> WORSE than polling in efficiency terms
  -> this is where a persistent connection wins
     decisively

SERVER-SIDE
  held requests occupy connections just like
  WebSockets
  -> must be handled asynchronously
  -> a thread per held request exhausts the pool at
     a few thousand clients
```

| Metric | Value | Note |
|---|---|---|
| Latency | near zero | like push |
| Low rate | 30× fewer requests | **than polling** |
| High rate | worse than polling | overhead per message |
| Requirement | async handling | not a thread each |

> **A thread held per waiting request exhausts the server at a few thousand clients**  
> The mechanism depends on keeping many requests open simultaneously, and a server that dedicates a thread to each one runs out at a few thousand — far below the connection count the same hardware could otherwise sustain. Asynchronous request handling, where a waiting request consumes only its connection, is a prerequisite rather than an optimisation.

**Technologies**

| Context | Use | Note |
|---|---|---|
| Fallback layers | When upgrades are blocked | Libraries often degrade to this automatically |
| Async servers | Holding many requests cheaply | Essential; synchronous models do not work |
| API-based notifications | Systems that only make requests | No client library needed |
| WebSocket | The preferred alternative | Better for bidirectional or high-rate traffic |
| Server-sent events | The preferred one-directional alternative | Similar simplicity, no gap between responses |
| Regular polling | The simpler fallback | Predictable, wasteful, higher latency |

Real-time libraries commonly implement a negotiation that prefers a persistent connection and falls back to long polling where it fails, which is why the pattern remains widely deployed even in systems whose engineers never chose it explicitly.

**Trade-offs**

**Delivery mechanisms compared**

| Mechanism | Latency | Overhead at low rates | Overhead at high rates |
|---|---|---|---|
| Polling | Up to the interval | High | Moderate |
| Long polling | Near zero | Low | High |
| Server-sent events | Near zero | Low | Low |
| WebSocket | Near zero | Low | Lowest |

The two right-hand columns explain where long polling belongs: excellent for infrequent updates, poor for frequent ones, because every message costs a complete request cycle. Message rate, more than anything else, decides whether it is the right fallback or the wrong choice.

> **Ask before choosing it**  
> Why can a persistent connection not be used here? If the answer is a genuine infrastructure constraint, long polling is the correct response; if it is unfamiliarity with WebSockets or event streams, the constraint is worth removing instead.

**How it fails**

**How long polling goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Events lost intermittently | No cursor across the gap | Client sends its position; server replays from it |
| Requests killed by intermediaries | Timeout longer than proxy idle limits | Hold for 20-30 seconds at most |
| Server exhausted at low client counts | A thread per held request | Asynchronous request handling |
| Overhead worse than polling | High message rate | Switch to a persistent connection |
| Reconnection storm | All clients retrying together after an error | Jittered backoff |
| Duplicate processing | Cursor not advanced atomically with handling | Advance only after successful processing |
| Latency higher than expected | Server waiting even when data exists | Return immediately if there is anything to send |

> **Without a cursor, the gap between responses silently drops messages**  
> The client is disconnected between receiving a response and sending the next request, and anything published during that interval is not delivered. Because the window is short and the loss is intermittent, it typically presents as occasional missing notifications that nobody can reproduce — and the fix is not a shorter gap, which cannot be eliminated, but a position the next request carries so the server can replay what was missed.

**Where it is used**

- **Real-time libraries**, which negotiate down to long polling where upgrades are blocked.
- **Corporate and restricted networks**, where proxies interfere with anything unusual.
- **API-based notification delivery**, for clients that can only make requests.
- **Legacy infrastructure**, where load balancers do not support persistent connections.
- **Simple in-house implementations**, where the whole mechanism is a loop and a cursor.

**In the interview**

Present it as the compatibility fallback, and be precise about the gap and the message-rate limit.

- **Explain the deferred response** and why it produces push latency over ordinary request-response.
- **Name the gap between responses** and require cursors, since that is where messages are lost.
- **Give the message-rate crossover**: excellent at low rates, worse than polling at high ones.
- **Require asynchronous handling**, because a thread per held request exhausts the server early.
- **Choose a timeout below proxy idle limits**, which is why 20 to 30 seconds is conventional.

**Practice drill**

Deliver notifications to 10,000 clients behind a corporate proxy that blocks protocol upgrades. Design the long-polling loop on both sides, including the cursor, the timeout and the error backoff. Compute the request volume at one message per minute per client and at ten per second, and say at which point you would insist on a different mechanism. Then describe precisely how a message published during the gap still reaches the client.

**Go deeper**

Long polling delivers push-like latency using ordinary HTTP by holding each request open until there is something to return.

**Its value is compatibility rather than efficiency.** Every proxy, gateway and firewall understands a request and a response, so the mechanism works in environments that block protocol upgrades or terminate long-lived connections. That is the entire reason it persists alongside better-performing alternatives — it works where they do not, which for some client populations is decisive.

**The gap between responses is a structural property, not an implementation flaw.** After the server answers, the client is disconnected until it issues the next request, and anything published in that interval belongs to no connection. Cursors are therefore part of the protocol: the client states what it last received and the server replays from there. Implementations that omit this lose messages intermittently, in a way that is very difficult to reproduce from a report.

**Message rate determines whether it is efficient or wasteful.** At one message a minute it produces far fewer requests than interval polling and delivers with near-zero latency, which is an unambiguous improvement. At ten messages a second, each one terminates a request and begins another, so the full HTTP overhead is paid per message and the mechanism performs worse than the polling it replaced. The crossover is the main thing to reason about when choosing it.

**Server-side it costs as much as a persistent connection, plus request overhead.** Held requests occupy connections exactly as WebSockets do, so capacity planning is the same — and the model only works at all with asynchronous request handling, because dedicating a thread to each waiting client exhausts the pool at a small fraction of the achievable connection count. Synchronous frameworks fail at this early and for reasons that look like a capacity problem rather than an architectural one.

**Timeout selection is bounded by infrastructure rather than by preference.** Holding a request longer reduces overhead and increases the chance an intermediary closes it, at which point the client sees an error rather than an empty response. Twenty to thirty seconds is conventional because it sits comfortably below common proxy idle limits, and the value is really a statement about the network the clients are on rather than about the application.

**It should be chosen as a fallback, not as a default.** Where persistent connections work, both WebSockets and event streams deliver the same latency with less overhead and no gap to compensate for. The right reason to implement long polling is a specific environment that rejects the alternatives — and the right implementation is usually a negotiated degradation, so that clients which can do better are not held to the weakest option.

**Related patterns:** WebSocket · Server Sent Events · Retry With Jitter · Work Queue

---

### Server Sent Events

*Stream updates from server to client over one long-lived HTTP response, with reconnection and resumption handled by the browser.*

> **When you hear…** One-directional server updates · push without the weight of WebSockets · standard HTTP infrastructure preferred

**Flow:** `Client opens stream` → `Server holds response` → `Events pushed as text` → `Browser auto-reconnects` → `Last-Event-ID resumes`

**The problem**

A dashboard needs live updates from the server. The client never sends anything over that channel — it only receives — and adopting a bidirectional protocol brings connection management, reconnection logic and a separate infrastructure path for a capability that will not be used.

Polling remains wasteful, and the reconnection logic that any persistent connection requires is exactly the code that most implementations get wrong.

> **For one-directional push, the existing protocol already suffices**  
> An HTTP response that is never closed is a stream, and the browser's built-in client handles reconnection and resumption automatically. The push latency matches any other persistent connection, everything in the HTTP path continues to work unchanged, and the reconnection code that is the main source of bugs elsewhere is provided by the platform rather than written by hand.

**Mental model**

A normal HTTP GET whose response never ends, delivering a stream of text events that the browser parses and dispatches.

1. **Open** — The client requests a stream endpoint like any other resource.
2. **Hold** — The server keeps the response open.
3. **Send** — Events are written as plain text with optional identifiers.
4. **Reconnect** — The browser reconnects automatically if the connection drops.
5. **Resume** — It sends the last event identifier so the server can replay what was missed.

> **It is one-directional; anything the client sends is a separate request**  
> Client-to-server communication requires an ordinary HTTP request, so an interactive feature involves two mechanisms with independent lifecycles. That is perfectly reasonable when client sends are occasional, and it becomes awkward when the interaction is genuinely conversational — at which point a bidirectional protocol is the better fit.

**How it works**

**The wire format and what it provides**

```text
REQUEST
  GET /events
  Accept: text/event-stream

RESPONSE  (never closed)
  Content-Type: text/event-stream
  Cache-Control: no-cache

  id: 1042
  event: price
  data: {"symbol":"ABC","price":41.2}

  id: 1043
  event: price
  data: {"symbol":"XYZ","price":9.8}

  : this is a comment, used as a keepalive

BUILT IN, NO CODE REQUIRED
  automatic reconnection on drop
  Last-Event-ID header sent on reconnect
    -> the server replays from that point
  retry: <ms> to control the backoff
  -> the two things WebSocket clients must implement
     by hand are provided here

SERVER OBLIGATIONS
  replay from Last-Event-ID
    requires a buffer of recent events per stream
  send periodic comments as keepalives
    prevents proxies idling the connection out
  disable response buffering
    a buffering proxy holds events until the buffer
    fills, which destroys the latency benefit
```

1. **Assign an identifier to every event** — Resumption depends entirely on it.
2. **Keep a replay buffer of recent events** — The browser will ask to resume; the server must be able to.
3. **Send periodic comment lines as keepalives** — Intermediaries close connections that appear idle.
4. **Disable buffering in every proxy on the path** — A buffering proxy silently converts streaming into batching.
5. **Use the retry field to control reconnection** — It is the only backoff control the protocol offers.
6. **Account for browser connection limits** — Older HTTP/1.1 limits constrain how many streams one origin can hold.

**Where it fits, and its limits**

```text
GOOD FIT
  notifications and alerts
  live dashboards and metrics
  progress updates for long operations
  price and score feeds
  activity streams
  -> all one-directional

POOR FIT
  chat and collaboration     bidirectional
  binary payloads            text-only format
  very high message rates    text framing overhead
  low-latency gaming         wrong tool entirely

HTTP/1.1 CONNECTION LIMIT
  browsers allow ~6 connections per origin
  each open stream consumes one
  -> a few streams starve ordinary requests
  -> HTTP/2 multiplexes, and removes this entirely
  -> with HTTP/1.1, use ONE stream and multiplex
     event types over it

SERVER COST
  similar to WebSocket: one held connection per
  client
  -> same capacity planning, same connection limits
  -> the saving is in client code and infrastructure
     compatibility, not in server resources
```

| Metric | Value | Note |
|---|---|---|
| Direction | server to client | **one way** |
| Reconnection | built in | no client code |
| Resumption | Last-Event-ID | server must buffer |
| Server cost | same as WebSocket | one connection each |

> **Any buffering proxy on the path converts a stream into a batch**  
> Reverse proxies frequently buffer responses by default, which means events accumulate until the buffer fills or the connection closes rather than being delivered as they are written. The feature appears to work in development and behaves as slow batching in production, with no error anywhere — and it is one of the most common reasons a correctly implemented stream feels broken.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Client | Native EventSource | Reconnection and resumption included |
| Server | Any HTTP server that supports streaming | Response buffering must be off |
| Proxies | Must be configured not to buffer | The most common deployment problem |
| HTTP/2 | Multiplexed streams | Removes the per-origin connection limit |
| WebSocket | Bidirectional alternative | More capability, more client code |
| Long polling | Fallback | Works where streaming is blocked entirely |

HTTP/2 materially changes the calculus: without the six-connection limit, several independent streams per origin become practical, and the main historical argument for choosing WebSockets over this for one-directional data disappears.

**Trade-offs**

**Server-sent events against WebSocket**

| Property | SSE | WebSocket |
|---|---|---|
| Direction | Server to client | Bidirectional |
| Client reconnection | Automatic | Implemented by hand |
| Resumption | Built into the protocol | Implemented by hand |
| Payload | Text only | Text or binary |
| Infrastructure compatibility | Ordinary HTTP | Requires upgrade support |
| Server connection cost | One per client | One per client |

The two reconnection rows are the practical argument. Hand-written reconnection with backoff and resumption is where persistent-connection bugs concentrate, and the protocol providing it correctly removes an entire class of defects.

> **Ask before choosing it**  
> Does the client need to send on this channel, and is the payload binary? Two noes make this the simpler choice; either yes points to WebSockets, and hedging by adopting the heavier protocol for a one-directional feature is a common over-engineering.

**How it fails**

**How server-sent events go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Events arrive in batches | A proxy buffering the response | Disable buffering on every hop |
| Connection dropped every minute | Proxy idle timeout | Send keepalive comments regularly |
| Events lost across reconnects | No identifiers or no replay buffer | Assign ids; buffer recent events |
| Other requests to the origin stall | HTTP/1.1 connection limit consumed | One multiplexed stream, or HTTP/2 |
| Reconnection storm | Default backoff identical for all clients | Set retry with jitter server-side |
| Server memory exhausted | Connection count underestimated | Plan capacity by connections |
| Binary data corrupted | Encoding binary into a text protocol | Use WebSocket, or encode explicitly |

> **Resumption is only as good as the server's replay buffer**  
> The browser faithfully sends the last event identifier it saw, and a server that keeps no history simply resumes from the present — so the client silently misses everything that occurred during the disconnection. The protocol provides the mechanism and the server must provide the data, and implementations that ignore the header have automatic reconnection without automatic recovery, which is easy to mistake for correctness.

**Where it is used**

- **Live dashboards and monitoring**, where updates flow one way and payloads are small.
- **Notification delivery**, pushing alerts to open browser sessions.
- **Progress reporting**, streaming the status of long-running operations.
- **Price and score feeds**, updating continuously with no client input.
- **Streaming responses from language models**, delivering tokens as they are produced.

**In the interview**

Position it as the simpler choice for one-directional push, and name the deployment traps.

- **Say the protocol provides reconnection and resumption**, which are the two things WebSocket clients implement by hand and get wrong.
- **Note that the server must maintain a replay buffer** for Last-Event-ID to mean anything.
- **Raise proxy buffering** as the deployment failure that turns streaming into batching with no error.
- **Mention the HTTP/1.1 connection limit** and that HTTP/2 removes it.
- **Be clear that server cost is the same** — one held connection per client — so the saving is in complexity, not resources.

**Practice drill**

A dashboard needs live metric updates for 50,000 concurrent users. Design the stream: the event format, identifiers, keepalives and the replay buffer. State what the client does on disconnection and what the server must do to honour it. Then list every place a proxy could break the stream and how you would verify, before launch, that none of them does.

**Go deeper**

Server-sent events deliver a stream of updates over one long-lived HTTP response, with reconnection and resumption defined by the protocol.

**It fits the one-directional case exactly, which is most real-time features.** Notifications, dashboards, progress reports and price feeds flow only from server to client, and adopting a bidirectional protocol for them brings connection management and client-side reconnection logic for a capability that goes unused. Matching the mechanism to the actual direction of data removes work rather than capability.

**The protocol provides precisely the parts that are usually implemented badly.** Automatic reconnection and resumption via a last-event identifier are exactly what hand-written persistent-connection clients get wrong — missing jitter, losing messages across gaps, reconnecting in unison. Having them specified and implemented by the browser eliminates a class of bugs that otherwise has to be rediscovered in every project.

**Resumption is a shared responsibility that is half-implemented surprisingly often.** The client sends the identifier of the last event it received, and the server must maintain enough recent history to replay from that point. A server ignoring the header still appears to work — the connection re-establishes and events resume — while silently dropping everything that occurred during the gap, which is easy to mistake for correct behaviour until someone notices missing data.

**Its most common deployment failure is invisible and infrastructural.** Reverse proxies that buffer responses accumulate events instead of forwarding them, converting a stream into periodic batches with no error reported anywhere. The feature works in development, where no such proxy exists, and feels sluggish in production for reasons that look like application behaviour — so verifying the full path before launch is part of implementing the pattern.

**Server resource cost is the same as any persistent connection.** One held connection per client means the same memory, descriptor and capacity-planning considerations as WebSockets, so the saving is in client code and infrastructure compatibility rather than in server load. Describing it as the lightweight option is accurate about complexity and misleading about capacity.

**Its historical limitation has largely disappeared.** The six-connection-per-origin limit in HTTP/1.1 meant open streams could starve ordinary requests, which pushed many teams towards WebSockets for reasons unrelated to direction. With HTTP/2 multiplexing, that constraint is gone, and the choice reduces to the genuine question — whether the client needs to send on the same channel, and whether the payload is binary.

**Related patterns:** WebSocket · Long Polling · CDN · Backpressure

---

## Media Delivery

### CDN

*Serve content from caches near users, so most requests never reach the origin and distance stops dominating latency.*

> **When you hear…** Users far from the origin · static assets and media dominating traffic · origin bandwidth or load becoming the constraint

**Flow:** `User requests` → `Nearest edge` → `Cache hit served` → `Miss fetches origin` → `Cached for the next user`

**The problem**

A two-megabyte image served from a single region takes hundreds of milliseconds to reach a user on another continent before a single byte of content is transferred, and every user pays that cost independently for identical bytes.

The origin also pays. Serving the same asset a million times consumes bandwidth and connections that could be doing work only the origin can do, and a traffic spike on static content can exhaust capacity needed for dynamic requests.

> **Identical bytes requested by many people should be stored near them**  
> Content that does not vary per user can be copied to locations close to users, where it is served without ever contacting the origin. Latency falls because the distance falls, and origin load falls because most requests terminate at the edge — both from the same change, which is why this is usually the single highest-leverage performance improvement available.

**Mental model**

A global network of caches. A request goes to the nearest one; if it holds the content it answers, otherwise it fetches from the origin and keeps a copy.

1. **Route** — DNS or anycast directs the user to a nearby edge.
2. **Check** — The edge looks for a valid cached copy.
3. **Serve or fetch** — A hit is answered locally; a miss goes to the origin.
4. **Store** — The response is cached according to its headers.
5. **Expire** — Content ages out, is revalidated, or is purged explicitly.

> **Cached content is difficult to recall once distributed**  
> A wrong or sensitive response cached at hundreds of edges is present in hundreds of places, and purging is neither instant nor guaranteed to be complete. This makes cache-control headers a correctness concern rather than a tuning detail: caching something user-specific by accident can serve one user's data to another, and the mistake propagates globally within seconds.

**How it works**

**Cache keys, TTLs and invalidation**

```text
CACHE KEY
  by default: method + host + path (+ query)
  must also vary on anything that changes the
  response:
    Accept-Encoding   (gzip vs brotli)
    device class      (if responses differ)
  -> every extra dimension multiplies the number of
     cached objects and lowers the hit rate

NEVER CACHE BY DEFAULT
  anything varying by user
  anything behind authentication
  -> Cache-Control: private, no-store
  -> a single misconfiguration here leaks data
     between users

TTL STRATEGY
  immutable assets (hashed filenames)
    Cache-Control: public, max-age=31536000, immutable
    -> cache forever; a new version has a new name
  html and frequently changing content
    short max-age + stale-while-revalidate
    -> fresh enough, and always fast
  api responses
    usually private, or very short TTLs

INVALIDATION
  purge         explicit, propagates in seconds to
                minutes, not instant
  versioned url best method: a new name is a new
                object, so nothing needs purging
  short ttl     simplest; content is briefly stale

-> versioned URLs remove the invalidation problem
   entirely and should be the default for assets
```

1. **Use content-hashed filenames for static assets** — Immutable objects can be cached forever and never need purging.
2. **Set cache headers explicitly on every response** — Defaults differ between providers, and the wrong default leaks data.
3. **Keep the cache key as narrow as correctness allows** — Each variation dimension fragments the cache and lowers hit rates.
4. **Use stale-while-revalidate for changing content** — Users get an instant response while the edge refreshes behind them.
5. **Enable origin shielding** — A designated tier absorbs edge misses so the origin sees far fewer.
6. **Monitor hit rate by content type** — An aggregate hit rate hides the specific thing that stopped caching.

**What a CDN actually saves**

```text
LATENCY
  origin in one region, user on another continent
    RTT ~250 ms before any data moves
  nearest edge
    RTT ~10-30 ms
  -> for a multi-request page, this compounds

ORIGIN LOAD
  1,000,000 requests for one asset
  90% hit rate  -> 100,000 reach the origin
  99% hit rate  -> 10,000
  99.9%         -> 1,000
  -> the last percent of hit rate matters more than
     the first ninety

BANDWIDTH
  origin egress is usually the most expensive
  bandwidth you buy
  -> moving it to the edge is often the single
     largest cost saving available

THUNDERING HERD ON A MISS
  a popular object expires
  many edges miss simultaneously
  -> the origin receives a burst
  -> mitigations: request collapsing at the edge,
     origin shielding, staggered TTLs

WHAT IT DOES NOT HELP
  personalised responses
  write requests
  anything requiring the origin's state
```

| Metric | Value | Note |
|---|---|---|
| Latency | 250 ms → 20 ms | **by distance** |
| Origin load | hit rate dependent | 99% → 1% remains |
| Best practice | hashed filenames | no purging needed |
| Risk | caching private data | propagates globally |

> **Caching an authenticated response serves one user's data to everyone**  
> A response that varies by user, cached without a private directive or a user-specific key, is stored at an edge and returned to whoever asks next. The failure is immediate, global and difficult to detect from the inside — the application behaves correctly and the cache is doing exactly what it was told. Explicit cache-control on every authenticated path is what prevents it, and it belongs in the framework rather than in individual handlers.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Global CDN providers | Managed edge networks | The standard choice; differ in features and pricing |
| Origin shielding | An intermediate cache tier | Collapses edge misses before the origin |
| Edge compute | Functions running at the edge | Personalisation without reaching the origin |
| Image optimisation at the edge | Format and size negotiation | Reduces bytes as well as distance |
| Browser cache | The layer before the CDN | Free, and frequently under-used |
| Object storage with a CDN | Standard static hosting | Cheap origin, cached everywhere |

The browser cache deserves mention because it is the only layer that eliminates the request entirely. Long-lived caching of hashed assets means a returning user makes no request at all, which beats even the fastest edge.

**Trade-offs**

**Serving strategies**

| Approach | Latency | Origin load | Freshness |
|---|---|---|---|
| Origin only | Distance-bound | Full | Always current |
| CDN with short TTL | Low | Reduced | Seconds stale |
| CDN with long TTL | Low | Minimal | Until expiry or purge |
| CDN with versioned URLs | Low | Minimal | Always current |
| Stale-while-revalidate | Low | Minimal | Briefly stale, self-refreshing |

Versioned URLs are the strongest row: because a change produces a new name, content can be cached indefinitely while remaining always current, which is why hashed asset filenames are effectively universal in modern build tooling.

> **Ask before choosing it**  
> What proportion of bytes served are identical for every user? If most traffic is static assets and media, a CDN is the highest-leverage change available; if most is personalised, the benefit is confined to the shared remainder and edge compute becomes the more relevant tool.

**How it fails**

**How CDNs go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Private data served to other users | Authenticated response cached publicly | Explicit private or no-store on those paths |
| Stale content after a deploy | Long TTL with no versioning | Hashed filenames, or purge on release |
| Poor hit rate | Cache key varying on too many dimensions | Normalise the key; strip irrelevant query parameters |
| Origin overwhelmed after expiry | Many edges missing at once | Request collapsing, shielding, staggered TTLs |
| Purge not taking effect | Propagation delay assumed instant | Version URLs instead of relying on purges |
| Everything bypassing the cache | Cookies or headers defeating caching | Strip unnecessary cookies on asset paths |
| Errors cached | Error responses inheriting a long TTL | Short or zero TTL on non-200 responses |

> **Caching an error response distributes an outage that has already ended**  
> If a failing origin returns a 500 with a cacheable header, edges store it and continue serving it long after the origin recovers, so a brief incident becomes a prolonged one that looks unrelated to any current fault. Error responses need explicit short or zero lifetimes, and the mistake is easy to make because the caching configuration was written with successful responses in mind.

**Where it is used**

- **Static asset delivery**, the universal use — scripts, styles, images and fonts.
- **Video and large media**, where bandwidth savings dominate the economics.
- **Software and package distribution**, serving identical artefacts globally.
- **API response caching** for public, non-personalised endpoints.
- **Edge compute deployments**, personalising at the edge rather than at the origin.

**In the interview**

Cover both benefits — latency and origin offload — and treat cache-control as a correctness topic.

- **Quantify both effects**: distance-bound latency falling to tens of milliseconds, and origin load falling with the hit rate.
- **Propose content-hashed filenames** so assets are immutable and invalidation never arises.
- **Treat caching of authenticated responses as a data-leak risk**, not a configuration preference.
- **Raise the thundering herd on expiry** and give shielding or request collapsing as the mitigation.
- **Note that error responses must not inherit long lifetimes**, since a cached failure outlives the incident.

**Practice drill**

A site serves 2 MB of static assets per page view to users worldwide from one region. Design the CDN configuration: cache keys, TTLs, filename strategy and invalidation approach. Estimate the origin load at 90%, 99% and 99.9% hit rates for a million requests. Then identify every response on the site that must not be cached, and describe the mechanism that guarantees it.

**Go deeper**

A CDN serves content from caches close to users, reducing latency by shortening distance and reducing origin load by terminating most requests at the edge.

**It delivers two benefits from one change, which is unusual.** Moving bytes closer cuts the round trip from hundreds of milliseconds to tens, and every request answered at the edge is one the origin never sees. For a site whose traffic is dominated by identical assets, this is typically the largest single performance and cost improvement available, and it requires no application changes beyond correct headers.

**Immutability is what makes caching simple.** Content-hashed filenames mean a changed file is a different object, so assets can be cached indefinitely and still always be current — invalidation never arises because nothing is ever updated in place. This removes the hardest operational aspect of caching entirely, which is why it has become the default in build tooling rather than an optimisation to consider.

**Cache-control on authenticated paths is a correctness concern.** A per-user response cached without a private directive is stored at an edge and served to whoever asks next, which is a data leak that propagates globally in seconds and is invisible from the application's perspective. Because the default behaviour varies between providers and a single misconfigured route is sufficient, the safe pattern is explicit headers applied centrally rather than per handler.

**The last increments of hit rate matter most.** Going from ninety to ninety-nine per cent reduces origin traffic tenfold, and the next nine reduces it tenfold again — so investigating why a specific content type is missing is far more valuable than general tuning. Aggregate hit rate obscures this, since a healthy overall figure can hide one high-volume path bypassing the cache entirely because of a cookie or a query parameter.

**Expiry of popular objects produces a burst at the origin.** When a widely requested object becomes stale, many edges miss simultaneously and forward to the origin together, which is the stampede problem at global scale. Request collapsing at the edge, a shielding tier that absorbs those misses, and staggered lifetimes each address it, and a system relying on a single long TTL without any of them will see periodic origin spikes it cannot explain from its own traffic.

**Its boundary is content that is identical across users.** Personalised responses, writes and anything requiring current origin state are outside what a cache can serve, so the benefit is confined to the shared portion of traffic. Edge compute has narrowed that boundary by allowing per-user assembly near the user, but the underlying principle is unchanged: the CDN helps precisely to the extent that many people are asking for the same bytes.

**Related patterns:** Adaptive Bitrate Streaming · Cache Invalidation · Stale While Revalidate · Object Storage

---

### Adaptive Bitrate Streaming

*Encode video at several qualities, split it into short segments, and let the player switch between them as available bandwidth changes.*

> **When you hear…** Video delivery to varied connections · buffering harming the experience · mobile networks whose capacity changes constantly

**Flow:** `Encode at several bitrates` → `Split into segments` → `Player measures bandwidth` → `Requests matching quality` → `Switches mid-stream`

**The problem**

A single high-quality encode is unwatchable on a slow connection — the player buffers continuously — while a single low-quality encode wastes the capability of fast connections and looks poor on a large screen.

The connection also changes while watching. A viewer who moves between networks or whose signal degrades will experience a stream chosen for conditions that no longer apply, and one fixed choice cannot follow that.

> **Make quality a per-segment decision the player keeps re-making**  
> If the video exists at several bitrates and is divided into short segments that are interchangeable at the boundaries, the player can choose a different quality for each one. Quality then tracks bandwidth continuously — degrading rather than stalling when capacity drops, and improving as soon as it returns — because the decision is revisited every few seconds instead of once at the start.

**Mental model**

Several parallel encodings of the same content, aligned so that segments are interchangeable, described by a manifest the player reads and uses to choose.

1. **Encode** — The source is transcoded to several bitrate and resolution combinations.
2. **Segment** — Each rendition is split into aligned chunks of a few seconds.
3. **Describe** — A manifest lists the renditions and their segments.
4. **Measure** — The player estimates throughput and watches its buffer.
5. **Switch** — It requests the next segment at whatever quality it can sustain.

> **Switching decisions based only on measured throughput oscillate visibly**  
> Bandwidth estimates are noisy, so a player reacting to each measurement flips between qualities every few seconds, which viewers find more objectionable than a consistently lower quality. Buffer occupancy is the more stable signal — a full buffer means there is room to try higher, a draining one means drop now — and good algorithms combine both with hysteresis.

**How it works**

**Ladder, segments and switching**

```text
ENCODING LADDER
  1080p  5.0 Mbps
   720p  2.5 Mbps
   480p  1.0 Mbps
   360p  0.6 Mbps
   240p  0.3 Mbps
  -> roughly a factor of 2 between rungs
  -> closer spacing means more storage and more
     switching for little perceptual gain

SEGMENTS
  2-10 s, typically 4-6 s
  each begins with a keyframe so renditions are
  interchangeable at boundaries
  -> shorter: faster adaptation, more requests,
     more overhead
  -> longer: fewer requests, slower reaction to
     bandwidth changes

STORAGE COST
  5 renditions ≈ 2.3x the top bitrate in total
  plus per-segment overhead
  -> a 1-hour video: several GB across the ladder

SWITCHING LOGIC
  throughput estimate  noisy, reacts fast
  buffer occupancy     stable, reacts late
  combined:
    buffer high AND throughput supports it -> step up
    buffer draining                        -> step
                                              down
    require sustained evidence before stepping up
  -> asymmetric: drop quickly, rise slowly

STARTUP
  begin at a low rendition for a fast first frame
  step up once bandwidth is measured
  -> startup time matters more to abandonment than
     quality does
```

1. **Space the ladder roughly by a factor of two** — Closer rungs add storage and switching without perceptible benefit.
2. **Align segment boundaries across renditions** — Interchangeability at boundaries is what makes switching possible.
3. **Drive switching from buffer occupancy, not throughput alone** — Throughput estimates are too noisy to act on directly.
4. **Step down quickly and up slowly** — A stall is far worse than a few seconds at lower quality.
5. **Start low for a fast first frame** — Startup delay drives abandonment more than quality does.
6. **Serve every segment from a CDN** — Segments are static files, which is what makes this scale.

**Why segmentation makes delivery easy**

```text
EVERY SEGMENT IS A STATIC FILE
  -> cacheable at the edge like any other object
  -> no streaming protocol at the server
  -> no per-viewer server state
  -> ordinary HTTP, so it works everywhere

THIS IS THE REAL INSIGHT
  adaptive streaming turned video from a stateful
  streaming problem into a static file delivery
  problem
  -> which CDNs already solve extremely well

LIVE STREAMING
  same mechanism, manifest updated continuously
  latency = segment duration x buffer depth
    4 s segments, 3 buffered -> ~12 s behind live
  -> low-latency variants use partial segments to
     cut this to 2-5 s

DRM AND ENCRYPTION
  segments encrypted, keys served separately
  -> licence acquisition adds startup latency
  -> another reason to keep the first segment small

QUALITY METRICS THAT MATTER
  startup time
  rebuffering ratio        (the one viewers hate)
  average bitrate delivered
  switch frequency
  -> optimising average bitrate alone produces a
     worse experience than optimising rebuffering
```

| Metric | Value | Note |
|---|---|---|
| Ladder | ~5 renditions | ×2 spacing |
| Segment | 4-6 s | **aligned across renditions** |
| Delivery | static files | CDN does the work |
| Priority | avoid rebuffering | over peak quality |

> **Optimising for the highest sustainable bitrate produces a worse experience than avoiding stalls**  
> Players tuned to maximise delivered quality sit close to the limit of available bandwidth and stall whenever it dips, and viewers consistently rate a rebuffer as far more damaging than a step down in resolution. The correct objective is smooth playback at a reasonable quality, which means keeping a safety margin and preferring to descend early rather than to hold a rendition the connection can barely support.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Protocols | HLS, DASH | Both segment-and-manifest; differ in details |
| Codecs | H.264, H.265, AV1 | Newer codecs cut bitrate at higher encoding cost |
| Transcoding | Batch or just-in-time | Just-in-time saves storage, adds latency |
| Delivery | CDN | Segments are ordinary cacheable files |
| Players | Client libraries with switching logic | The switching algorithm is where quality of experience is won |
| Per-title encoding | Ladder tuned per video | Animation and live action need very different bitrates |

Per-title encoding is a meaningful refinement: a fixed ladder wastes bitrate on simple content and starves complex content, whereas analysing each title and choosing rungs for it can cut delivered bytes substantially at the same perceived quality.

**Trade-offs**

**Video delivery approaches**

| Approach | Adapts to bandwidth | Storage | Complexity |
|---|---|---|---|
| Single progressive file | No | Lowest | Lowest |
| Several files, user chooses | Only manually | Moderate | Low |
| Adaptive bitrate | Automatically, per segment | Highest | Moderate |
| Just-in-time transcoding | Automatically | Low | High; compute per view |
| Per-title adaptive ladders | Automatically, tuned | High | Highest |

Just-in-time transcoding trades storage for compute, which suits long-tail catalogues where most titles are rarely watched — encoding on demand costs nothing for content nobody requests, while a full ladder for every title does.

> **Ask before choosing it**  
> How varied are the viewers' connections, and how long is the content? A short clip on predictable connections may not justify a ladder at all, while anything watched on mobile networks for more than a minute almost certainly does.

**How it fails**

**How adaptive streaming goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Quality oscillates visibly | Switching on noisy throughput estimates | Buffer-based decisions with hysteresis |
| Frequent rebuffering | Player holding a rendition too aggressively | Step down earlier; keep a margin |
| Slow startup | Starting at a high rendition | Begin low, step up after measuring |
| Switching produces glitches | Segment boundaries not aligned | Align keyframes across renditions |
| Storage costs excessive | Too many rungs, or full ladders for rare titles | Wider spacing; just-in-time for the tail |
| Poor experience on fast connections | Ladder topping out too low | Add a higher rendition |
| Live latency too high | Long segments and deep buffers | Shorter or partial segments |

> **Misaligned segment boundaries make switching visible or impossible**  
> Renditions must be cut at the same points, each starting with a keyframe, or the player cannot splice them cleanly — producing artefacts, audio discontinuities, or an inability to switch at all. The problem originates in the encoding pipeline rather than the player, which is why it is often diagnosed late and why encoding configuration deserves verification against actual switching behaviour.

**Where it is used**

- **Video-on-demand platforms**, where the ladder and player algorithm largely determine perceived quality.
- **Live streaming services**, using the same segmentation with continuously updated manifests.
- **Social video feeds**, where fast startup matters more than peak quality.
- **Education and conferencing platforms**, adapting to widely varying connection quality.
- **Audio streaming**, applying the same principle with a much smaller ladder.

**In the interview**

The point worth making is that segmentation turns streaming into static file delivery.

- **Describe the ladder and segmentation** with concrete numbers — five renditions, four-to-six-second segments.
- **Say segments are static files**, which is what lets a CDN carry the load with no per-viewer server state.
- **Prefer buffer-based switching** and explain why throughput estimates alone cause oscillation.
- **Make the objective explicit**: minimise rebuffering rather than maximise bitrate, because viewers weight stalls heavily.
- **Note the live latency arithmetic** — segment duration times buffer depth — and how partial segments reduce it.

**Practice drill**

Design delivery for a video platform whose viewers range from mobile networks to fibre. Specify the encoding ladder, the segment duration and the storage cost for one hour of content. Describe the player's switching algorithm, including what it uses to decide and how it avoids oscillation. Then compute the live latency your design produces and say how you would reduce it by half.

**Go deeper**

Adaptive bitrate streaming encodes content at several qualities in aligned segments, letting the player choose a rendition for each segment as conditions change.

**Its most consequential property is architectural rather than perceptual.** Splitting video into aligned segments turns a stateful streaming problem into the delivery of ordinary static files, which CDNs already solve extremely well. There is no per-viewer server state, no specialised protocol at the origin, and every segment is cacheable at the edge — which is why global video delivery became tractable on commodity infrastructure.

**Continuous re-decision is what handles changing conditions.** A single quality chosen at the start is wrong as soon as the network changes, whereas a decision revisited every few seconds follows the connection — degrading gracefully when capacity falls and recovering when it returns. The segment duration therefore sets the adaptation speed, and the usual four-to-six-second choice balances responsiveness against request overhead.

**The switching signal matters more than the ladder.** Throughput estimates are noisy enough that acting on them directly produces visible oscillation between qualities, which viewers dislike more than a steady lower resolution. Buffer occupancy is slower but far more stable, and combining the two with hysteresis — plus an asymmetry that drops quickly and rises cautiously — is what separates a player that feels smooth from one that is technically adaptive and unpleasant to watch.

**The objective should be smooth playback, not maximum bitrate.** Viewers weight rebuffering very heavily and resolution changes comparatively lightly, so an algorithm that holds the highest sustainable quality and stalls occasionally scores worse than one that keeps a margin and never stalls. Startup time matters similarly: beginning at a low rendition produces a first frame quickly, and abandonment is driven far more by waiting than by initial quality.

**Alignment across renditions is a pipeline requirement, not a player one.** Segments must be cut at identical points with keyframes at their starts, or splicing produces artefacts or fails outright. Because the symptom appears during playback and the cause is in encoding, this class of problem is typically found late — which makes verifying switching behaviour against actual encoded output part of the delivery pipeline rather than an afterthought.

**Storage and compute are the adjustable costs.** A full ladder multiplies storage by roughly two and a half times the top rendition, which is substantial across a large catalogue, while just-in-time transcoding trades that for compute at view time and suits long tails where most titles are rarely watched. Per-title ladders refine it further, since animation and live action need very different bitrates for the same perceived quality, and a fixed ladder is necessarily wrong for both.

**Related patterns:** CDN · Object Storage · Multipart Upload · Backpressure

---

## Media Storage

### Object Storage

*Store immutable blobs addressed by key in a flat namespace, trading filesystem semantics for effectively unlimited, cheap, durable capacity.*

> **When you hear…** Large files outgrowing a database · media, backups and logs · storage that must scale without capacity planning

**Flow:** `Key and bytes` → `Flat namespace` → `Replicated durably` → `HTTP access` → `Immutable objects`

**The problem**

Uploaded images stored in a database bloat it, slow every backup, and consume the buffer pool that queries need. Stored on a server's local disk they are lost when the instance is replaced and unavailable to every other instance.

A shared filesystem solves the sharing and adds its own limits: capacity must be planned, directories degrade with millions of entries, and throughput is bounded by whatever is mounting it.

> **Give up filesystem semantics and the constraints go with them**  
> Most large files are written once and read many times, never modified in place, and never need directory operations. A store that offers only put, get and delete on whole objects in a flat namespace can distribute them arbitrarily, replicate them for durability, serve them over HTTP, and grow without anyone planning capacity — because none of the guarantees that make filesystems hard to distribute are being provided.

**Mental model**

A distributed key-value store for large values. Keys look like paths but are flat strings; values are opaque bytes plus metadata.

1. **Put** — An object is written whole and becomes immutable.
2. **Replicate** — Copies or erasure-coded fragments are spread across failure domains.
3. **Get** — Objects are retrieved by key, usually over HTTP.
4. **Tier** — Access patterns move objects between cost classes.
5. **Delete** — Objects are removed; there is no in-place modification.

> **There is no rename, no append and no partial update**  
> Changing one byte means rewriting the object, and moving a prefix means copying every object under it. Operations that are trivial on a filesystem are linear in data volume here, which makes naming decisions permanent in practice — a directory structure chosen badly cannot be reorganised cheaply once it holds millions of objects.

**How it works**

**Naming, listing and consistency**

```text
THE NAMESPACE IS FLAT
  "photos/2026/09/14/abc.jpg" is one key
  the slashes are a convention, not a hierarchy
  -> listing by prefix is a scan, not a directory read
  -> listing a prefix with millions of objects is
     slow and paginated

KEY DESIGN
  include everything needed to find the object
  without listing:
    tenant/entity/date/id
  -> the goal is to construct the key from what you
     already know, never to search for it

  avoid sequential prefixes where the provider
  partitions by key range
    20260914-0001, 20260914-0002 ...
    -> all writes land in one partition
  -> prefix with a hash where write rates are high

IMMUTABILITY IS A FEATURE
  content-addressed keys (hash of the bytes)
    -> deduplication for free
    -> caching forever is safe
    -> no invalidation problem

CONSISTENCY
  modern providers give read-after-write for new
  objects
  -> but LIST operations can still lag
  -> never use a list to determine whether a write
     succeeded; use the write's own response
```

1. **Design keys so objects are found by construction, not by listing** — Listing is a scan and becomes unusable at scale.
2. **Avoid sequential key prefixes at high write rates** — Range-partitioned stores concentrate those writes on one partition.
3. **Use content-addressed keys where it fits** — Deduplication, safe permanent caching and no invalidation.
4. **Keep metadata in a database, not in the object store** — Queries, filters and joins need an index the store does not have.
5. **Apply lifecycle policies deliberately** — Tiering and expiry are where most of the cost saving lives.
6. **Serve public objects through a CDN** — Direct reads at scale cost more and are slower than cached ones.

**Cost structure and durability**

```text
WHAT YOU PAY FOR
  storage per GB-month     cheap
  requests                 per thousand; matters at
                           high object counts
  egress                   usually the dominant cost
  -> serving through a CDN cuts egress dramatically

TIERING
  hot      immediate access, highest storage price
  infrequent  cheaper storage, retrieval fee
  archive  very cheap, minutes to hours to restore
  -> lifecycle rules move objects automatically
  -> retrieval fees make aggressive tiering backfire
     for data that is actually read

DURABILITY
  providers quote very high durability figures based
  on replication or erasure coding across failure
  domains
  -> durability is not the same as availability, and
     neither is a backup

WHAT IT DOES NOT PROTECT AGAINST
  deleting the object yourself
  a bug overwriting it
  credentials being compromised
  -> versioning, and separate backups with separate
     access, are what cover these
```

| Metric | Value | Note |
|---|---|---|
| Namespace | flat | **prefix listing is a scan** |
| Mutation | whole object only | no append or rename |
| Dominant cost | egress | use a CDN |
| Durability | not a backup | versioning matters |

> **High durability is not protection against your own mistakes**  
> Replication protects against hardware failure and nothing else. An object deleted by a buggy job, overwritten by a bad deploy, or destroyed by a compromised credential is gone with the same reliability that keeps it safe from disk failure. Versioning, deletion protection and a copy under separate credentials are what address that, and the quoted durability figure says nothing about any of it.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Cloud object stores | S3 and equivalents | The default; effectively unlimited |
| Self-hosted | MinIO, Ceph | Compatible APIs on your own hardware |
| CDN in front | Any provider | Cuts egress cost and latency |
| Metadata store | A database alongside | The object store cannot answer queries |
| Block storage | Attached volumes | For data needing filesystem semantics |
| File storage | Managed shared filesystems | When applications require POSIX behaviour |

Pairing an object store with a database is the standard arrangement, and the division is worth stating clearly: bytes live in the store, and everything you might want to query — ownership, tags, timestamps, status — lives in the database with the key as a pointer.

**Trade-offs**

**Storage options for large files**

| Option | Scales | Semantics | Cost |
|---|---|---|---|
| Database blobs | Poorly | Transactional | Highest |
| Local disk | Not at all | Full filesystem | Low, and ephemeral |
| Network filesystem | To a point | Full filesystem | Moderate |
| Object storage | Effectively unlimited | Put, get, delete only | Lowest |
| Object storage with a CDN | Effectively unlimited | Same, read-optimised | Lowest, plus cache |

Database blobs are worth a moment of defence: for small files in modest quantities, keeping them transactional with the row that references them removes an entire class of consistency problems, and the cost only becomes unreasonable as volume grows.

> **Ask before choosing it**  
> Do these files ever need to be modified in place, or listed dynamically by attributes? Both are awkward here — the first requires rewriting whole objects, the second requires a database — and discovering either after a hundred million objects exist is expensive.

**How it fails**

**How object storage goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Listing operations time out | Millions of objects under one prefix | Design keys for direct construction |
| Write throughput limited | Sequential key prefixes | Hash or randomise the prefix |
| Egress costs dominate | Serving directly to users | Put a CDN in front |
| Objects deleted irrecoverably | No versioning | Enable versioning and deletion protection |
| Data exposed publicly | Overly broad access policy | Default deny; explicit narrow grants |
| Retrieval costs unexpected | Aggressive archival tiering on active data | Tier by measured access patterns |
| Consistency confusion | Using listings to confirm writes | Trust the write response, not a list |

> **Public access misconfiguration is the most common serious failure in object storage**  
> A bucket or prefix granted broad read access exposes everything under it to anyone who finds the name, and because the store is reachable over ordinary HTTP there is no network boundary providing a second line of defence. The failure is silent — nothing in the application behaves differently — and it is discovered from outside, which is why default-deny policies and automated checking of access configuration are standard practice rather than optional hardening.

**Where it is used**

- **User-uploaded media**, the archetypal use: images, video and documents.
- **Backups and archives**, where durability and cheap cold tiers matter most.
- **Data lakes**, storing partitioned columnar files for batch processing.
- **Static website and asset hosting**, usually behind a CDN.
- **Video segment storage** for adaptive streaming, where every segment is an object.

**In the interview**

Be precise about what is given up, since the constraints are what make the scalability possible.

- **Name the missing semantics** — no append, no rename, no partial update — and note the consequences for key design.
- **Say the namespace is flat** and that prefix listing is a scan, so keys must be constructible rather than searchable.
- **Keep metadata in a database**, with the object key as the pointer, because the store cannot answer queries.
- **Distinguish durability from backup.** Replication does not protect against deletion, bugs or compromised credentials.
- **Raise egress cost and access policy**, which are the two things that most often go wrong in production.

**Practice drill**

Design storage for a product where users upload photos, each with a caption, tags and privacy settings. Specify the key structure, what lives in the database, and how a user's photos are listed. Say what happens when a photo is edited and when it is deleted. Then estimate the monthly cost at ten million photos averaging 2 MB with a million views a day, and identify which line dominates.

**Go deeper**

Object storage holds immutable blobs addressed by key in a flat namespace, offering vast capacity by declining to provide filesystem semantics.

**What it gives up is precisely what makes it scalable.** In-place modification, rename, append and hierarchical directories are the operations that make distributed filesystems difficult, and large files rarely need any of them — they are written once and read many times. Removing those guarantees lets objects be distributed arbitrarily and replicated independently, which is why capacity becomes something nobody plans rather than something that runs out.

**Key design is a permanent decision and should be treated as one.** Because the namespace is flat, finding an object means constructing its key rather than searching for it, and reorganising a prefix means copying every object beneath it. A structure chosen casually at the start becomes expensive to change once millions of objects exist, which makes the naming scheme worth more deliberation than it usually receives.

**The store holds bytes and a database holds everything you might ask about them.** Object stores cannot filter by attribute, sort by date or join, so any query — this user's photos, everything tagged this way, uploads from last week — needs an index elsewhere with the key as a pointer. Systems that attempt to answer such questions by listing prefixes work during development and fail at scale, since listing is a paginated scan.

**Immutability composes well with content addressing.** Naming an object by the hash of its contents gives deduplication automatically, makes indefinite caching safe because the name changes whenever the content does, and removes invalidation entirely. This is the same reasoning that makes hashed asset filenames standard at the CDN layer, applied one level down.

**Egress usually dominates the bill, which surprises people who budget for storage.** Storing bytes is inexpensive; moving them out repeatedly is not, and a popular object served directly to users can cost far more in transfer than in storage. Fronting the store with a CDN addresses both cost and latency, and it is the single most effective economic change available for read-heavy workloads.

**Quoted durability addresses one failure mode and people hear it as addressing all of them.** Replication across failure domains protects against hardware loss and does nothing about a job that deletes the wrong prefix, a deploy that overwrites objects, or a leaked credential. Versioning, deletion protection and a copy held under separate access are what cover those — and because the store is reachable over ordinary HTTP with no network boundary behind it, access policy misconfiguration remains the most common way real data is lost or exposed.

**Related patterns:** Multipart Upload · Presigned URL · CDN · Batch Processing

---

### Multipart Upload

*Split a large upload into independent parts that can be sent in parallel and retried individually, then assemble them server-side.*

> **When you hear…** Files large enough that a single request is fragile · uploads failing near completion · throughput limited by one connection

**Flow:** `Initiate upload` → `Split into parts` → `Upload in parallel` → `Retry failed parts` → `Complete and assemble`

**The problem**

A five-gigabyte upload sent as one request fails after forty minutes because the connection dropped, and every byte must be sent again. On an unreliable network the probability of completing a long single transfer approaches zero as the file grows.

One connection is also slow. A single TCP stream rarely saturates available bandwidth over a long path, so the transfer is limited by one connection's throughput rather than by the network's capacity.

> **Make failure cost one part instead of the whole file**  
> If the upload is divided into independently addressable pieces, a failure retries only the piece that failed, and the pieces can be sent concurrently to use the full available bandwidth. The transfer also becomes resumable across sessions, because completed parts are durable on the server and the client only needs to know which ones remain.

**Mental model**

A three-phase protocol: begin an upload and receive an identifier, send numbered parts independently, then complete by listing the parts and their checksums.

1. **Initiate** — The server returns an upload identifier.
2. **Split** — The client divides the file into numbered parts.
3. **Upload** — Parts are sent concurrently, each acknowledged with a checksum.
4. **Retry** — Failed parts are resent individually.
5. **Complete** — The client submits the part list and the server assembles the object.

> **Incomplete uploads consume storage indefinitely and are invisible**  
> Parts of an upload that is never completed remain stored, billed, and absent from any object listing — so they accumulate silently. Without a lifecycle rule that aborts stale uploads, a system with frequent client failures can accrue substantial cost in data nobody can see and nobody is using.

**How it works**

**Part sizing and parallelism**

```text
PART SIZE
  too small  more requests, more overhead, per-part
             limits reached
  too large  a failure costs more to retry
  typical 5-100 MB; 8-16 MB is a common default

  5 GB file, 10 MB parts -> 500 parts
  5 GB file, 50 MB parts -> 100 parts

PARALLELISM
  4-8 concurrent parts saturates most connections
  more than that contends with itself and with the
  user's other traffic
  -> on mobile, keep it low; battery and radio
     behaviour matter

RETRY
  each part retried independently with backoff
  -> a single failure costs one part, not the file
  -> failure probability per part is small, and the
     whole transfer becomes reliable

RESUMABILITY
  the upload identifier and completed part numbers
  are enough to resume after the app is closed
  -> persist them client-side
  -> query the server for which parts it already has

COMPLETION
  the client sends part numbers with their checksums
  the server verifies and assembles
  -> a mismatch fails the completion rather than
     producing a corrupt object
```

1. **Choose parts in the tens of megabytes** — Small enough to retry cheaply, large enough to avoid request overhead.
2. **Upload a handful of parts concurrently** — Four to eight saturates most links without self-contention.
3. **Persist the upload identifier client-side** — It is what makes resumption across sessions possible.
4. **Verify each part with a checksum** — Corruption should fail completion, not produce a broken object.
5. **Set a lifecycle rule to abort stale uploads** — Otherwise abandoned parts accumulate cost invisibly.
6. **Combine with presigned URLs** — Parts can then go directly from the client to storage.

**Reliability arithmetic**

```text
SINGLE UPLOAD, 5 GB over 40 minutes
  P(connection survives 40 min) on a mobile network
  might be 0.7
  -> 30% of uploads fail and restart from zero
  -> and each retry has the same odds

MULTIPART, 10 MB parts, ~5 s each
  P(a part succeeds) ~0.999
  a failed part costs 5 s, not 40 minutes
  -> effectively every upload completes
  -> total time is lower because parts run in
     parallel

THROUGHPUT
  one TCP stream over a long path is limited by
  window size and loss
  4 parallel streams often deliver 2-4x the
  throughput of one
  -> the speed benefit is frequently larger than
     expected

WHEN NOT TO BOTHER
  files under ~100 MB on reliable networks
  -> the protocol overhead is not worth it
  -> most SDKs switch automatically above a
     threshold
```

| Metric | Value | Note |
|---|---|---|
| Failure cost | one part | **not the file** |
| Throughput | 2-4× faster | parallel streams |
| Part size | 8-16 MB | common default |
| Hazard | abandoned parts | lifecycle rule |

> **Parallelism beyond a handful of parts stops helping and starts hurting**  
> Concurrent streams compete for the same bandwidth, so past roughly eight the aggregate throughput plateaus while contention, memory use and battery consumption keep rising. On mobile clients especially, aggressive parallelism can make transfers slower and noticeably degrade everything else the device is doing — the right number is small and should be verified rather than maximised.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Object store APIs | Native multipart support | The standard mechanism |
| Client SDKs | Automatic switching above a threshold | Handles splitting and retry |
| Presigned part URLs | Direct client-to-storage upload | Keeps bytes away from your servers |
| Resumable HTTP uploads | Protocols like tus | Similar goals for arbitrary servers |
| Chunked transfer encoding | Streaming a single request | Not resumable; different purpose |
| Lifecycle rules | Abort incomplete uploads | Necessary cost control |

Combining multipart with presigned URLs is the arrangement most large-upload systems use: the application authorises each part and the bytes travel directly from the client to storage, so upload volume never touches application servers.

**Trade-offs**

**Upload strategies**

| Strategy | Failure cost | Throughput | Complexity |
|---|---|---|---|
| Single request | The whole file | One stream | Lowest |
| Single request with resume | Remaining bytes | One stream | Low |
| Multipart | One part | Parallel streams | Moderate |
| Multipart via presigned URLs | One part | Parallel, direct to storage | Moderate |
| Client-side chunking to your API | One chunk | Parallel, through your servers | High; you assemble |

The last row is what teams build when they do not realise the object store already provides this: chunking to an application API means handling assembly, storage of partial data and cleanup, all of which the storage layer does natively.

> **Ask before choosing it**  
> How large are the files and how reliable are the clients' networks? Below about a hundred megabytes on good connections the overhead is not repaid, and most SDKs already switch automatically at a sensible threshold — so the decision is usually about whether to override that default.

**How it fails**

**How multipart uploads go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Storage cost from invisible data | Abandoned uploads never aborted | Lifecycle rule to expire incomplete uploads |
| Uploads still fail entirely | Parts too large | Reduce part size |
| Slow despite parallelism | Too many concurrent parts contending | Reduce concurrency to a handful |
| Corrupt objects assembled | No per-part checksums | Verify each part; fail completion on mismatch |
| Cannot resume after app restart | Upload identifier not persisted | Store it and the completed part numbers |
| Application servers saturated | Bytes routed through your API | Presigned part URLs direct to storage |
| Part count limits exceeded | Very small parts on a very large file | Scale part size with file size |

> **Abandoned multipart uploads are billed storage that appears nowhere**  
> Parts belonging to an upload that was never completed do not appear in object listings, so they are invisible to the usual ways of inspecting what a bucket contains, while continuing to consume storage and accrue charges. In systems where clients frequently abandon uploads this accumulates steadily, and it is typically discovered through a cost investigation rather than through anything in the application — which is why a lifecycle rule aborting stale uploads belongs in the initial configuration.

**Where it is used**

- **Video upload platforms**, where files are large and client networks are unpredictable.
- **Backup and archival tools**, transferring very large artefacts reliably.
- **Mobile applications**, where resumption across sessions and interruptions is essential.
- **Data ingestion pipelines**, moving large datasets into object storage.
- **Desktop sync clients**, uploading large files in the background over long periods.

**In the interview**

Use the reliability arithmetic; it makes the case in one calculation.

- **Contrast failure costs**: one part of a few seconds against a forty-minute transfer starting over.
- **Mention the throughput gain**, since parallel streams often deliver several times a single stream's rate.
- **Give a part size and a concurrency figure**, and say why more parallelism stops helping.
- **Describe resumption** — persisted upload identifier and completed part list — as the client-side requirement.
- **Raise abandoned uploads** and the lifecycle rule, because it is invisible cost that few candidates mention.

**Practice drill**

Design the upload path for 5 GB video files from mobile clients. Choose the part size and concurrency, and justify both. Estimate the transfer time and the probability of completion compared with a single request. Describe exactly what the client persists so an upload survives the app being closed, and specify the storage configuration that prevents abandoned uploads from accumulating.

**Go deeper**

Multipart upload divides a large transfer into independently retryable parts that can be sent concurrently and assembled server-side.

**It changes the unit of failure, which is what makes large uploads viable.** A single long transfer must survive its entire duration, and that probability falls as files grow and networks degrade — so a five-gigabyte upload over a mobile connection may rarely complete at all. Splitting it means a failure costs seconds rather than the whole transfer, and the aggregate reliability becomes high even when individual attempts are not.

**Parallelism provides a throughput gain that is often larger than expected.** A single TCP stream over a long path is limited by window size and packet loss well below the available bandwidth, so several concurrent streams frequently deliver multiples of one stream's rate. The benefit plateaus quickly — beyond a handful of parts the streams contend with each other — which makes a small concurrency figure correct and maximising it counterproductive, particularly on mobile devices where it also affects battery and other traffic.

**Resumability comes almost for free and matters most on the clients that need it.** Because completed parts are durable server-side, an upload can continue after the application is closed or the device changes networks, provided the client persists the upload identifier and knows which parts succeeded. For mobile and desktop sync clients this turns a fragile operation into a background one, which is frequently the difference between a feature working and being abandoned by users.

**Per-part checksums make corruption a failed completion rather than a bad object.** Verifying each part as it arrives, and validating the list at completion, means a transmission error produces an error the client can act on instead of an assembled object with corrupted bytes in the middle. Without it the corruption is discovered later by whoever reads the file, at which point the original is long gone.

**Abandoned uploads are a genuine and invisible cost.** Parts of an upload that never completes do not appear in object listings, so they are absent from the usual ways of examining storage, while continuing to consume and bill. Where clients frequently abandon transfers — which is normal for mobile — this accumulates steadily and is typically discovered through cost analysis rather than through anything in the application, making an abort-incomplete lifecycle rule part of the initial setup rather than a later optimisation.

**It composes naturally with direct-to-storage uploads.** Authorising each part individually and letting the client send bytes straight to the object store keeps upload traffic entirely off application servers, which for a video platform is the difference between a small API tier and one sized for the full ingest bandwidth. The application retains control over who may upload and where, while never handling the data itself.

**Related patterns:** Object Storage · Presigned URL · Retry With Jitter · Idempotency Key

---

### Presigned URL

*Issue a time-limited signed URL that grants one specific operation, so clients read or write storage directly without holding credentials.*

> **When you hear…** Uploads or downloads proxied through application servers · large files consuming API capacity · clients needing storage access without credentials

**Flow:** `Client asks permission` → `Server signs a URL` → `Client uses it directly` → `Storage verifies signature` → `URL expires`

**The problem**

Every upload passes through the application servers so they can authorise it, which means the API tier must be sized for the full bandwidth of user uploads — a video platform's ingest capacity becomes an application scaling problem rather than a storage one.

The alternative of giving clients storage credentials is worse. Credentials in a mobile application are extractable, they usually grant far more than the single operation needed, and revoking them affects every user at once.

> **Sign the permission, not the session**  
> A URL carrying a signature that encodes one operation, one object and an expiry is a capability rather than a credential: it authorises exactly one thing for a short time, cannot be extended by the holder, and needs no account on the storage system. The application keeps full control over who may do what, while the bytes travel directly between the client and the store.

**Mental model**

The application signs a request the client will make later. The storage service verifies the signature and honours precisely what was signed.

1. **Request** — The client asks the application for permission to upload or download.
2. **Authorise** — The application applies its own rules — ownership, quota, content type.
3. **Sign** — It produces a URL encoding the operation, the object, constraints and an expiry.
4. **Use** — The client sends bytes to or from storage directly.
5. **Expire** — The URL stops working; no revocation is needed.

> **A presigned URL is a bearer capability and anyone holding it can use it**  
> The signature does not identify the user, so a URL shared, logged or leaked works for whoever has it until it expires. Short lifetimes are the primary control — minutes rather than hours — and download URLs for sensitive content should be issued per request rather than embedded in pages that might be cached, forwarded or indexed.

**How it works**

**What the signature can constrain**

```text
THE SIGNED REQUEST FIXES
  method        PUT for upload, GET for download
  object key    exactly one object
  expiry        absolute time
  optionally:
    content-type       must match on upload
    content-length     size bounds
    checksum           the client must supply one
  -> anything not constrained is unconstrained

UPLOAD FLOW
  client: "I want to upload a 4 MB jpeg"
  server: checks quota, ownership, plan limits
          generates key: users/42/photos/<uuid>.jpg
          signs a PUT for that key, 15 min expiry,
          content-type image/jpeg, max 10 MB
  client: PUTs the bytes directly to storage
  client: tells the server it finished
  server: verifies the object exists and its size,
          then records it in the database

WHY THE CONFIRMATION STEP MATTERS
  the application never sees the bytes
  -> it does not know the upload happened unless the
     client says so, or storage notifies it
  -> a client that uploads and vanishes leaves an
     orphan object
  -> use storage event notifications where available;
     otherwise reconcile periodically

DOWNLOAD FLOW
  server authorises, signs a GET, short expiry
  -> content-disposition can force a filename
  -> per-request signing keeps the capability tight
```

1. **Keep expiry short** — The URL is a bearer capability; minutes, not hours.
2. **Constrain content type and size on uploads** — An unconstrained signature permits any content of any size.
3. **Generate the object key server-side** — Letting the client choose allows overwriting other objects.
4. **Confirm uploads through storage events where possible** — Relying on the client to report completion loses objects.
5. **Reconcile orphaned objects periodically** — Uploads that are never confirmed accumulate otherwise.
6. **Sign download URLs per request** — Embedding long-lived URLs in pages spreads the capability widely.

**What it saves, and the risks**

```text
WITHOUT PRESIGNED URLS
  10,000 uploads/hour at 5 MB
  = 50 GB/hour through the application tier
  -> servers sized for bandwidth, not for logic
  -> every upload holds a connection for its duration

WITH PRESIGNED URLS
  10,000 signing requests/hour
  each a few hundred bytes and a signature
  -> the API tier is sized for its actual work
  -> bandwidth is storage's problem, where it is
     cheap and elastic

THE RISKS AND THEIR CONTROLS
  leaked URL
    -> short expiry limits the window
  client uploading something unexpected
    -> constrain content-type and size
  client overwriting another object
    -> server chooses the key; never the client
  orphaned uploads
    -> storage event notifications, plus reconciliation
  URL appearing in logs or referrers
    -> short expiry; avoid embedding in HTML where
       possible

WHAT IT IS NOT
  not a substitute for authorisation; the
  application still decides who may do what — it
  just decides once, in advance
```

| Metric | Value | Note |
|---|---|---|
| API bandwidth | 50 GB/h → ~0 | **direct to storage** |
| Capability | one operation | one object |
| Expiry | minutes | the main control |
| Key choice | server-side | never the client |

> **Letting the client choose the object key permits overwriting other people's data**  
> If the upload key comes from the request, a client can request a signature for a path belonging to someone else and overwrite it — the storage service checks the signature, not the intent. The application must construct the key from the authenticated identity and its own naming scheme, and treat any client-supplied path as untrusted input to be discarded rather than used.

**Technologies**

| Piece | Options | Note |
|---|---|---|
| Object store signing | Native presigned URL support | Available across providers |
| Signed policies | Constrain size, type and prefix | Stronger constraints than a plain signed URL |
| CDN signed URLs | Time-limited access to cached content | For protected downloads at the edge |
| Storage event notifications | Server learns of completed uploads | Removes reliance on client confirmation |
| Multipart with signed parts | Large uploads direct to storage | The standard combination |
| Proxying through the API | The alternative | Simple, and sized for full bandwidth |

Event notifications are the piece that makes this robust: rather than trusting a client to report that it finished, the storage service tells the application an object appeared, which closes the gap where uploads complete but are never recorded.

**Trade-offs**

**Client access to storage**

| Approach | API bandwidth | Credential exposure | Control |
|---|---|---|---|
| Proxy through the API | Full | None | Complete, per byte |
| Presigned URLs | Signing requests only | None | Per operation, in advance |
| Client-held credentials | None | High | Coarse, hard to revoke |
| Public bucket | None | n/a | None |
| CDN signed URLs | None | None | Per operation, cached |

Proxying retains one genuine advantage: the application sees the bytes and can scan, transform or validate them inline. Where content inspection is a requirement, presigned uploads must be paired with asynchronous scanning after the fact, which is a different guarantee.

> **Ask before choosing it**  
> Does anything need to inspect the content as it passes? Virus scanning, transcoding triggers and content moderation can all run after the upload, but they then run against an object that already exists — and if the requirement is to reject content before it is stored, proxying is the honest answer.

**How it fails**

**How presigned URLs go wrong**

| Failure | Cause | Fix |
|---|---|---|
| One user overwrites another's object | Client-supplied key | Server constructs every key |
| Unexpected content uploaded | No content-type or size constraint | Constrain both in the signature |
| Objects uploaded but never recorded | Client never confirmed | Storage event notifications; reconciliation |
| URLs shared and reused | Long expiry | Minutes, not hours |
| Sensitive content leaked via logs | URLs recorded in access logs or referrers | Short expiry; per-request signing |
| Storage costs from orphans | Unconfirmed uploads never cleaned | Lifecycle rules and reconciliation |
| Malware stored and served | No scanning after upload | Asynchronous scan before making the object available |

> **Objects that were uploaded but never confirmed become invisible orphans**  
> Because the application never handles the bytes, it learns that an upload succeeded only if the client tells it — and clients close tabs, lose connectivity and crash. The object exists in storage, is billed, and has no database record pointing to it, so it is absent from every view the application provides. Storage event notifications close this properly; without them, periodic reconciliation between storage and the database is the minimum.

**Where it is used**

- **Media upload paths**, letting large files bypass the application tier entirely.
- **Protected downloads**, issuing short-lived links for paid or private content.
- **Mobile applications**, granting storage access without embedding credentials.
- **Third-party integrations**, allowing a partner to deliver files without an account.
- **Multipart upload flows**, signing individual parts for direct transfer.

**In the interview**

Frame it as a capability with constraints, and cover the confirmation gap.

- **Quantify the bandwidth saving**, since sizing the API tier for upload traffic is the problem being solved.
- **List what the signature constrains** — method, key, expiry, content type, size — and note that anything unconstrained is permitted.
- **Insist the server generates the key**, because a client-supplied path allows overwriting other objects.
- **Raise the confirmation gap** and propose storage event notifications rather than trusting the client.
- **Keep expiry short** and explain that the URL is a bearer capability, not an authenticated session.

**Practice drill**

Design a photo upload flow where clients send files directly to storage. Specify what the client requests, what the server checks, and exactly what the signature constrains. Describe how the application learns that an upload completed, and what happens if the client disappears midway. Then extend the design to protected downloads and state the expiry you would choose for each, with reasons.

**Go deeper**

A presigned URL is a time-limited signed capability granting one specific storage operation, letting clients transfer bytes directly while the application retains authorisation.

**It separates the decision from the data path, which is the whole point.** The application applies its rules — ownership, quota, content type, plan limits — once, in advance, and then steps out of the way. Bytes flow between the client and storage, so the API tier is sized for the requests it actually handles rather than for the bandwidth of everything users upload, which for media platforms is a difference of orders of magnitude.

**A signature is a capability, not an identity.** It does not know who is using it, so anyone holding the URL can perform the signed operation until it expires — which makes expiry the primary control and argues for minutes rather than hours. It also argues against embedding download URLs in pages that may be cached, forwarded or logged, since each of those copies is an equally valid capability for as long as it lasts.

**Everything the signature does not constrain is permitted.** A URL signed for an upload with no content type and no size limit allows any content of any size at that key, so constraints are not hardening but part of the authorisation itself. Most importantly the key must be generated server-side from the authenticated identity: accepting a client-supplied path lets a caller request a signature for someone else's object and overwrite it, and the storage service will honour it because the signature is valid.

**Not handling the bytes means not knowing the upload happened.** The application learns of completion only if the client reports it, and clients close tabs, lose connectivity and crash — leaving objects that exist in storage, accrue cost, and have no record pointing to them. Storage event notifications remove the dependency on the client entirely, and where they are unavailable, periodic reconciliation between storage contents and database records is the minimum acceptable substitute.

**Content inspection becomes asynchronous, which is a real change in guarantee.** Virus scanning, moderation and validation can all run after an object appears, but they run against something that has already been stored — so a design requiring content to be rejected before storage must proxy instead. In practice the usual arrangement is to upload to a quarantine location, scan, and only then move the object into a location the application will serve from.

**It composes with multipart uploads to remove the application from large transfers entirely.** Signing individual parts lets a multi-gigabyte upload proceed in parallel, resumably, straight into storage, with the application involved only in authorising and recording. That combination is what allows a modest API tier to support a platform whose ingest volume would otherwise dictate its entire architecture.

**Related patterns:** Object Storage · Multipart Upload · CDN · API Gateway

---

## Storage Internals

### Write Ahead Log

*Append every change to a sequential log and make it durable before applying it, so a crash can be recovered by replaying what was recorded.*

> **When you hear…** Durability required without paying random write cost · crash recovery to a consistent state · the foundation of transactional storage

**Flow:** `Change requested` → `Append to log` → `Flush to disk` → `Apply in memory` → `Replay on recovery`

**The problem**

Committing a transaction means its effects must survive a crash, and those effects are scattered across pages all over the data file. Writing every touched page synchronously turns one logical change into many random writes, which is the slowest thing storage does.

It is also not atomic. A crash partway through writing those pages leaves some updated and some not, with nothing recording which — so the data file is in a state that no correct sequence of operations could have produced.

> **Record the intention sequentially, apply it lazily**  
> If the change is first appended to a log that is written in order, durability costs one sequential write regardless of how many pages the change touches. The data pages can then be updated in memory and flushed whenever convenient, because a crash is recovered by replaying the log — the log is the authority, and the data file is a cache of what the log already says.

**Mental model**

An append-only sequence of change records, flushed before the change is acknowledged, replayed from the last checkpoint after a crash.

1. **Append** — The change is written to the end of the log.
2. **Flush** — The log write is forced to durable storage.
3. **Acknowledge** — Only now is the change reported as committed.
4. **Apply** — Data pages are updated in memory and flushed later.
5. **Recover** — After a crash, the log is replayed from the last checkpoint.

> **The durability guarantee is exactly as good as the flush**  
> A log write that reaches the operating system's cache but not the physical device is lost in a power failure, and the system will have acknowledged a commit that no longer exists. Every layer between the application and the platter — the filesystem, the drive's own write cache, a virtualised disk — can hold data that appears written, which is why correctness depends on an actual durable flush rather than a successful write call.

**How it works**

**Log records, checkpoints and recovery**

```text
RECORD
  LSN 1042 | txn 77 | page 813 | before | after
  -> before and after images allow both redo and undo

COMMIT PROTOCOL
  append records for the change
  append a commit record
  FLUSH the log
  acknowledge to the client
  -> apply to data pages whenever convenient

WHY THIS IS FAST
  one sequential append, one flush
  instead of N random page writes
  sequential throughput is 10-100x random on
  spinning media, and still several times higher on
  flash

CHECKPOINT
  periodically: flush dirty pages, record a
  checkpoint in the log
  -> recovery replays only from that point
  -> without checkpoints, recovery replays the
     entire history

RECOVERY
  find the last checkpoint
  REDO every change after it (committed or not)
  UNDO changes belonging to transactions that never
  committed
  -> the result is exactly the committed state at
     the moment of the crash

GROUP COMMIT
  many transactions flush together
  -> one disk flush serves dozens of commits
  -> adds a little latency, multiplies throughput
```

1. **Flush the log before acknowledging** — A commit acknowledged from a buffer is not durable.
2. **Checkpoint regularly** — Recovery time is proportional to the log since the last one.
3. **Use group commit under load** — Amortising one flush across many transactions is the main throughput lever.
4. **Keep before and after images** — Redo alone cannot roll back uncommitted work.
5. **Retain log segments until the changes are applied** — Truncating too early makes recovery impossible.
6. **Verify that flushes actually reach the device** — Caches at several layers can acknowledge without persisting.

**The trade-offs the log creates**

```text
RECOVERY TIME vs STEADY-STATE COST
  frequent checkpoints
    -> short recovery
    -> more page flushing during normal operation
  rare checkpoints
    -> minimal interference
    -> recovery may take many minutes
  -> choose from how long an outage may last

WRITE AMPLIFICATION
  every change is written twice: once to the log,
  once to the data file
  -> the log write is cheap (sequential)
  -> the data write happens once per page rather
     than once per change, so batching recovers most
     of it

LOG AS A REPLICATION SOURCE
  the same records that recover a crash can be
  shipped to replicas
  -> physical replication is just streaming the log
  -> change data capture reads it too
  -> one mechanism, three uses

IF THE LOG DISK FILLS
  no new writes can be made durable
  -> the database stops accepting writes
  -> this is a very common production outage, and it
     is usually caused by a replication slot or a
     long transaction preventing truncation
```

| Metric | Value | Note |
|---|---|---|
| Durability cost | one sequential flush | **not N random writes** |
| Recovery | replay from checkpoint | bounded by interval |
| Group commit | one flush, many txns | throughput lever |
| Failure | log disk full | writes stop |

> **A full log disk stops the database entirely**  
> Nothing can be made durable once the log cannot grow, so writes fail immediately even though the data disk has space. The usual causes are indirect — a replication slot no longer being consumed, or a very old open transaction preventing old segments from being removed — which means the disk fills for reasons unrelated to write volume and the remedy is rarely simply adding space.

**Technologies**

| System | Name | Note |
|---|---|---|
| Relational databases | WAL, redo log, transaction log | The mechanism underlying durability and recovery |
| LSM-tree stores | Commit log before the memtable | Same role, different storage engine |
| Filesystems | Journals | Metadata consistency by the same principle |
| Distributed logs | Replicated log as the system of record | Consensus protocols build on this idea |
| Replication | Log shipping | Physical replication is log streaming |
| Change data capture | Reading the log | Turns durability machinery into an event source |

The last three rows are the same artefact serving different purposes: a sequential record of every change is exactly what recovery, replication and change capture all need, which is why one mechanism ends up underpinning all three.

**Trade-offs**

**Durability strategies**

| Strategy | Commit cost | Recovery | Data loss on crash |
|---|---|---|---|
| Write pages synchronously | Many random writes | Complex; may be inconsistent | None if complete |
| Write-ahead log with flush | One sequential flush | Replay from checkpoint | None |
| Log with group commit | Amortised flush | Replay from checkpoint | None |
| Log without flush | Buffered write | Replay what survived | Recent commits |
| No log | None | Impossible | Everything unflushed |

The fourth row is a real configuration choice, not a mistake: some systems deliberately relax the flush for workloads where losing the last second of writes is acceptable in exchange for a large throughput gain. It should be a decision someone made, not a default nobody examined.

> **Ask before choosing it**  
> How much recently acknowledged data may be lost in a power failure? Zero requires a genuine flush per commit or a synchronous replica; anything else is a deliberate trade, and the answer determines both the commit protocol and the replication configuration.

**How it fails**

**How write-ahead logging goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Committed data lost in a power failure | Flush not reaching the device | Verify durability through the whole stack |
| Writes stop entirely | Log disk full | Monitor log growth; find what blocks truncation |
| Recovery takes many minutes | Checkpoints too infrequent | Checkpoint more often; accept steady-state cost |
| Throughput limited by flushes | One flush per commit | Group commit |
| Recovery impossible | Log truncated before pages were applied | Retain segments until checkpointed |
| Replica falls behind and blocks truncation | Unconsumed replication slot | Monitor slot lag; bound retention |
| Long transactions block log cleanup | A very old open transaction | Alert on transaction age |

> **A durability guarantee that does not survive power loss is not a durability guarantee**  
> Write caches in the operating system, the filesystem, the drive controller and any virtualisation layer can each report success before data is physically persisted. A system configured for full durability on hardware that lies about flushes will lose acknowledged commits in a power failure, and nothing in normal operation reveals it — the guarantee has to be verified against the actual storage stack rather than assumed from the database's configuration.

**Where it is used**

- **Every relational database**, where the log is the basis of commit, rollback and recovery.
- **LSM-tree stores**, which write to a commit log before the in-memory table.
- **Journaling filesystems**, applying the same principle to metadata consistency.
- **Consensus systems**, replicating a log as the authoritative sequence of operations.
- **Change data capture pipelines**, reading the log to emit events downstream.

**In the interview**

The sequential-versus-random argument is the core insight; make it explicit.

- **State why it is faster**: one sequential flush instead of many random page writes, whatever the change touched.
- **Give the commit order** — append, flush, acknowledge — and say the guarantee lives in the flush.
- **Explain checkpoints as the recovery-time control**, trading steady-state cost against outage length.
- **Mention group commit**, since amortising flushes is the main throughput lever under load.
- **Connect it to replication and change capture**, because the same log serves all three.

**Practice drill**

Explain how a database survives a power failure midway through a transaction. Describe the exact sequence at commit time and state where durability is established. Then describe recovery: what is replayed, what is rolled back, and how the checkpoint interval affects how long it takes. Finally, explain why a replication slot that nobody is reading can stop the database from accepting writes.

**Go deeper**

A write-ahead log records every change sequentially and durably before it is applied, so a crash is recovered by replaying the log.

**It converts random writes into sequential ones, which is the entire performance argument.** A change touching many pages would otherwise require many random writes to be durable; appending a description of that change costs one sequential write and one flush regardless of how scattered its effects are. Sequential throughput exceeds random by a wide margin on every storage medium, so this single change makes durable commits affordable.

**The log becomes the authority and the data file becomes a cache.** Because recovery replays the log, data pages can be updated lazily and flushed in batches whenever convenient, which also means multiple changes to the same page cost one eventual write rather than one each. The apparent write amplification of recording everything twice is largely recovered by that batching.

**Durability is established at the flush and nowhere else.** A commit acknowledged from a buffer is a promise the system cannot keep, and every layer between the application and the physical device — filesystem, drive cache, virtualised storage — can report success before persisting. This makes the guarantee a property of the whole stack rather than of the database configuration, and verifying it requires testing power loss rather than reading settings.

**Checkpoints are the dial between steady-state cost and recovery time.** Replay begins at the last checkpoint, so frequent checkpoints keep recovery short at the cost of more page flushing during normal operation, and infrequent ones do the reverse. The correct setting comes from how long an outage may acceptably last, which is an availability decision rather than a performance one.

**Group commit is what makes high transaction rates possible.** Flushing once per commit bounds throughput by the device's flush rate, whereas batching many commits into one flush amortises that cost across all of them, adding a small amount of latency and multiplying throughput. It is the standard answer when commit rate rather than data volume is the constraint.

**Its most common production failure is the log disk filling for indirect reasons.** Nothing can be made durable once the log cannot grow, so writes stop entirely even with abundant space in the data files — and the cause is usually a replication slot nobody is consuming or a very old open transaction preventing old segments from being released. Because the trigger is unrelated to write volume, monitoring log growth alongside replication lag and transaction age is what turns this from an outage into an alert.

**Related patterns:** MVCC · LSM Tree · Change Data Capture · Two Phase Commit

---

### MVCC

*Keep multiple versions of each row so readers see a consistent snapshot without blocking writers, and writers never block readers.*

> **When you hear…** Long-running reads blocking writes · lock contention between analytics and transactions · snapshot isolation required

**Flow:** `Write creates a version` → `Readers get a snapshot` → `No read locks` → `Old versions retained` → `Vacuum reclaims`

**The problem**

A report scanning a large table holds read locks for its duration, so every write touching those rows waits. Meanwhile a writer holding row locks blocks the report. Readers and writers obstruct each other constantly, and the contention grows with both concurrency and query duration.

Reading without locks is worse: the report sees rows as they change underneath it, producing totals that correspond to no actual state of the database at any point in time.

> **Do not overwrite — create a new version and let readers choose**  
> If an update writes a new version of a row rather than replacing the old one, then a reader can continue seeing the version that was current when it started. Readers need no locks because nothing they are reading changes, and writers need not wait because they are writing somewhere else. The cost is storing versions and eventually reclaiming the ones nobody can see.

**Mental model**

Every row version is stamped with the transaction that created it and the one that removed it. A transaction reads the versions that were visible when its snapshot was taken.

1. **Snapshot** — A transaction records which other transactions had committed when it began.
2. **Write** — An update creates a new version and marks the old one as superseded.
3. **Read** — Each reader sees the version appropriate to its snapshot.
4. **Isolate** — Concurrent transactions see consistent, possibly different, states.
5. **Reclaim** — Versions no transaction can see are eventually removed.

> **Old versions accumulate and must be reclaimed, or the table degrades**  
> Every update leaves a dead version behind, and a table receiving heavy updates can grow far beyond the size of its live data. Scans then read mostly obsolete rows, and performance falls steadily in a way that looks like a query problem. Reclamation is background work that must keep pace with the write rate, and when it cannot, the degradation is gradual and easy to misattribute.

**How it works**

**Visibility and version accumulation**

```text
ROW VERSIONS
  row 42:
    version A  created by txn 100, deleted by 150
    version B  created by txn 150, deleted by null

VISIBILITY RULE  (simplified)
  a version is visible to a transaction if:
    its creating transaction committed before the
    snapshot, AND
    its deleting transaction had not committed
    before the snapshot
  -> readers take no locks at all

WRITE CONFLICTS STILL EXIST
  two transactions updating the same row
  -> the second waits, or fails, depending on the
     isolation level
  -> MVCC removes read-write contention, not
     write-write

VERSION ACCUMULATION
  a table updated 1,000 times per second
  vacuum running every few minutes
  -> hundreds of thousands of dead versions between
     runs
  -> the table occupies several times its live size
  -> scans read mostly dead rows

THE LONG TRANSACTION PROBLEM
  one transaction open for hours holds a snapshot
  -> NO version newer than it can be reclaimed
  -> dead rows accumulate across the whole database
  -> a single forgotten session degrades everything
```

1. **Keep transactions short** — An open transaction blocks reclamation database-wide, not just for its own rows.
2. **Monitor table bloat and reclamation lag** — Degradation is gradual and presents as slow queries.
3. **Alert on the oldest open transaction** — It is the single most common cause of runaway version accumulation.
4. **Understand the isolation level in use** — Snapshot isolation is not serializable and permits specific anomalies.
5. **Expect write-write conflicts to remain** — MVCC eliminates read-write blocking only.
6. **Size storage for versions, not just live data** — Update-heavy tables occupy several times their logical size.

**What MVCC does and does not guarantee**

```text
PROVIDES
  readers never block writers
  writers never block readers
  a consistent snapshot for the whole transaction
  no read locks at all

DOES NOT PROVIDE
  serializability, unless explicitly requested
  freedom from write-write conflicts
  free storage

WRITE SKEW  (the classic snapshot anomaly)
  rule: at least one doctor must remain on call
  two doctors, both on call
  both transactions read: "2 on call, fine"
  both remove themselves
  both commit
  -> zero on call; each decision was valid against
     its own snapshot
  -> snapshot isolation permits this
  -> serializable isolation detects and prevents it

VERSION STORAGE STRATEGIES
  versions in the table   updates leave dead rows in
                          place; vacuum reclaims
  versions in undo space  the table stays compact;
                          long readers can exhaust
                          undo space instead
  -> different systems choose differently, and the
     operational failure modes differ accordingly
```

| Metric | Value | Note |
|---|---|---|
| Readers | never blocked | **no read locks** |
| Writers | never blocked by readers | write-write still conflicts |
| Cost | version storage | plus reclamation |
| Anomaly | write skew | under snapshot isolation |

> **One long-running transaction prevents reclamation across the entire database**  
> Because any version potentially visible to an open snapshot cannot be removed, a single session left open — an idle transaction, a stalled report, a forgotten console — stops cleanup everywhere, not only for the rows it touched. Dead versions then accumulate globally, tables bloat, and performance degrades across unrelated workloads, which makes the age of the oldest open transaction one of the most valuable metrics to alert on.

**Technologies**

| System | Approach | Note |
|---|---|---|
| PostgreSQL | Versions stored in the table | Vacuum reclaims; bloat is the operational concern |
| Oracle and MySQL InnoDB | Versions in undo segments | Table stays compact; undo space can be exhausted |
| Distributed SQL engines | MVCC with timestamps | Snapshots across nodes require clock coordination |
| Key-value stores with versions | Timestamped values | Same principle, simpler visibility rules |
| Two-phase locking | The alternative | Serializable by construction; far more blocking |
| Serializable snapshot isolation | MVCC plus conflict detection | Serializability without most of the locking |

Serializable snapshot isolation is the significant refinement: it keeps the non-blocking reads of MVCC while detecting the dependency patterns that produce anomalies, aborting one transaction when a genuine conflict occurs rather than preventing concurrency in advance.

**Trade-offs**

**Concurrency control approaches**

| Approach | Reader blocking | Storage | Anomalies |
|---|---|---|---|
| Two-phase locking | Readers block writers | Minimal | None at serializable |
| MVCC snapshot isolation | None | Versions retained | Write skew possible |
| Serializable snapshot isolation | None | Versions plus tracking | None; some aborts |
| Read uncommitted | None | Minimal | Many |
| Optimistic concurrency | None | Minimal | Aborts on conflict |

The middle row is what most systems run by default, and the anomalies it permits are subtle enough that many applications contain latent write-skew bugs which appear only under concurrency that the test suite never produces.

> **Ask before choosing it**  
> Does any business rule depend on a condition across rows that a transaction checks and then acts on? That is the shape of write skew, and under snapshot isolation two concurrent transactions can both pass the check and jointly violate the rule — which needs either serializable isolation or an explicit lock.

**How it fails**

**How MVCC goes wrong**

| Failure | Cause | Fix |
|---|---|---|
| Tables grow far beyond their data | Reclamation not keeping pace | Tune vacuum; reduce update rate |
| Global performance degradation | A long-open transaction blocking cleanup | Alert on oldest transaction age |
| Queries slow down gradually | Scans reading mostly dead versions | Reclaim; rebuild badly bloated tables |
| Business rules violated under load | Write skew under snapshot isolation | Serializable isolation, or explicit locking |
| Undo space exhausted | Long readers in undo-based systems | Bound query duration; size undo space |
| Unexpected serialization failures | Serializable isolation aborting conflicts | Retry aborted transactions |
| Write conflicts surprising developers | Expecting MVCC to remove all blocking | It removes read-write blocking only |

> **Write skew produces violations that every individual transaction considered legal**  
> Two transactions each read a condition, each conclude their action is permitted, and each commit — leaving a state neither would have allowed had it seen the other. Snapshot isolation permits this by design, so the bug is not in the database and not obviously in the application either. It surfaces only under concurrency, frequently in production, and the remedies are serializable isolation or taking an explicit lock on whatever the rule ranges over.

**Where it is used**

- **PostgreSQL**, where MVCC and vacuum behaviour are central operational concerns.
- **MySQL InnoDB**, using undo segments for versions and consistent reads.
- **Distributed SQL databases**, extending snapshots across nodes with coordinated timestamps.
- **Analytics against transactional databases**, reading consistently without blocking writers.
- **Key-value stores with versioned values**, applying the same visibility principle.

**In the interview**

Say what it removes and what it does not; the write-skew example is what demonstrates real understanding.

- **State the guarantee precisely**: readers never block writers and vice versa, but write-write conflicts remain.
- **Explain visibility by snapshot**, which is why readers need no locks at all.
- **Give the write-skew example** with the on-call doctors, since it shows the limits of snapshot isolation concretely.
- **Raise version accumulation** and the operational cost of reclamation.
- **Name the long-transaction hazard**, where one open session degrades the entire database.

**Practice drill**

Explain how a report scanning ten million rows for two minutes coexists with a thousand updates per second on the same table. Describe what each sees and why neither waits. Then construct a write-skew scenario for a rule requiring at least one administrator per account, show why both transactions commit, and give two different ways to prevent it. Finally, explain what a session left open for six hours does to the database.

**Go deeper**

Multiversion concurrency control keeps several versions of each row so that readers see a consistent snapshot without locking and writers proceed without waiting for readers.

**It removes the largest source of contention in mixed workloads.** Analytical queries and transactional writes obstruct each other severely under lock-based concurrency control, and MVCC eliminates that interaction entirely: readers observe versions that were current when they started, so nothing they read can change, and writers create new versions rather than modifying what readers hold. Write-write conflicts remain, because two transactions genuinely cannot both decide the next value of the same row.

**Versions are the currency, and they must be reclaimed.** Every update leaves an obsolete version, so an update-heavy table accumulates dead rows continuously and can occupy several times the size of its live data. Scans then spend most of their time reading versions nobody can see, which presents as gradually worsening query performance rather than as a storage problem — making bloat and reclamation lag metrics worth watching directly.

**A single long transaction degrades everything.** Reclamation can only remove versions that no open snapshot could see, so one session held open for hours prevents cleanup across the whole database, not merely for the rows it touched. An idle transaction, a stalled report or a forgotten console therefore causes global accumulation, and the age of the oldest open transaction is among the highest-value alerts a system of this kind can have.

**Snapshot isolation is weaker than serializability in a specific and non-obvious way.** Write skew occurs when two transactions each read a condition, each conclude independently that their action is permitted, and together violate the rule neither would have broken alone. Nothing is wrong with either transaction in isolation and nothing is wrong with the database; the anomaly is inherent to reading from a snapshot, which makes it a latent bug in applications whose invariants span rows.

**Where versions are stored changes the operational failure mode.** Keeping them in the table means dead rows accumulate in place and require vacuuming, so the characteristic problem is bloat. Keeping them in a separate undo area keeps tables compact and makes long-running readers a different hazard — they can exhaust undo space and fail, or force other transactions to fail. Neither is better in general, but they fail differently, and the monitoring that matters differs accordingly.

**Serializable snapshot isolation resolves the anomaly without returning to locking.** By tracking read and write dependencies and aborting a transaction when a genuine conflict pattern is detected, it preserves non-blocking reads while providing full serializability, at the cost of occasional aborts that the application must retry. For workloads with invariants spanning rows this is usually a better trade than either accepting write skew or reintroducing explicit locks everywhere.

**Related patterns:** Write Ahead Log · LSM Tree · Consistent Snapshot · Quorum Replication

---

### Bloom Filter

*A compact probabilistic structure that answers definitely-not-present or possibly-present, letting a system skip expensive lookups for keys it does not have.*

> **When you hear…** Expensive lookups for keys that usually do not exist · memory too small for a full index · disk or network reads worth avoiding

**Flow:** `Hash the key` → `Set several bits` → `Query checks bits` → `Any zero means absent` → `All ones means maybe`

**The problem**

Checking whether a key exists means consulting an index that does not fit in memory, so most checks cost a disk read — and in many workloads the majority of those reads discover that the key is absent. The expensive operation was performed to learn nothing.

Keeping a complete set of keys in memory would answer instantly and is exactly what does not fit; that is why the index is on disk in the first place.

> **Accept false positives and the memory requirement collapses**  
> A structure that may occasionally say a key might be present when it is not, but never says absent when it is present, can be built from a handful of bits per key rather than the key itself. Since a definite no eliminates the lookup entirely and a maybe simply proceeds as before, the error is asymmetric in exactly the direction that costs nothing but a wasted check.

**Mental model**

A bit array and several hash functions. Insertion sets the bits a key hashes to; a query reports absent if any of those bits is zero.

1. **Size** — Choose the bit array size and hash count from the expected key count and acceptable error rate.
2. **Insert** — Hash the key several ways and set each resulting bit.
3. **Query** — Hash the same ways and inspect those bits.
4. **Conclude** — Any zero proves absence; all ones means probably present.
5. **Verify** — A probable hit is confirmed by the real lookup.

> **Elements cannot be removed from a standard Bloom filter**  
> Clearing the bits for one key would also clear bits shared with others, producing false negatives — which destroys the one guarantee the structure provides. Deletion requires a counting variant with several bits per position, or rebuilding the filter, and a design that assumes removal works will silently start reporting that present keys are absent.

**How it works**

**Sizing and the error rate**

```text
PARAMETERS
  n  expected number of elements
  m  bits in the array
  k  number of hash functions

OPTIMAL k = (m/n) x ln 2
FALSE POSITIVE RATE ≈ (1 - e^(-kn/m))^k

PRACTICAL SIZING  (bits per element)
  ~4.8 bits  -> 10% false positives
  ~9.6 bits  -> 1%
  ~14.4 bits -> 0.1%
  ~19.2 bits -> 0.01%
  -> each additional ~4.8 bits per element divides
     the error rate by ten

CONCRETE
  1,000,000 keys at 1% error
    9.6 Mbit = 1.2 MB, with k = 7
  compare storing the keys themselves:
    1,000,000 x 20 B = 20 MB
  -> ~17x smaller, and constant regardless of key
     length

THE LONG KEY ADVANTAGE
  a 200-byte URL still costs 9.6 bits
  -> the saving grows with key size

OVERFILLING
  inserting far more than n elements raises the
  error rate towards 100%
  -> the filter degrades into always saying maybe
  -> size for the eventual count, not the current one
```

1. **Size from the eventual element count** — An overfilled filter degrades until it answers maybe to everything.
2. **Choose the error rate from the cost of a false positive** — It is a wasted lookup, so the acceptable rate depends on that lookup's expense.
3. **Use one hash with different seeds** — Computing several independent hashes is unnecessary and slower.
4. **Never rely on removal** — Standard filters cannot delete; use a counting variant or rebuild.
5. **Keep it in memory** — A filter on disk defeats its own purpose.
6. **Monitor the observed false positive rate** — It reveals overfilling before performance degrades noticeably.

**Where the asymmetry pays**

```text
LSM-TREE READS  (the canonical use)
  a read may need to check many sorted files
  each check is a disk read
  a filter per file:
    definitely not here -> skip the file entirely
    maybe               -> read it
  -> most files are skipped
  -> read amplification falls dramatically

CACHE ADMISSION
  do not cache an item until it has been seen twice
  -> a filter records "seen once" cheaply
  -> prevents one-off scans evicting hot data

DISTRIBUTED SET MEMBERSHIP
  send a filter instead of a key set
  -> a peer can check membership locally
  -> anti-entropy and replica comparison use this

WHEN IT DOES NOT HELP
  most queries are for keys that DO exist
  -> the filter says maybe almost every time
  -> you paid memory and saved nothing
  -> the benefit is proportional to the miss rate

THE DECIDING QUESTION
  what fraction of lookups are for absent keys?
  high  -> a filter is extremely effective
  low   -> it is overhead
```

| Metric | Value | Note |
|---|---|---|
| Memory | ~10 bits/key | **at 1% error** |
| Guarantee | no false negatives | absent is certain |
| Benefit | scales with miss rate | useless if all hit |
| Limitation | no deletion | without a variant |

> **The benefit is proportional to how often keys are absent**  
> A filter earns its memory by eliminating lookups, so a workload whose keys nearly always exist receives a maybe on almost every query and skips nothing. The structure is then pure overhead — memory spent and a hash computed per lookup for no saving. Measuring the miss rate before adding one is what separates a large improvement from a small regression.

**Technologies**

| Use | System | Note |
|---|---|---|
| LSM-tree read paths | Cassandra, RocksDB, HBase | One filter per sorted file; the defining use |
| Cache admission | Modern cache implementations | Prevents one-off items evicting hot data |
| Replica comparison | Anti-entropy protocols | Exchange filters rather than key sets |
| Counting Bloom filter | Where deletion is required | Several bits per position; more memory |
| Cuckoo filter | Supports deletion | Comparable size, better locality |
| Exact structures | Hash sets | When memory allows, and certainty is needed |

Cuckoo filters are worth knowing as the modern alternative: they support deletion, achieve similar or better space efficiency at low error rates, and access fewer memory locations per query — which matters when the filter is consulted on every read.

**Trade-offs**

**Membership testing structures**

| Structure | Memory per key | False positives | Deletion |
|---|---|---|---|
| Hash set of keys | Key size plus overhead | None | Yes |
| Bloom filter | ~10 bits at 1% | Yes | No |
| Counting Bloom filter | ~40 bits at 1% | Yes | Yes |
| Cuckoo filter | ~12 bits at 1% | Yes | Yes |
| Sorted key index on disk | Key size | None | Yes |

The counting variant's fourfold memory cost is the price of deletion, which is often enough to change the decision — rebuilding a standard filter periodically is frequently cheaper than paying that overhead continuously.

> **Ask before choosing it**  
> What fraction of lookups are for keys that do not exist, and what does one lookup cost? Those two numbers determine the entire value of the structure, and if most lookups hit, no amount of tuning will make it worthwhile.

**How it fails**

**How Bloom filters go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Error rate approaching 100% | Far more elements than the filter was sized for | Size for the eventual count; rebuild when exceeded |
| Present keys reported absent | Deletion attempted on a standard filter | Counting or cuckoo variant, or rebuild |
| No measurable improvement | Low miss rate in the workload | Measure before adopting; remove if unhelpful |
| Memory larger than expected | Error rate set unnecessarily low | Relax it; the cost is a wasted lookup |
| Slow queries | Too many hash functions | Use the optimal count for the chosen ratio |
| Filter itself becoming a bottleneck | Stored on disk or across the network | It must be in memory to be worthwhile |
| Degradation unnoticed | Observed error rate not tracked | Monitor it and rebuild on a threshold |

> **Attempting deletion turns the one reliable answer into an unreliable one**  
> Clearing bits for a removed key also clears bits that other keys depend on, so those keys begin reporting as definitely absent — and because the entire value of the structure rests on absence being certain, the system will skip lookups for data it actually holds. The result is silent data loss from the caller's perspective, with no error anywhere, which makes this the most damaging way to misuse the structure.

**Where it is used**

- **LSM-tree storage engines**, where a filter per sorted file eliminates most disk reads on lookups.
- **Distributed databases**, avoiding network round trips for keys a node does not hold.
- **Cache admission policies**, keeping one-off items from displacing frequently used data.
- **Anti-entropy protocols**, comparing replica contents by exchanging filters.
- **Web and security systems**, checking membership in large lists without storing them.

**In the interview**

Give the space figure and the asymmetry; both are concrete and immediately convincing.

- **State the guarantee precisely**: no false negatives, so a negative answer is certain and a positive one is a maybe.
- **Quote the space cost** — roughly ten bits per element for one per cent error — and note it is independent of key length.
- **Give the LSM-tree use** as the canonical example, since it shows exactly where the asymmetry pays.
- **Say deletion is not supported** and what happens if it is attempted.
- **Tie the benefit to the miss rate**, because a workload that mostly hits gains nothing.

**Practice drill**

A storage engine checks 20 sorted files per read, each check costing a disk seek, and 80% of reads are for keys that exist in only one file. Size a filter per file for a million keys at 1% error and compute the memory. Estimate the reduction in disk reads. Then explain what happens if a file accumulates ten times the expected keys, and what you would monitor to detect it.

**Go deeper**

A Bloom filter is a compact probabilistic set that never reports a present element as absent, allowing expensive lookups to be skipped with certainty when it says no.

**Its usefulness comes entirely from the asymmetry of its error.** A false positive costs one unnecessary lookup — exactly what would have happened without the filter — while a negative answer is definitive and eliminates the lookup completely. Because the errors only ever cost what the system was already going to pay, the acceptable error rate can be set quite loosely, which is what makes the memory requirement so small.

**Space efficiency comes from storing no keys at all.** Roughly ten bits per element achieves a one per cent error rate regardless of whether the keys are short integers or long URLs, so the saving against storing the keys grows with key size. Each additional five bits or so divides the error rate by ten, which makes the memory-versus-accuracy curve easy to reason about when sizing.

**The benefit is proportional to the miss rate, and this is the question to ask first.** A workload whose lookups almost always find the key receives a maybe nearly every time and skips nothing, so the filter is memory and hashing spent for no return. Measuring how often lookups are for absent keys before adopting one distinguishes a substantial improvement from a small regression, and it is a measurement rather than an intuition.

**Overfilling degrades it gradually into uselessness.** Inserting far more elements than the sizing assumed drives the error rate towards certainty, at which point the filter answers maybe to everything and the system performs exactly as it would without it — while still paying the memory. Because the degradation is smooth and silent, tracking the observed false positive rate is what reveals it before someone investigates a performance regression with no obvious cause.

**Deletion is not merely unsupported; attempting it breaks the guarantee.** Clearing a key's bits also clears bits other keys rely on, so those keys start reporting as definitely absent and the system skips lookups for data it holds. This converts the structure's one reliable answer into an unreliable one and manifests as data that exists but cannot be found, with no error raised — which is why counting variants, cuckoo filters, or periodic rebuilds are the only correct approaches when removal is needed.

**Its defining application is the LSM-tree read path.** A lookup may need to consult many sorted files, each costing a disk read, and most of them do not contain the key. One filter per file turns the majority of those checks into memory operations that skip the file entirely, which is the difference between a read amplification of twenty and one closer to one — and it is the reason the structure appears in essentially every storage engine of that design.

**Related patterns:** LSM Tree · Cache Aside · Anti Entropy · Inverted Index

---

### LSM Tree

*Buffer writes in memory, flush them as sorted immutable files, and merge those files in the background, turning random writes into sequential ones.*

> **When you hear…** Write-heavy workloads · random writes limiting throughput · time-series, logs and event data

**Flow:** `Write to memtable` → `Flush sorted file` → `Files accumulate` → `Compaction merges` → `Reads check several`

**The problem**

A B-tree updates data in place, so every write touches the page holding that key — a random write. At high write rates the device spends its time seeking rather than transferring, and throughput is bounded by random I/O rather than by bandwidth.

Writes also amplify: changing a few bytes rewrites a whole page, and page splits rewrite more. For workloads dominated by inserts and updates this overhead dominates everything else.

> **Never update in place; accumulate and merge**  
> If writes go into a sorted in-memory structure and are flushed as whole immutable files, every disk write is sequential and large. Nothing is ever modified in place, so there are no random writes at all. The cost moves to reads, which must consult several files, and to background merging that keeps the number of files bounded — both of which are more tractable than random write throughput.

**Mental model**

An in-memory sorted buffer, a sequence of immutable sorted files on disk, and a background process merging those files into fewer, larger ones.

1. **Log** — The write is appended to a commit log for durability.
2. **Buffer** — It is inserted into a sorted in-memory table.
3. **Flush** — When the table is full it is written out as an immutable sorted file.
4. **Accumulate** — Files build up, each containing a snapshot of some writes.
5. **Compact** — Background merging combines files, discarding superseded values.

> **Reads may have to consult many files, and compaction competes with live traffic**  
> A key may live in the memtable or in any of the files on disk, so a read potentially examines all of them. Bloom filters and sorted structure make this manageable, and compaction is what keeps the count bounded — but compaction reads and rewrites large amounts of data using the same disks serving queries, which is why LSM systems exhibit latency variability that B-trees do not.

**How it works**

**Write path, read path and compaction**

```text
WRITE
  append to commit log      (durability)
  insert into memtable      (sorted, in memory)
  return
  -> no disk seek, no page read
  -> extremely fast

FLUSH
  memtable full -> write it out as one sorted file
  -> one large sequential write
  -> the file is immutable forever

READ
  check memtable
  then each file, newest first
  stop at the first match
  -> without help this is many disk reads
  -> Bloom filter per file skips most of them
  -> sparse index locates the block within a file

COMPACTION STRATEGIES
  SIZE-TIERED
    merge files of similar size
    + low write amplification
    - more files to read, more space used
    -> good for write-heavy

  LEVELLED
    keep each level's files non-overlapping
    + few files per read, predictable
    - higher write amplification
    -> good for read-heavy

DELETES ARE WRITES
  a delete writes a TOMBSTONE
  -> the key is only truly removed when compaction
     has passed every file containing it
  -> deleting a lot of data temporarily uses MORE
     space
```

1. **Put a Bloom filter on every file** — Without it a read must touch every file on disk.
2. **Choose the compaction strategy from the workload** — Size-tiered favours writes, levelled favours reads.
3. **Throttle compaction** — It uses the same devices as live traffic and will starve it otherwise.
4. **Provision free space for compaction** — Merging requires room for the output before the inputs are released.
5. **Expect tombstones to delay space reclamation** — Deleted data persists until compaction has passed it everywhere.
6. **Monitor the file count and compaction backlog** — A growing backlog degrades reads steadily.

**Amplification and the three-way trade**

```text
THE THREE AMPLIFICATIONS
  write   bytes written to disk per byte of data
  read    files consulted per lookup
  space   disk used per byte of live data
  -> improving any one worsens at least one other

SIZE-TIERED
  write  low   (each byte rewritten a few times)
  read   high  (many files to check)
  space  high  (duplicates until merged)

LEVELLED
  write  high  (a byte may be rewritten ~10x)
  read   low   (one file per level)
  space  low   (little duplication)

B-TREE FOR COMPARISON
  write  high on random writes (page rewrites)
  read   low   (one path down the tree)
  space  moderate (fragmentation)

WHY LSM WINS ON WRITES
  all disk writes are sequential and large
  sequential throughput far exceeds random on every
  medium
  -> an LSM tree can absorb write rates a B-tree
     cannot, on the same hardware

COMPACTION IS THE OPERATIONAL RISK
  it must keep pace with writes
  falling behind -> file count grows -> reads slow
  -> and catching up competes with live traffic,
     which is when latency spikes appear
```

| Metric | Value | Note |
|---|---|---|
| Writes | sequential only | **no random I/O** |
| Reads | several files | Bloom filters help |
| Trade | write/read/space | pick two |
| Risk | compaction backlog | reads degrade |

> **Deleting large amounts of data temporarily increases disk usage**  
> A delete is written as a tombstone, so removing a million rows writes a million records and frees nothing until compaction has merged past every file containing the originals. A system already low on space can therefore fail while attempting to free space, which is counter-intuitive enough that it surprises operators during exactly the incident where they are trying to recover capacity.

**Technologies**

| System | Use | Note |
|---|---|---|
| RocksDB and LevelDB | Embedded storage engines | The reference implementations |
| Cassandra | Distributed store on an LSM engine | Compaction strategy is a per-table decision |
| HBase and Bigtable-style stores | Wide-column storage | Same design lineage |
| Time-series databases | Append-heavy workloads | An excellent fit for the write pattern |
| B-tree engines | The alternative | Better for read-heavy and range-scan workloads |
| Bloom filters | Companion structure | Essential to the read path |

The B-tree comparison is worth keeping concrete: B-trees remain preferable for read-heavy workloads with frequent range scans and in-place updates, while LSM trees dominate where writes are numerous and reads are mostly point lookups.

**Trade-offs**

**LSM against B-tree**

| Property | LSM tree | B-tree |
|---|---|---|
| Write pattern | Sequential only | Random, in place |
| Write throughput | Very high | Bounded by random I/O |
| Point reads | Several files; filters help | One tree traversal |
| Range scans | Merge across files | Sequential along leaves |
| Space | Duplicates until compacted | Fragmentation |
| Latency stability | Compaction causes variance | More predictable |

Latency stability is the row that decides many production choices: LSM engines deliver higher throughput with more variance, and a workload with strict tail-latency requirements may prefer a B-tree even where the write rate would favour an LSM tree.

> **Ask before choosing it**  
> Is this workload write-heavy with point lookups, or read-heavy with range scans? The first is exactly what LSM trees are for; the second is what B-trees do better, and choosing on write throughput alone ignores half the question.

**How it fails**

**How LSM trees go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Reads slowing over time | Compaction falling behind | More compaction throughput; revisit the strategy |
| Latency spikes | Compaction competing with queries | Throttle and schedule compaction |
| Disk full while deleting data | Tombstones consuming space before reclaiming it | Provision headroom; compact deliberately |
| Reads touching every file | No Bloom filters, or overfilled ones | Filters per file, sized correctly |
| Write amplification excessive | Levelled compaction on a write-heavy workload | Size-tiered instead |
| Space usage far above data size | Size-tiered with infrequent compaction | Levelled, or compact more often |
| Deleted data reappearing | Tombstones expiring before full compaction | Ensure repair and compaction outpace tombstone lifetime |

> **Compaction falling behind degrades reads in a way that compounds**  
> When merging cannot keep pace with writes, files accumulate, every read consults more of them, and the extra read I/O competes with the compaction that is already behind. The system slows gradually and then sharply, and catching up requires I/O that the live workload is consuming — which is why compaction backlog is one of the most important metrics in any LSM-based system and why throttling policy deserves attention before it becomes urgent.

**Where it is used**

- **Embedded storage engines**, used inside many databases and applications.
- **Cassandra and wide-column stores**, built directly on this design.
- **Time-series and metrics databases**, whose append-heavy pattern suits it exactly.
- **Event and log storage**, where writes vastly outnumber reads.
- **Key-value caches with persistence**, using the design for durable write throughput.

**In the interview**

Explain the sequential-write insight, then be explicit about the three-way amplification trade.

- **Say why writes are fast**: every disk write is sequential and large, because nothing is ever updated in place.
- **Describe the read path** — memtable then files, newest first — and the role of Bloom filters in making it viable.
- **Lay out write, read and space amplification** and say that improving one worsens another.
- **Contrast the compaction strategies** and tie each to a workload shape.
- **Raise tombstones and compaction backlog**, since both are operational realities that distinguish experience from theory.

**Practice drill**

Design storage for a metrics system ingesting 500,000 writes per second with occasional point lookups. Explain why an LSM tree suits it and what the write path costs. Choose a compaction strategy and justify it against the three amplifications. Then describe what happens operationally when compaction cannot keep up, and what you would monitor to see it coming.

**Go deeper**

An LSM tree buffers writes in memory, flushes them as immutable sorted files, and merges those files in the background, so all disk writes are sequential.

**Eliminating in-place updates is what makes it fast.** A B-tree modifies the page containing a key, which is a random write, and random throughput is far below sequential on every storage medium. Accumulating changes and writing whole sorted files means the device only ever performs large sequential writes, which is why an LSM engine can absorb write rates that a B-tree cannot on identical hardware.

**The cost is moved to reads and to background work.** A key may reside in the memtable or in any file on disk, so a lookup potentially consults all of them — made tractable by a Bloom filter per file that eliminates most checks, and by sorted structure with sparse indexes that locate blocks cheaply. Compaction then keeps the file count bounded, which is what prevents the read path from degrading without limit.

**Write, read and space amplification form a trade in which improving one worsens another.** Size-tiered compaction rewrites data few times but leaves many files and duplicate data, favouring writes; levelled compaction keeps few files per read and little duplication at the cost of rewriting each byte many times. Neither is correct in general, and choosing between them is the main tuning decision an LSM deployment presents.

**Compaction is the operational characteristic that distinguishes these systems.** It consumes the same devices serving live traffic, so it must be throttled to avoid starving queries and must still keep pace with writes — and when it falls behind, files accumulate, reads slow, and the additional read load competes with the compaction that is already late. The degradation compounds, which makes compaction backlog one of the most valuable metrics available and throttling policy something to establish before it is urgent.

**Deletes behave counter-intuitively and surprise operators.** A deletion is a written tombstone rather than a removal, so freeing space requires compaction to merge past every file containing the original values — and in the interim, deleting large volumes increases disk usage. A system already short of space can therefore fail while attempting to reclaim it, precisely when that recovery is most needed.

**It is a workload-shaped choice rather than a generally superior design.** Write-heavy workloads with point lookups — metrics, events, logs, time series — are exactly what it is for, and read-heavy workloads with frequent range scans and in-place updates remain better served by B-trees. Latency stability matters too: LSM engines trade predictable response times for throughput, so a system with strict tail-latency requirements may reasonably choose the lower-throughput structure.

**Related patterns:** Bloom Filter · Write Ahead Log · MVCC · Batch Processing

---

## Search

### Inverted Index

*Map each term to the documents containing it, so text search becomes a lookup and intersection rather than a scan.*

> **When you hear…** Text search over many documents · scans too slow for interactive queries · relevance ranking required

**Flow:** `Analyse text` → `Extract terms` → `Term to document lists` → `Intersect postings` → `Rank results`

**The problem**

Finding documents containing a phrase by scanning every document costs time proportional to the corpus, which for millions of documents is far beyond interactive latency. A database index on the text column does not help, because it orders whole values rather than the words inside them.

Search also needs more than matching. Results must be ordered by relevance, which requires knowing how often terms occur, where, and how distinctive they are — information no ordinary index carries.

> **Index the words, not the documents**  
> Turning the relationship around — from documents containing terms to terms appearing in documents — makes a query a lookup of small sorted lists followed by an intersection. Matching stops depending on corpus size and starts depending on how many documents contain the query terms, and the same structure naturally holds the statistics that ranking needs.

**Mental model**

A dictionary of terms, each pointing to a sorted list of the documents containing it along with positions and frequencies.

1. **Analyse** — Text is tokenised, lowercased, stemmed and filtered.
2. **Index** — Each resulting term gains an entry pointing to this document.
3. **Store** — Postings lists are kept sorted and compressed.
4. **Query** — The same analysis is applied to the query, and postings are intersected.
5. **Rank** — Matches are scored using term statistics and returned in order.

> **The analysis chain must be identical at index and query time**  
> If documents are stemmed and queries are not, a search for running will not match a document indexed as run. The mismatch produces no error and no obviously broken behaviour — simply results that are quietly missing — and it is among the most common causes of search that appears to work while omitting relevant documents.

**How it works**

**Structure, analysis and scoring**

```text
THE INDEX
  "database" -> [3, 17, 42, 108, ...]
  "systems"  -> [8, 17, 42, 99,  ...]
  query "database systems"
    -> intersect the two lists -> [17, 42]
    -> cost is proportional to list length, not
       corpus size

POSTINGS CARRY MORE THAN IDS
  doc id, term frequency, positions
  -> frequency feeds ranking
  -> positions enable phrase queries

ANALYSIS PIPELINE
  "The Running Databases"
    tokenise   -> [The, Running, Databases]
    lowercase  -> [the, running, databases]
    stop words -> [running, databases]
    stem       -> [run, databas]
  -> the SAME pipeline must run on queries

SCORING
  term frequency      more occurrences, more relevant
  inverse document
  frequency           rare terms are more informative
  field length        a match in a short title beats
                      one in a long body
  -> BM25 combines these and is the modern default

COMPRESSION
  postings are sorted, so store gaps not ids
    [3, 17, 42, 108] -> [3, 14, 25, 66]
  -> small numbers compress extremely well
  -> indexes are often smaller than the text they
     index
```

1. **Use one analysis definition for indexing and querying** — Any divergence silently loses matches.
2. **Store positions where phrase search is needed** — They cost space and are the only way to match adjacency.
3. **Choose stemming deliberately per language** — Aggressive stemming raises recall and lowers precision.
4. **Compress postings with gap encoding** — Sorted lists of small differences compress dramatically.
5. **Rebuild rather than update in place** — Segment-based designs make updates cheap by never modifying existing segments.
6. **Keep the source of truth elsewhere** — A search index is derived data and must be rebuildable.

**Updates, segments and the near-real-time gap**

```text
UPDATES ARE AWKWARD
  changing one document could touch the postings
  list of every term in it
  -> so indexes are built in SEGMENTS

SEGMENTS
  new documents go into a new small segment
  segments are immutable once written
  searches query all segments and merge results
  background merging combines small segments into
  large ones
  -> the same shape as an LSM tree

DELETES
  marked in a deletion list, filtered at query time
  -> space reclaimed only when segments merge

NEAR REAL TIME
  a document is searchable once its segment is
  visible
  -> typically ~1 s after indexing, not instantly
  -> forcing visibility per document creates tiny
     segments and severe merge pressure

SIZING
  index size is typically 20-100% of the raw text,
  depending on positions, stored fields and
  compression
  -> storing full documents in the index doubles it,
     and is often unnecessary
```

| Metric | Value | Note |
|---|---|---|
| Query cost | postings length | **not corpus size** |
| Structure | immutable segments | merged in background |
| Visibility | ~1 s | near real time |
| Risk | analysis mismatch | silent missing results |

> **Forcing every document to be searchable immediately destroys indexing throughput**  
> Visibility requires a segment to be published, so demanding per-document immediacy creates enormous numbers of tiny segments, each of which must be searched and later merged. Query latency rises because there are more segments to consult, and merge pressure consumes the resources indexing needs. A one-second delay is the standard compromise, and insisting on less is usually a product requirement that has not been examined.

**Technologies**

| Layer | Options | Note |
|---|---|---|
| Search engines | Elasticsearch, OpenSearch, Solr | Distributed inverted indexes with ranking |
| Libraries | Lucene | The engine underneath most of the above |
| Embedded | SQLite FTS, Postgres full text | Sufficient for modest corpora; no separate system |
| Vector search | Embedding-based retrieval | Complementary; matches meaning rather than terms |
| Database LIKE queries | The naive alternative | Scans; no ranking; unusable at scale |
| Hybrid retrieval | Terms plus vectors | Increasingly the default for quality |

Hybrid retrieval deserves attention: term matching excels at exact and rare terms while vector search captures meaning, and combining their results typically outperforms either alone — which is why modern search systems rarely rely on one mechanism.

**Trade-offs**

**Text search approaches**

| Approach | Query cost | Ranking | Operational cost |
|---|---|---|---|
| Scan with pattern matching | Corpus size | None | None |
| Database full-text index | Postings length | Basic | Low; no new system |
| Dedicated search engine | Postings length | Rich | A cluster to run |
| Vector search | Approximate nearest neighbour | Semantic | Embedding pipeline |
| Hybrid | Both | Best | Highest |

The database full-text row is frequently the right starting point: for corpora in the millions rather than billions, it provides real search with ranking and no additional system to operate, and postponing a dedicated cluster until it is genuinely needed avoids a substantial operational commitment.

> **Ask before choosing it**  
> Is the requirement exact matching or relevance ranking? Filtering by known attributes is a database question, and treating it as search brings a whole system to solve something an index already handles — while genuine free-text relevance is exactly what this structure exists for.

**How it fails**

**How inverted indexes go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Relevant documents never returned | Analysis differing between index and query | One shared analysis definition |
| Phrase queries not working | Positions not stored | Enable position indexing |
| Indexing throughput collapses | Forcing immediate visibility | Accept near-real-time refresh |
| Queries slowing over time | Too many small segments | Allow merging to keep pace |
| Index far larger than expected | Storing full documents and all fields | Store only what is retrieved |
| Poor relevance | Default scoring unsuited to the corpus | Tune field weights; consider hybrid retrieval |
| Index and database disagree | Index treated as the source of truth | Keep it derived and rebuildable |

> **Treating the search index as a system of record makes data loss permanent**  
> Search indexes are derived structures with lossy analysis — stemmed terms, dropped stop words, normalised text — so the original content cannot be reconstructed from them. A system that writes only to the index has no authoritative copy, and any corruption, mapping change or accidental deletion is unrecoverable. The index must always be rebuildable from a source that holds the real data.

**Where it is used**

- **Site and product search**, the most common application.
- **Log and observability platforms**, searching enormous volumes of text.
- **Document management and knowledge bases**, where relevance ranking is the point.
- **Code search**, with analysis chains adapted to identifiers and symbols.
- **Retrieval for language model applications**, usually combined with vector search.

**In the interview**

The inversion and the cost model are the core; the analysis pipeline is where practical experience shows.

- **State the inversion** and why it changes the cost from corpus size to postings length.
- **Describe the analysis pipeline** and insist it is identical at index and query time.
- **Explain segments** — immutable, merged in the background — and the near-real-time visibility that follows.
- **Cover ranking briefly**: term frequency, rarity and field length, combined by a modern scoring function.
- **Say the index is derived**, never the system of record, and must be rebuildable.

**Practice drill**

Design search over ten million product listings with titles, descriptions and attributes. Specify the analysis pipeline, which fields carry positions, and how relevance is scored across fields. State how quickly a newly listed product becomes searchable and why. Then describe what happens when the analysis configuration changes, and how you would deploy that change without losing search.

**Go deeper**

An inverted index maps terms to the documents containing them, turning search into a lookup and intersection of sorted lists.

**The inversion changes the cost model fundamentally.** Scanning costs time proportional to the corpus, while intersecting postings costs time proportional to how many documents contain the query terms — which for a specific query is a tiny fraction of the whole. This is what makes interactive search over very large collections possible at all, and it is why no amount of optimising a scan approaches the same result.

**Analysis is where correctness actually lives.** Tokenisation, lowercasing, stop-word removal and stemming transform text into terms, and the same transformation must apply to queries or matches are lost. A mismatch produces no error and no visibly broken behaviour — only results that quietly omit relevant documents — which makes a single shared analysis definition an architectural requirement rather than a configuration convenience.

**Segment-based construction solves the update problem the same way LSM trees do.** Modifying one document could touch the postings of every term it contains, so indexes are built as immutable segments that are searched together and merged in the background. The consequences follow directly: deletes are filtered rather than removed, space is reclaimed only on merge, and a document becomes searchable when its segment is published rather than when it is indexed.

**Near-real-time visibility is a deliberate compromise worth defending.** Publishing a segment per document creates vast numbers of tiny segments, each searched on every query and each requiring later merging, which degrades both query latency and indexing throughput. The customary one-second delay is what keeps segments large enough to be efficient, and a requirement for instant visibility usually turns out, on examination, not to be a real requirement.

**Ranking is inseparable from the structure.** The same postings that answer whether a term appears also record how often and where, which is exactly what relevance scoring needs — frequency within the document, rarity across the corpus, and the length of the field matched. This is why search engines exist as distinct systems rather than as an index type: matching is the easy half, and ordering is what users actually judge.

**It is derived data and must be treated as such.** Analysis is lossy, so the original text cannot be recovered from the index, and any mapping change, corruption or mistaken deletion is unrecoverable without a separate authoritative copy. Systems that write only to the index have no source of truth, and the ability to rebuild from scratch is also what makes analysis changes deployable — since altering the pipeline means reindexing everything it ever processed.

**Related patterns:** Spatial Index · Scatter Gather · Materialized View · LSM Tree

---

### Spatial Index

*Index locations so that proximity queries examine only nearby candidates rather than scanning everything.*

> **When you hear…** Find everything within a radius · nearest neighbour queries · maps, delivery and location-based matching

**Flow:** `Points in space` → `Partition the plane` → `Query defines a region` → `Fetch nearby cells` → `Filter by true distance`

**The problem**

Finding every driver within two kilometres by computing the distance to every driver costs a full scan, and the arithmetic itself is expensive. At a hundred thousand drivers and thousands of queries a second, this is not viable.

Ordinary indexes do not help either. An index on latitude narrows one dimension and leaves a band stretching around the world; combining two single-column indexes still produces a rectangle far larger than the region actually wanted.

> **Reduce two dimensions to one while preserving locality**  
> If space is divided into cells and each cell is given an identifier such that nearby cells have nearby identifiers, then a proximity query becomes a small number of range lookups in an ordinary sorted index. The two-dimensional problem is mapped onto the one-dimensional structures databases are already good at, and the query examines only the cells the region touches.

**Mental model**

A partition of space into cells, each with an identifier that preserves locality, so that nearby points have nearby keys.

1. **Partition** — Space is divided into cells, uniformly or adaptively.
2. **Encode** — Each point is assigned the identifier of its cell.
3. **Query** — The search region is converted into the set of cells it covers.
4. **Fetch** — Candidates are retrieved from those cells.
5. **Refine** — True distances are computed to remove candidates outside the region.

> **The index returns candidates, not answers**  
> Cells are rectangles or squares and query regions are usually circles, so retrieved candidates include points outside the true radius. The exact distance filter is mandatory, and omitting it returns results that are visibly wrong at the edges — a driver four kilometres away appearing in a two-kilometre search.

**How it works**

**Geohash, grids and trees**

```text
GEOHASH  (interleave and encode)
  interleave the bits of latitude and longitude
  encode into a string
    "u4pruyd"  -> a cell a few metres across
    "u4pruy"   -> a larger cell containing it
  -> a shared prefix means spatial proximity
  -> a prefix range query finds everything in a cell

THE EDGE PROBLEM
  two points either side of a cell boundary can be
  metres apart with completely different geohashes
  -> ALWAYS query the target cell plus its 8
     neighbours
  -> forgetting this is the classic geohash bug

QUADTREE  (adaptive)
  split a cell into four when it holds too many
  points
  -> dense cities get fine cells, oceans get coarse
     ones
  -> uniform candidate counts regardless of density

R-TREE  (bounding boxes)
  hierarchical rectangles around groups of shapes
  -> handles polygons and lines, not only points
  -> what most spatial databases use

S2 AND H3
  sphere-aware cell systems
  -> no distortion near the poles
  -> hexagons (H3) have uniform neighbour distances,
     which suits movement and coverage problems
```

1. **Always include neighbouring cells in a query** — Points just across a boundary are near in space and far in key order.
2. **Choose the cell size from the typical query radius** — Cells much smaller than the radius mean many lookups; much larger means many candidates.
3. **Always filter by exact distance afterwards** — Cells approximate the region; they do not define it.
4. **Prefer adaptive structures where density varies** — A uniform grid returns thousands of candidates downtown and none in the countryside.
5. **Use a sphere-aware system for global data** — Flat grids distort badly at high latitudes.
6. **Consider update cost for moving objects** — Frequently moving points change cells and rewrite index entries.

**Density, updates and what to index**

```text
DENSITY VARIES ENORMOUSLY
  1 km cell in a city centre   -> 5,000 drivers
  1 km cell in farmland        -> 0
  -> a uniform grid is wrong everywhere at once
  -> adaptive structures keep candidate counts even

MOVING OBJECTS
  100,000 drivers, position every 4 s
  = 25,000 updates/s
  -> most updates stay within the same cell
  -> only rewrite the index entry when the cell
     changes
  -> this optimisation is frequently the difference
     between viable and not

NEAREST NEIGHBOUR
  expanding ring search:
    query the containing cell
    if fewer than k results, widen to the
    surrounding ring
    repeat
  -> bounded work in dense areas, correct in sparse
     ones

WHAT NOT TO DO
  computing distance to every row       full scan
  latitude BETWEEN ... AND longitude    a rectangle
  BETWEEN ...                           spanning far
                                        more than
                                        intended, and
                                        only usable
                                        with a
                                        composite
                                        index
```

| Metric | Value | Note |
|---|---|---|
| Query | cells, not scans | **candidates only** |
| Boundary | query 9 cells | not 1 |
| Density | adaptive structures | even candidate counts |
| Moving points | update on cell change | not every ping |

> **Querying only the containing cell misses everything just across a boundary**  
> Two points a metre apart on opposite sides of a cell edge fall in different cells with unrelated identifiers, so a query restricted to the containing cell omits genuinely near results. The symptom is subtle — nearby items occasionally missing, depending on exactly where the query point sits — and the fix is to always include the surrounding cells, which is cheap and non-negotiable.

**Technologies**

| Option | Structure | Note |
|---|---|---|
| PostGIS | R-tree indexes | Full geometry support in a relational database |
| Geohash in any key-value store | Prefix ranges | Simple; requires neighbour queries |
| S2 | Spherical cells | Sphere-aware; used for large-scale geospatial systems |
| H3 | Hexagonal cells | Uniform neighbour distances; good for movement and coverage |
| Redis geospatial commands | Sorted sets over geohashes | Convenient for real-time proximity |
| Search engines with geo queries | Combined with text filters | Useful when search and location are both needed |

Hexagonal systems have a property that matters for movement problems: every neighbour is the same distance away, whereas a square cell's diagonal neighbours are further than its edge neighbours — which distorts anything that reasons about spreading, coverage or travel between cells.

**Trade-offs**

**Spatial indexing approaches**

| Approach | Handles density | Shapes | Update cost |
|---|---|---|---|
| Full scan with distance | n/a | Any | None |
| Uniform grid or geohash | Poorly | Points | Low |
| Quadtree | Well | Points | Moderate |
| R-tree | Well | Points and polygons | Higher |
| S2 or H3 cells | Well, with level choice | Points and regions | Low |

Geohash remains popular despite its weaknesses because it requires nothing beyond an ordinary sorted index — a string prefix query in any database provides spatial search, which for many applications is enough and needs no specialised system.

> **Ask before choosing it**  
> Are the objects moving, and how unevenly are they distributed? Static, evenly spread points work with almost any approach; rapidly moving points in wildly varying density are what distinguish the structures, and they are also exactly the case most location-based products have.

**How it fails**

**How spatial indexes go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Nearby results missing | Only the containing cell queried | Include neighbouring cells |
| Results outside the radius | No exact distance filter | Refine candidates by true distance |
| Slow queries in dense areas | Uniform cells returning thousands of candidates | Adaptive structure or finer levels |
| Empty results in sparse areas | Cells too small for the density | Expanding ring search |
| Index write load excessive | Reindexing on every position update | Update only when the cell changes |
| Distortion at high latitudes | Flat grid over a sphere | Sphere-aware cell system |
| Distance calculations wrong | Euclidean arithmetic on latitude and longitude | Proper geodesic distance, or a projection |

> **Treating latitude and longitude as flat coordinates gives wrong distances**  
> A degree of longitude is about 111 kilometres at the equator and near zero at the poles, so Euclidean distance over raw coordinates is increasingly wrong as latitude rises — a two-kilometre search in northern Europe can return points several kilometres away. Correct results require geodesic distance or a suitable projection, and the error is easy to miss when testing near the equator or over small areas.

**Where it is used**

- **Ride-hailing and delivery platforms**, matching moving vehicles to requests in real time.
- **Mapping and local search**, finding places within a viewport or radius.
- **Geofencing**, detecting entry and exit from defined regions.
- **Location-based social features**, surfacing nearby users or content.
- **Logistics and coverage planning**, where hexagonal cells simplify adjacency reasoning.

**In the interview**

The locality-preserving reduction is the idea; the boundary problem is the detail that shows practical experience.

- **Explain the reduction**: two dimensions mapped to one key such that nearby points have nearby keys.
- **Raise the boundary problem** and say that a query must include the surrounding cells.
- **Insist on the exact distance filter**, because cells are approximations of the region.
- **Address density**, since uniform cells fail simultaneously in cities and in open country.
- **Discuss update cost for moving objects**, and the optimisation of reindexing only on cell change.

**Practice drill**

Design driver location search for a ride-hailing service: 100,000 drivers updating every 4 seconds, with queries for all drivers within 2 kilometres. Choose the index structure and cell size, and justify both. Describe the query, including how you handle cell boundaries and exact distance. Then compute the index write load and explain the optimisation that reduces it, and say how your design behaves in a dense city centre compared with a rural area.

**Go deeper**

A spatial index organises locations so that proximity queries examine only the region of interest rather than the whole dataset.

**It works by reducing dimensions while preserving locality.** Assigning each point a cell identifier where nearby cells have nearby identifiers turns a two-dimensional problem into range queries over an ordinary sorted index — which is precisely what existing storage engines do well. Composing two single-column indexes cannot achieve this, because each narrows one dimension independently and the intersection is a rectangle far larger than the region actually wanted.

**Cell boundaries create the characteristic bug of the whole family.** Two points a metre apart on opposite sides of an edge receive unrelated identifiers, so querying only the containing cell silently omits genuinely nearby results. Including the surrounding cells is inexpensive and mandatory, and the reason the mistake persists is that the failure is intermittent and depends on where the query point happens to fall.

**The index produces candidates and the application produces answers.** Cells are rectangles or hexagons while query regions are typically circles, so retrieved points include some outside the true radius. The exact distance filter is part of the query rather than an optimisation, and computing that distance correctly matters too — Euclidean arithmetic on raw coordinates is wrong by a factor that grows with latitude.

**Density variation is what separates the structures.** A uniform grid sized for a city centre returns nothing useful in open country, and one sized for open country returns thousands of candidates downtown; adaptive structures subdivide where points are dense so that candidate counts stay roughly even regardless of location. Since most location-based products serve exactly this kind of wildly uneven distribution, the choice usually matters.

**Moving objects change the economics entirely.** A fleet reporting position every few seconds generates enormous index write volume, and the optimisation that makes it viable is recognising that most updates do not change the cell — so the index entry need not be rewritten. This single observation frequently determines whether a real-time location system is affordable, and it is invisible if the problem is considered only as a query problem.

**Cell system choice encodes assumptions worth being explicit about.** Flat grids distort on a sphere and become badly wrong at high latitudes; spherical systems avoid that; hexagonal systems additionally give every neighbour an equal distance, which matters for anything reasoning about spread, coverage or movement between cells. Selecting one because it is familiar rather than because its geometry suits the problem is a decision that becomes expensive to revisit once identifiers are stored everywhere.

**Related patterns:** Inverted Index · Consistent Hashing · Sharding · Scatter Gather

---

## Scheduling

### Priority Queue

*Order pending work by importance rather than arrival, so limited capacity is spent on what matters most first.*

> **When you hear…** Mixed workloads with different urgency · bulk work delaying interactive work · capacity insufficient for everything at once

**Flow:** `Work arrives with priority` → `Ordered by importance` → `Workers take the highest` → `Low priority waits` → `Ageing prevents starvation`

**The problem**

A queue processes a password reset email behind twenty thousand marketing messages because they arrived first. The user waits ten minutes for something that should take seconds, while the system works on items nobody is waiting for.

Separating everything into its own queue helps until there are eight queues, each with its own workers, each idle while another is overwhelmed — capacity is partitioned rather than shared, and the partitioning is wrong as soon as the traffic mix changes.

> **Order by what the work is worth, not by when it arrived**  
> First-in-first-out is only correct when all items are equally urgent, which is rarely true. Ordering by priority means that when capacity is scarce — exactly when the ordering matters — it is spent on the most valuable work, and when capacity is ample the ordering costs nothing because everything is processed promptly anyway.

**Mental model**

A single pool of pending work ordered by a priority value, with workers always taking the highest-priority item available.

1. **Classify** — Each item is assigned a priority when it is enqueued.
2. **Order** — The queue maintains ordering by priority, then by arrival.
3. **Dequeue** — Workers always take the most important available item.
4. **Age** — Long-waiting items gain priority so they are not starved.
5. **Observe** — Wait time is tracked per priority, not in aggregate.

> **Low-priority work can wait forever if higher-priority work never stops arriving**  
> A queue that always serves the most important item will never serve the least important one when the arrival rate of important work exceeds capacity. The low-priority backlog grows silently and indefinitely, and because the aggregate metrics look healthy — most items are processed quickly — the starvation is invisible until someone notices work from three weeks ago.

**How it works**

**Priorities, ageing and fairness**

```text
KEEP THE LEVELS FEW
  1  interactive: password resets, verification
  2  user-visible: order confirmations
  3  background: reports, exports
  4  bulk: marketing, analytics
  -> three to five levels
  -> more than that and nobody can say what the
     difference between 6 and 7 means

AGEING  (the anti-starvation mechanism)
  effective = base_priority - (waiting_minutes / 10)
  -> a level-4 item waiting 30 minutes competes with
     level 1
  -> guarantees eventual service without abandoning
     the ordering

WEIGHTED FAIR ALTERNATIVE
  reserve capacity per class instead of ordering
    70% interactive, 20% background, 10% bulk
  -> every class always progresses
  -> high priority may wait behind reserved bulk
     capacity
  -> better when starvation is unacceptable and
     strict ordering is not required

IMPLEMENTATION NOTES
  a heap in memory is trivial and not durable
  a database queue: ORDER BY priority, created_at
                    FOR UPDATE SKIP LOCKED
  brokers vary: some support priorities natively,
  many do not and need separate queues per level
```

1. **Use a small number of meaningful levels** — Fine-grained priorities cannot be assigned consistently by anyone.
2. **Implement ageing from the start** — Starvation is certain under sustained load and invisible in aggregate metrics.
3. **Monitor wait time per priority level** — An overall percentile hides a starved class entirely.
4. **Consider weighted capacity instead of strict ordering** — Reserving shares guarantees progress for every class.
5. **Do not let callers set their own priority** — Everything becomes urgent; priority must be assigned by policy.
6. **Keep the ordering cheap** — A priority computation that queries other systems becomes the bottleneck.

**Where priorities go wrong in practice**

```text
PRIORITY INFLATION
  teams choose the priority for their own work
  -> everything becomes level 1 within a quarter
  -> the queue is FIFO again, with extra machinery
  -> priorities must be assigned by policy, centrally

FALSE PRECISION
  priority 1 through 100
  -> nobody can justify 43 over 47
  -> and the distinction does not change behaviour
  -> use levels people can name

THE REAL QUESTION IS CAPACITY
  if high-priority work alone exceeds capacity,
  prioritisation only decides who is disappointed
  -> it reorders, it does not create throughput
  -> a permanently deep queue is a capacity problem
     wearing a scheduling costume

MEASURING CORRECTLY
  overall p99 wait: 2 s   -> looks excellent
  level 4 p99 wait: 6 days -> nobody is looking
  -> always segment by priority
```

| Metric | Value | Note |
|---|---|---|
| Ordering | by value | **not arrival** |
| Required | ageing | or starvation |
| Levels | 3-5 | nameable |
| Metric | wait per level | not aggregate |

> **Letting each team set its own priority converts the queue back into first-in-first-out**  
> When the caller chooses, everything becomes urgent — not through bad faith, but because every team's work genuinely matters to them. Within a couple of quarters every item is at the top level and the ordering carries no information, while the system retains all the complexity of supporting it. Priority has to be assigned by a policy that compares work across teams, which is an organisational decision rather than a technical one.

**Technologies**

| Option | Mechanism | Note |
|---|---|---|
| Database-backed queue | Order by priority with skip locked | Simple, durable, flexible ordering |
| Brokers with native priorities | Priority field on messages | Support and semantics vary considerably |
| Separate queues per level | Workers prefer higher queues | Works everywhere; capacity is partitioned |
| Weighted fair queueing | Reserved shares per class | Guarantees progress for every class |
| In-memory heaps | Within a single process | Fast, not durable |
| Work queue without priorities | First in, first out | Correct when urgency is uniform |

Separate queues per level with workers that prefer the higher ones is the most portable implementation and the most commonly deployed, because many brokers either lack priority support or implement it with surprising semantics under load.

**Trade-offs**

**Scheduling policies**

| Policy | Urgent work | Starvation risk | Complexity |
|---|---|---|---|
| First in, first out | Waits its turn | None | Lowest |
| Strict priority | Served first | High | Low |
| Priority with ageing | Served first | Bounded | Moderate |
| Weighted fair queueing | Served within its share | None | Moderate |
| Separate queues and pools | Isolated | None | Partitioned capacity |

Weighted fair queueing is underused relative to strict priority: guaranteeing every class a share of capacity removes starvation by construction, and for most systems the difference between urgent work being served first and being served within its reserved share is not worth the risk of a class that never progresses.

> **Ask before choosing it**  
> Is the queue permanently deep, or only deep during bursts? Prioritisation reorders a backlog and does not shorten it, so a queue that never drains is a capacity problem — and scheduling will merely determine which work is permanently delayed.

**How it fails**

**How priority queues go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Low-priority work never processed | Strict priority under sustained load | Ageing, or reserved capacity per class |
| Everything at the top level | Callers choosing their own priority | Central policy assigns priorities |
| Starvation unnoticed for weeks | Only aggregate metrics tracked | Wait time per priority level |
| Priorities meaningless | Too many levels | Reduce to a few nameable classes |
| Ordering becomes a bottleneck | Expensive priority computation | Compute it at enqueue time and store it |
| Capacity partitioned badly | One queue and pool per level | Shared pool with ordering |
| Urgent work still delayed | Long-running low-priority items holding workers | Bound task duration; preempt where possible |

> **Starvation hides behind healthy aggregate metrics**  
> When ninety-five per cent of items are high priority and served in seconds, the overall percentiles look excellent while the remaining class waits for days. Nothing alerts, nothing errors, and the problem surfaces when someone notices an export request from a fortnight ago still pending. Segmenting wait-time metrics by priority is what makes this visible, and it must be in place before the load that causes it arrives.

**Where it is used**

- **Notification and email delivery**, separating transactional messages from bulk sends.
- **Job processing platforms**, running interactive tasks ahead of scheduled reports.
- **Support and incident systems**, where severity determines processing order.
- **Media processing pipelines**, prioritising user-initiated work over re-encoding.
- **Operating system and database schedulers**, the original application of the idea.

**In the interview**

Raise starvation before being asked; it is the failure that defines the pattern.

- **Give a small set of named levels** rather than a numeric range nobody can apply consistently.
- **Introduce ageing immediately** as the mechanism that bounds how long low-priority work waits.
- **Insist on per-level metrics**, because aggregate percentiles conceal a starved class completely.
- **Warn about priority inflation** and say assignment must be by central policy, not by the caller.
- **Distinguish reordering from capacity**: a permanently deep queue needs throughput, not scheduling.

**Practice drill**

A queue handles password resets, order confirmations, scheduled reports and marketing emails with capacity for 60% of peak volume. Define the priority levels and the ageing function. Compute how long a marketing email waits during a peak hour with and without ageing. Then describe the metrics you would publish, and explain how you would recognise starvation if ageing were not implemented.

**Go deeper**

A priority queue orders pending work by importance so that scarce capacity is spent on the most valuable items first.

**It only matters when capacity is insufficient, which is exactly when it matters most.** With ample throughput everything is processed promptly and ordering is irrelevant; under load the ordering determines whether a user waits seconds or minutes for something urgent. This means the value of the mechanism is concentrated entirely in the periods that define the product's perceived reliability.

**Strict priority guarantees starvation whenever important work saturates capacity.** If high-priority arrivals alone exceed throughput, the lowest class is never reached, and its backlog grows without bound. Ageing — raising effective priority with waiting time — bounds the delay while preserving the ordering, and weighted capacity reservation avoids the problem entirely by guaranteeing each class a share. One of the two is necessary; strict priority without either is a design that works until it is needed.

**The failure is invisible in aggregate measurements.** When most items are high priority and served quickly, overall percentiles look excellent while one class waits for days. Because nothing errors and nothing alerts, starvation is typically discovered by a person noticing stale work rather than by monitoring — which makes segmenting wait times by priority level a prerequisite for operating the pattern rather than an enhancement.

**Priority assignment is an organisational problem in technical clothing.** When each team sets the priority of its own work, everything reaches the top level within a few quarters, not through bad faith but because every team's work genuinely is important to them. Meaningful ordering requires a policy that compares work across teams, and a system that cannot make that comparison will end up with a first-in-first-out queue carrying the overhead of a priority one.

**Few levels beat many.** A range of a hundred priorities invites distinctions nobody can justify or apply consistently, and the fine gradations do not change behaviour. Three to five classes that people can name — interactive, user-visible, background, bulk — are assignable without debate and produce the same scheduling outcomes, which is the practical test of whether a level is worth having.

**Scheduling reorders a backlog; it does not shorten one.** A queue that is permanently deep has a capacity shortfall, and prioritisation only decides which work is permanently delayed. Recognising that distinction prevents a great deal of wasted tuning: if the queue drains between bursts, ordering is the right tool, and if it never drains, the answer is more throughput or less work regardless of how the remaining capacity is allocated.

**Related patterns:** Work Queue · Delay Queue · Load Shedding · Backpressure

---

### Delay Queue

*Schedule work to become available at a future time, so retries, reminders and deferred actions need no polling loop.*

> **When you hear…** Retries with backoff · reminders and timeouts · anything that must happen later rather than now

**Flow:** `Enqueue with a delay` → `Invisible until due` → `Becomes available` → `Worker processes it` → `Reschedule if needed`

**The problem**

A retry after thirty seconds, a reminder in three days and a cart abandonment check in an hour all require something to happen at a future moment. Implementing this by scanning a table every minute for due rows means a query that scales with pending work and a granularity no finer than the scan interval.

The scan is also wasteful and fragile. Most executions find nothing, the query competes with production traffic as the table grows, and a missed or delayed scan silently postpones everything scheduled within it.

> **Make the queue itself aware of time**  
> If an item can be enqueued with a time before which it is invisible, then scheduling is a property of the message rather than a separate scanning mechanism. Nothing polls, nothing scans, and the work becomes available exactly when it is due — with the queue handling ordering, durability and delivery as it does for immediate work.

**Mental model**

A queue where each item carries a visibility time. Items exist but cannot be received until that moment arrives.

1. **Enqueue** — The item is submitted with a delay or an absolute due time.
2. **Hide** — It is durable but invisible to consumers until due.
3. **Surface** — At the due time it becomes available like any other item.
4. **Process** — A worker receives and handles it normally.
5. **Reschedule** — Work needing another attempt is re-enqueued with a new delay.

> **Delivery is at-least-once and approximately on time, not exactly either**  
> Delayed items can be delivered late under load and can be delivered more than once, so anything scheduled must be idempotent and must tolerate arriving somewhat after its due time. Designs treating a delay queue as a precise timer — for financial cut-offs or hard deadlines — are relying on a guarantee it does not provide.

**How it works**

**Implementations and their limits**

```text
NATIVE QUEUE DELAY
  enqueue with delay_seconds
  + simple, durable, no extra infrastructure
  - many brokers cap the delay (commonly 15 minutes)
  -> fine for retries, not for three-day reminders

SORTED SET BY DUE TIME
  score = due timestamp
  a poller fetches everything with score <= now
  + arbitrary delays
  + efficient: one range query regardless of size
  - one component must poll, and must be reliable

DATABASE TABLE
  WHERE due_at <= now() ORDER BY due_at
  FOR UPDATE SKIP LOCKED
  + transactional with business data
  + arbitrary delays, easy inspection
  - polling load grows with table size unless indexed
    and pruned

DEAD LETTER AND REDRIVE
  delayed retries commonly implement backoff:
    attempt fails -> re-enqueue with 2^n seconds
    after N attempts -> dead letter queue

LONG DELAYS
  for delays beyond the broker's cap, chain them:
    enqueue with 15 min, re-enqueue on arrival
  -> works, but multiplies message volume
  -> a sorted set or table is usually cleaner
```

1. **Match the mechanism to the delay length** — Broker delays suit seconds to minutes; long delays need a scheduled store.
2. **Make every delayed task idempotent** — Delivery is at-least-once, and duplicates are normal.
3. **Store the due time, not the delay** — An absolute time survives restarts and re-enqueues unambiguously.
4. **Include cancellation in the design** — Scheduled work is frequently made obsolete before it runs.
5. **Expect late delivery under load** — The due time is the earliest it can run, not a guarantee.
6. **Prune completed and cancelled items** — A scheduling table that only grows eventually degrades the scan.

**Cancellation and the check-on-execution rule**

```text
THE PROBLEM
  schedule "cart abandonment email in 1 hour"
  the user completes the purchase 10 minutes later
  -> the email must not be sent

OPTION 1  DELETE THE SCHEDULED ITEM
  requires knowing its identifier and the store
  supporting removal
  -> queues generally do NOT support cancelling an
     enqueued message
  -> works with a database table or sorted set

OPTION 2  CHECK CONDITIONS ON EXECUTION
  the task re-reads current state when it runs:
    "is this cart still abandoned?"
    if not, do nothing
  -> works with ANY mechanism
  -> the task must be written to expect obsolescence
  -> this is the robust default

-> even where cancellation is possible, the check on
   execution should remain: a cancellation can be
   missed, and the state may have changed in ways
   nobody anticipated

TIMING EXPECTATIONS
  due at 10:00 might run at 10:00:03 or 10:04
  -> acceptable for reminders and retries
  -> unacceptable for anything with a hard deadline
```

| Metric | Value | Note |
|---|---|---|
| Scheduling | in the queue | **no scanning** |
| Delivery | at-least-once | idempotent tasks |
| Timing | earliest, not exact | late under load |
| Cancellation | check on execution | the robust default |

> **Most queues cannot cancel a message once it has been enqueued**  
> Scheduled work is frequently invalidated before it runs — the user acts, the order is cancelled, the condition resolves — and a broker that accepted a delayed message usually offers no way to withdraw it. The reliable pattern is for the task to verify its own preconditions when it executes and do nothing if they no longer hold, which works regardless of the mechanism and also covers cases nobody thought to cancel.

**Technologies**

| Option | Delay range | Note |
|---|---|---|
| Queue native delay | Seconds to minutes | Simplest; commonly capped around 15 minutes |
| Redis sorted set by due time | Arbitrary | Efficient range query; needs a reliable poller |
| Database table with a due column | Arbitrary | Transactional; easy to inspect and cancel |
| Scheduler services | Arbitrary | Managed; per-schedule cost and limits |
| Durable workflow engines | Arbitrary | Delays are a first-class step |
| Cron | Fixed times | For recurring work, not per-item scheduling |

The distinction between cron and a delay queue is worth stating: cron runs the same job on a schedule, while a delay queue schedules an individual item — and using cron plus a scan to emulate per-item scheduling is exactly the polling approach the pattern replaces.

**Trade-offs**

**Ways to do something later**

| Approach | Granularity | Scales with pending items | Cancellation |
|---|---|---|---|
| Polling scan | Scan interval | Poorly | Easy |
| Queue native delay | Seconds | Well | Usually impossible |
| Sorted set by due time | Seconds | Well | Easy |
| Scheduler service | Varies | Well | Usually possible |
| Workflow engine timer | Seconds | Well | Built in |
| In-process timer | Milliseconds | Poorly | Easy, and lost on restart |

In-process timers are the shortcut that fails quietly: a scheduled callback held in memory disappears on deploy or crash, so the work simply never happens and nothing records that it was lost.

> **Ask before choosing it**  
> How long is the delay, and what happens if the work runs twice or arrives late? Short delays with idempotent tasks suit a queue's native mechanism; long delays and anything needing cancellation point to a store you can query and modify.

**How it fails**

**How delay queues go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Scheduled work lost on deploy | In-process timers | Durable scheduling store |
| Obsolete actions taken | No condition check at execution | Verify preconditions when the task runs |
| Long delays unsupported | Broker delay cap | Sorted set, table, or chained re-enqueues |
| Scheduling scan slows over time | Table never pruned | Delete completed items; index the due column |
| Duplicate actions | At-least-once delivery | Idempotent tasks |
| Everything due at once | Round-number scheduling | Jitter the due times |
| Retry storms | Fixed backoff across many items | Exponential backoff with jitter |

> **Scheduling large numbers of items for the same instant produces a self-inflicted spike**  
> Delays computed as round numbers — an hour from now, midnight tomorrow — cluster due times, so thousands of tasks become available simultaneously and overwhelm the workers. The pattern is easy to create accidentally, since every natural way of expressing a delay produces alignment, and the remedy is to add randomness to due times whenever many items are scheduled together.

**Where it is used**

- **Retry with backoff**, the most common use, re-enqueuing failures with growing delays.
- **Reminders and notifications**, scheduled days or weeks ahead.
- **Abandonment and follow-up flows**, acting if a condition persists after an interval.
- **Timeouts on long-running operations**, scheduling a check that fires if nothing completes.
- **Durable workflows**, where waiting for a period is an ordinary step.

**In the interview**

Contrast it with polling, then cover cancellation — which is where most designs are incomplete.

- **Explain why polling is the wrong shape**: cost grows with pending work and granularity is bounded by the interval.
- **Match the mechanism to the delay length**, since broker delays are typically capped at minutes.
- **Say delivery is at-least-once and approximate**, so tasks must be idempotent and tolerate lateness.
- **Raise cancellation** and propose checking preconditions at execution as the mechanism-independent answer.
- **Add jitter to due times**, because natural scheduling expressions cluster and produce spikes.

**Practice drill**

Design scheduling for three cases: retry a failed payment in 30 seconds, send a reminder in 3 days, and check cart abandonment in 1 hour. Choose a mechanism for each and justify it. Describe how the abandonment check avoids emailing someone who has already purchased. Then explain what happens if fifty thousand reminders are all scheduled for 09:00 tomorrow, and how you would prevent it.

**Go deeper**

A delay queue makes work available at a future time, so deferred actions need no polling loop or scanning job.

**It moves scheduling into the queue, which changes the cost model.** Scanning a table for due items costs work proportional to pending volume and offers granularity no finer than the scan interval, while a queue that understands visibility times surfaces each item exactly when it is due with no periodic query at all. The scheduling becomes a property of the message rather than a separate mechanism that must itself be reliable.

**The mechanism has to match the delay length.** Broker-native delays are typically capped at minutes, which covers retries and short timeouts but not reminders days away — those need a store that can hold arbitrary due times and be queried by range. Chaining short delays to emulate long ones works and multiplies message volume, which is usually a sign that a sorted set or table is the better structure.

**Timing is a lower bound, not a guarantee.** Items can surface late under load and can be delivered more than once, so tasks must be idempotent and must tolerate running after their intended moment. Treating the mechanism as a precise timer — for a financial cut-off or a hard deadline — relies on a property it does not have, and such requirements need a design that verifies the time itself rather than trusting delivery.

**Cancellation is where most designs are incomplete.** Scheduled work is frequently invalidated before it runs, and queues generally offer no way to withdraw an enqueued message. The reliable approach is for the task to check its own preconditions when it executes and do nothing if they no longer hold, which works with every mechanism and additionally covers the cases nobody anticipated cancelling — making it the right default even where removal is possible.

**Aligned due times produce self-inflicted spikes.** Every natural way of expressing a delay — an hour from now, nine o'clock tomorrow — clusters items at the same instant, so a large batch becomes available simultaneously and saturates the workers. Adding randomness to due times whenever many items are scheduled together spreads the load, and it is the same reasoning that makes jitter essential in retry policies.

**In-process timers are the shortcut that loses work silently.** A callback scheduled in memory disappears on deploy, crash or scale-in, and nothing records that it was lost — the reminder simply never arrives and no error is generated anywhere. Any scheduled action that matters needs a durable store behind it, which is the minimum the pattern provides and the reason it exists as infrastructure rather than as a language feature.

**Related patterns:** Work Queue · Priority Queue · Retry With Jitter · Durable Workflow

---

### Durable Workflow

*Persist each step's completion so a multi-step process survives crashes, restarts and long waits without losing its place.*

> **When you hear…** Multi-step business processes · operations spanning minutes to months · orchestration that must survive deploys

**Flow:** `Define steps` → `Execute one` → `Persist the result` → `Crash safe` → `Resume where it stopped`

**The problem**

An onboarding process creates an account, sends a verification email, waits up to three days for confirmation, provisions resources and schedules a follow-up. Holding this in a function means the process dies with the process — and three days will certainly contain a deploy.

Rebuilding it from queue messages and database flags works and scatters the logic: the sequence exists nowhere in readable form, and answering where a particular customer's onboarding has reached requires querying several systems and inferring.

> **Make progress durable so the code can be written as a sequence**  
> If the result of every completed step is persisted, execution can be resumed by replaying that history and skipping what already happened. The process can then be expressed as ordinary sequential code — including waits of arbitrary length — because no crash, deploy or restart loses its position, and the code itself is the readable definition of the flow.

**Mental model**

A durably recorded history of completed steps. On resumption the history is replayed to reconstruct state, and execution continues from the first step with no recorded result.

1. **Define** — The workflow is written as a sequence of steps with waits and branches.
2. **Execute** — Each step runs, and its result is persisted before moving on.
3. **Wait** — Timers and external signals suspend the workflow without consuming resources.
4. **Recover** — After any interruption the history is replayed and execution resumes.
5. **Complete** — The final state and the full history remain available for inspection.

> **Workflow code must be deterministic, and this constrains how it is written**  
> Because resumption replays the recorded history, the code must take the same path given the same results — so current time, random values and direct external calls cannot appear in workflow logic. They must be performed inside steps whose results are recorded. Code that ignores this appears to work and fails only on recovery, which is both rare and exactly when correctness matters most.

**How it works**

**Steps, determinism and versioning**

```text
WHAT THE ENGINE RECORDS
  step 1 createAccount   -> {id: 4471}
  step 2 sendEmail       -> {sent: true}
  step 3 awaitConfirm    -> pending
  -> on resume, steps 1 and 2 are not re-executed;
     their recorded results are returned

DETERMINISM RULES
  NOT in workflow code:
    current time, random numbers, uuid generation
    direct network or database calls
    iteration over an unordered collection
  INSTEAD:
    perform them inside a step, so the result is
    recorded and replayed identically

-> the workflow decides; steps act

WAITS ARE FREE
  sleep for 3 days
  -> the workflow is suspended, not running
  -> no thread, no memory, no polling
  -> this is what makes long processes practical

VERSIONING IS THE HARD PART
  a workflow started last week is running the old
  code path
  deploy new code with different steps
  -> replay may not match the recorded history
  -> engines provide versioning primitives; they
     must be used deliberately
  -> in practice: avoid changing the step sequence
     of long-running workflows; branch on a version
     marker instead
```

1. **Keep all non-determinism inside steps** — Replay must follow the same path; time and randomness in workflow code break that.
2. **Make every step idempotent** — A step may be retried after a crash between execution and recording.
3. **Use the engine's waiting primitives** — A suspended workflow consumes nothing; a polling loop consumes everything.
4. **Plan versioning before long workflows are deployed** — In-flight instances outlive several releases.
5. **Keep workflow logic free of business rules that change often** — Frequent changes to long-running definitions are where versioning pain concentrates.
6. **Expose workflow state to operations** — Being able to answer where an instance is stuck is a principal benefit.

**What it replaces, and what it costs**

```text
WITHOUT AN ENGINE
  state machine in a database table
  queue messages between steps
  a scheduler for waits
  retry logic per step
  reconciliation for stuck instances
  -> all of this is written, tested and operated by
     you
  -> and the sequence is not visible anywhere

WITH AN ENGINE
  the sequence is the code
  progress, retries, waits and history are provided
  -> plus a service to operate, or a hosted one to
     pay for

COSTS
  an engine to run or a service to buy
  determinism constraints on how code is written
  versioning complexity for long workflows
  a new operational surface to understand

WHEN IT IS WORTH IT
  several multi-step processes, not one
  processes lasting longer than a deploy cycle
  failures that currently require manual repair
  a real need to answer "where is this instance?"

WHEN IT IS NOT
  a single three-step flow
  -> a saga with an outbox is lighter and sufficient
```

| Metric | Value | Note |
|---|---|---|
| Progress | durable per step | **survives anything** |
| Waits | free | no resources held |
| Constraint | determinism | in workflow code |
| Hard part | versioning | long-running instances |

> **Changing a workflow definition while instances of it are running is the principal operational hazard**  
> Instances started before a deploy replay their recorded history against the new code, and if the step sequence has changed the replay may not match — producing a failure that affects only in-flight instances and only on recovery. Long-running workflows routinely outlive several releases, so versioning is not an edge case but the normal condition, and engines provide explicit primitives precisely because naive changes break running work.

**Technologies**

| Option | Model | Note |
|---|---|---|
| Workflow engines | Code-as-workflow with replay | Full featured; an engine to operate or buy |
| Cloud step functions | State machines as configuration | Managed; less expressive than code |
| Saga orchestration by hand | Your own state machine and queues | Fine for a small number of short flows |
| Job scheduler with state | Table-driven progress | Simple; you build resumption yourself |
| Event sourcing | State derived from recorded events | Overlapping idea, broader commitment |
| Queue chaining | Each step enqueues the next | Works; the flow is invisible and hard to query |

Queue chaining is what most teams build before adopting an engine, and its defining weakness is not reliability but legibility: the process works, and nobody can see it, so answering an operational question means reconstructing the sequence from message handlers spread across services.

**Trade-offs**

**Orchestration approaches**

| Approach | Survives crashes | Long waits | Flow visible |
|---|---|---|---|
| In-process function | No | No | Yes, in code |
| Queue chaining | Yes | Yes | No |
| State machine table | Yes | Yes | Partly |
| Managed step functions | Yes | Yes | Yes, as configuration |
| Durable workflow engine | Yes | Yes | Yes, as code |

The last column is the one that distinguishes durable workflows from the alternatives that also survive crashes: several approaches keep a process running reliably, and few of them let a person read the process or ask where a particular instance currently is.

> **Ask before choosing it**  
> How many multi-step processes are there, and how long do they run? One short flow does not justify an engine, while several processes spanning days — with failures currently repaired by hand — is exactly the situation the pattern exists for.

**How it fails**

**How durable workflows go wrong**

| Failure | Cause | Fix |
|---|---|---|
| Replay fails after a deploy | Step sequence changed with instances in flight | Version workflows explicitly |
| Non-deterministic behaviour on recovery | Time or randomness in workflow code | Move it into steps |
| Steps executed twice | Crash between execution and recording | Idempotent steps |
| Resources consumed while waiting | Polling instead of the engine's timers | Use durable sleep and signals |
| Instances stuck indefinitely | No timeout on external signals | Bound every wait; define what happens on expiry |
| History growing without limit | Very long-lived workflows with many steps | Continue as a new instance; archive history |
| Engine becomes a single point of failure | All processes depending on it | Run it highly available; understand its failure modes |

> **Non-determinism in workflow code fails only during recovery, which is the worst possible time**  
> Reading the clock or generating a value directly in workflow logic produces correct behaviour on the first execution and a different path on replay, so the defect is invisible in testing and normal operation. It appears when a workflow resumes after a crash — precisely when the process was relying on durability — and manifests as an instance that cannot continue rather than as an obvious error in the code that caused it.

**Where it is used**

- **Order fulfilment and provisioning**, coordinating steps across several services with compensation.
- **Onboarding and approval flows**, waiting days for human action without holding resources.
- **Payment and settlement processes**, requiring durable progress and full auditability.
- **Infrastructure automation**, where a long provisioning sequence must survive operator restarts.
- **Data pipeline orchestration**, with dependencies, retries and long-running steps.

**In the interview**

Lead with durable progress enabling sequential code, then name determinism and versioning as the costs.

- **Say each step's result is persisted**, which is what allows the process to be written as ordinary sequential code.
- **Note that waits are free** — a suspended workflow holds no resources — which is what makes multi-day processes practical.
- **State the determinism constraint** and give the specific things that cannot appear in workflow code.
- **Raise versioning unprompted** as the main operational difficulty, since long instances outlive deploys.
- **Be honest about when it is unnecessary**: one short flow is better served by a saga with an outbox.

**Practice drill**

Design onboarding that creates an account, sends verification, waits up to 3 days for confirmation, provisions resources and schedules a follow-up in 7 days. Write it as a sequence and mark which parts must be steps rather than workflow logic. Say what happens if the service restarts during the 3-day wait and if it restarts between provisioning and recording that result. Then describe how you would add a new step without breaking instances already running.

**Go deeper**

A durable workflow persists the result of every completed step so a multi-step process can resume after any interruption.

**Durable progress is what allows the process to be written as ordinary code.** Because each step's outcome is recorded, execution can be reconstructed by replaying that history and continuing from the first unfinished step — so a sequence spanning days can be expressed as a readable series of statements rather than scattered across message handlers and database flags. The legibility is not a side benefit; for most teams it is the principal one.

**Waiting costs nothing, which changes what is practical.** A suspended workflow holds no thread, no memory and no connection, so waiting three days for a human to act is as cheap as waiting three seconds. Processes that would otherwise be decomposed into scheduled jobs and reconciliation passes, purely to avoid holding resources, can be written as they are actually understood.

**Replay imposes determinism, and this constrains how the code is written.** Reading the clock, generating random values or calling an external system directly inside workflow logic produces a different path on replay than on the original execution. Such code behaves correctly during normal operation and fails only on recovery — when durability was the entire point — which makes the rule about keeping non-determinism inside recorded steps a correctness requirement rather than a style preference.

**Versioning is the dominant operational difficulty.** Instances started before a release replay their recorded history against the new definition, and a changed step sequence can make that replay inconsistent. Since long-running workflows routinely outlive several deploys, this is the normal condition rather than an edge case, and it argues for keeping frequently changing business rules out of workflow definitions and inside steps where they can be altered freely.

**Steps must be idempotent because the crash window is real.** A process can fail between performing a step's effect and recording its result, so recovery re-executes it. Without idempotence that means the effect happens twice, which for anything touching money or external systems is exactly what the durability was meant to prevent — making step design the same problem as any at-least-once message consumer.

**It is worth its cost when there are several long processes, not one.** An engine to operate, determinism constraints and versioning complexity are a real commitment, and a single three-step flow is better served by a saga with an outbox. The signal that it is warranted is a collection of processes spanning longer than a deploy cycle, with failures currently repaired by hand and no reliable way to answer where a particular instance has reached — which is precisely the situation the pattern was built to remove.

**Related patterns:** Saga · Delay Queue · Work Queue · Transactional Outbox

---
