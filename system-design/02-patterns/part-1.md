# Pattern Gym · Part 1 of 3

[← System Design index](../README.md)

> Caching · Data Architecture · Transactions · Messaging · Reliability (24 patterns). Part of the Pattern Gym — Learn to hear the pattern in the problem.

## Contents

- **Caching** (7): [Cache Aside](#cache-aside) · [Read Through Cache](#read-through-cache) · [Write Through Cache](#write-through-cache) · [Write Behind Cache](#write-behind-cache) · [Cache Invalidation](#cache-invalidation) · [Stale While Revalidate](#stale-while-revalidate) · [Single Flight](#single-flight)
- **Data Architecture** (6): [CQRS](#cqrs) · [Event Sourcing](#event-sourcing) · [Change Data Capture](#change-data-capture) · [Materialized View](#materialized-view) · [Consistent Snapshot](#consistent-snapshot) · [Schema Evolution](#schema-evolution)
- **Transactions** (2): [Saga](#saga) · [Two Phase Commit](#two-phase-commit)
- **Messaging** (4): [Transactional Outbox](#transactional-outbox) · [Inbox Deduplication](#inbox-deduplication) · [Work Queue](#work-queue) · [Partitioned Log](#partitioned-log)
- **Reliability** (5): [Idempotency Key](#idempotency-key) · [Retry With Jitter](#retry-with-jitter) · [Circuit Breaker](#circuit-breaker) · [Bulkhead](#bulkhead) · [Hedged Requests](#hedged-requests)

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
