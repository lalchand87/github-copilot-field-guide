# Curriculum · Caching

[← System Design index](../README.md)

> 8 lessons in **Caching**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Caching** (8): [Cache-aside](#cache-aside) · [Write-through caching](#write-through-caching) · [Write-behind caching](#write-behind-caching) · [TTL and invalidation](#ttl-and-invalidation) · [Cache stampede protection](#cache-stampede-protection) · [Cache eviction policies](#cache-eviction-policies) · [Negative caching](#negative-caching) · [Multi-level caching](#multi-level-caching)

## Caching

### Cache-aside

*The application checks the cache, falls back to the source on a miss, and populates the cache itself — simple, ubiquitous, and quietly racy.*

**Flow:** `Request` → `Cache lookup` → `Source read` → `Cache fill` → `Response`

> **The 30-second version**  
> Check the cache, fall back to the source on a miss, and populate it yourself. Ubiquitous and simple — with a stale-write race, stampede risk, and a cold-start hazard that TTLs and coalescing must bound.

**The problem**

A database serves 40,000 reads per second for data that changes a few times an hour. Almost every one of those reads produces the same answer, computed again from the same pages. The database is doing enormous work to repeat itself.

Cache-aside is the obvious response and the most widely deployed caching pattern in existence: look in the cache, and if it is not there, read the source and put it there. Its ubiquity makes its failure modes worth understanding precisely, because almost every system has this pattern somewhere and almost every team has been surprised by one of them.

> **The three failures that define this pattern**
>
> - **Stale write race**: a slow reader writes an old value into the cache *after* a writer invalidated it, and the stale value persists until TTL.
> - **Stampede**: a popular key expires and thousands of concurrent requests all miss and all hit the source simultaneously.
> - **Cold start**: the cache is empty after a restart or failover, and the source receives the full unmitigated load it has not been sized for.

**Mental model**

The cache is a hint, not an authority. The application owns the logic: it decides what to cache, when to populate, and when to invalidate. The cache itself knows nothing about your data model.

1. **Read** — Check cache. Hit: return. Miss: read source, write to cache with a TTL, return.
2. **Write** — Write to the source, then invalidate (or update) the cached entry.
3. **TTL** — The backstop. Whatever invalidation you get wrong, TTL eventually corrects — which is why every entry should have one.
4. **Miss cost** — The first caller after an expiry pays the full source latency. With concurrency, many callers pay it simultaneously.
5. **Ownership** — Because the application populates the cache, it is also responsible for every consistency guarantee the cache provides.

> **Invalidate, do not update**  
> On a write, deleting the cache entry is safer than overwriting it with the new value. Overwriting introduces a second ordering problem — two concurrent writers can apply their cache updates in the opposite order to their database commits, leaving the cache permanently wrong. Deletion has only one outcome: the next reader repopulates from the source of truth.

**How it works**

**The read and write paths, and the race between them**

```text
READ
  v = cache.get(k)
  if v is None:
      v = db.read(k)
      cache.set(k, v, ttl=300)
  return v

WRITE
  db.write(k, new)
  cache.delete(k)          <- delete, not set

THE RACE (why TTL is mandatory)
  T1  reader: cache MISS for k
  T2  reader: db.read(k) -> "old"        (slow query)
  T3  writer: db.write(k, "new")
  T4  writer: cache.delete(k)            (deletes nothing)
  T5  reader: cache.set(k, "old")        <- STALE VALUE WRITTEN

  cache now holds "old" while the database holds "new",
  and it stays that way until the TTL expires.
```

1. **Always set a TTL** — It bounds the damage from every invalidation bug you have not found. A cache entry with no TTL is a permanent inconsistency waiting for the right race.
2. **Delete after the database commit, not before** — Deleting first leaves a window where a reader repopulates from the pre-commit state. Deleting after narrows the window to the race above, which TTL bounds.
3. **Consider delayed double-delete for hot keys** — Delete on write, then delete again after a short delay, to clear any stale value written by an in-flight reader. Crude, but effective and widely used.
4. **Use versioned keys where correctness matters more** — Include a version or updated-at value in the cache key. A write bumps the version, so old entries become unreachable rather than needing deletion — invalidation becomes a non-event.
5. **Coalesce misses on hot keys** — Single-flight: the first miss fetches, concurrent misses wait for that result. This is the difference between one source query and ten thousand.
6. **Treat cache unavailability as a latency event, not a failure** — If the cache is down, requests should fall through to the source — with a concurrency limit, because the source has not been sized for 100% miss rate.

**Versioned keys: invalidation without invalidating**

```text
INSTEAD OF
  key = "user:42"
  write -> db.write(); cache.delete("user:42")   <- racy

USE
  key = "user:42:v" + version
  version comes from the row itself (updated_at, or a counter)

  read:
    version = db.read_version(42)     -- cheap, or from a
                                         small hot "version cache"
    v = cache.get("user:42:v" + version)
    ...

  write:
    db.write(42, new)                 -- version increments
    -- no cache operation at all; old key is now unreachable

TRADE: one extra cheap lookup per read, in exchange for
       eliminating the invalidation race entirely.
       Old entries expire naturally via TTL.
```

> **Cache outages are load events, not availability events**  
> When a cache tier fails, hit rate goes to zero and the source sees the full request rate — often 20 to 100 times its normal load. A database sized for 5% of traffic will collapse. The mitigation is a concurrency limit in front of the source plus load shedding, so a cache outage degrades latency for everyone rather than taking the database down for everyone.

**Worked example**

A product catalogue at 40,000 reads per second. What the cache actually buys, and what happens when it fails.

**Sizing the cache and the failure it creates**

```text
TRAFFIC       40,000 reads/s, 400 writes/s
DATA          2M products x 3 KB = 6 GB
ACCESS        Zipfian: top 1% of products = 60% of reads

CACHE SIZING
  cache the hot 10% (200k products) = 600 MB
  expected hit rate at that working set: ~92%
  cache the hot 30% (600k)           = 1.8 GB
  expected hit rate:                   ~97%
  -> diminishing returns; 1.8 GB is the sweet spot

ORIGIN LOAD
  at 97% hit rate:  40,000 x 0.03 = 1,200 reads/s
  at 92% hit rate:  40,000 x 0.08 = 3,200 reads/s
  with NO cache:    40,000 reads/s   <- 33x the sized load

THE DESIGN CONSEQUENCE
  the database is provisioned for ~2,000 reads/s.
  a cache failure presents 40,000.
  -> MUST have: concurrency limit at the database client,
     load shedding above it, and a plan to warm the cache
     before shifting traffic back.
```

| Metric | Value | Note |
|---|---|---|
| Hit rate | 97% | 1.8 GB cache |
| Origin load | 1,200/s | sized for 2,000 |
| Cache down | 40,000/s | **33× over capacity** |
| Required | concurrency cap | + shedding |

> **The cache changes your failure model, not just your latency**  
> Once a cache absorbs 97% of reads, the database is no longer sized for the real traffic — it is sized for the miss traffic. That is an efficient and correct design, but it means the cache has become a tier-0 dependency whose failure is a database outage unless you explicitly bound the fall-through. Teams routinely add the cache and never revisit the failure analysis.

**When to use it**

- **Read-heavy workloads** where the same data is requested repeatedly — the higher the read/write ratio, the better the return.
- **Expensive computations or queries** whose results are reusable across requests.
- **Tolerable staleness**, where being a few seconds or minutes behind is acceptable.
- **Protecting a source of truth** that cannot be scaled easily or cheaply.
- **Anywhere the access distribution is skewed**, since a small cache captures a large share of traffic.

**When to avoid it**

- **Do not cache without a TTL.** Every invalidation bug becomes permanent.
- **Do not cache data that must be strictly current** — balances at the point of spending, inventory at the point of allocation, permissions at the point of enforcement.
- **Do not use `set` on write instead of `delete`**; it introduces an ordering hazard between concurrent writers.
- **Do not cache without a plan for the cold-start and cache-down cases**, which are where the outages are.
- **Do not cache uniformly-accessed data** — if every key is equally likely, hit rate tracks cache size over dataset size and the return is poor.

**Advantages**

- **Simple and universally understood**, with no special support required from the cache or the database.
- **Only cached data is what was actually requested**, so the cache naturally holds the working set.
- **Resilient to cache failure by design** — a miss is just a slower read, provided you bound the fall-through.
- **Works with any store**, since the cache has no knowledge of the data model.
- **Very high leverage**: at 97% hit rate the source sees 3% of traffic.

**Disadvantages**

- **The stale-write race is inherent**, and only TTL or versioned keys bound it.
- **Every read on a miss costs an extra round trip** — cache miss, then source, then cache write.
- **Cold caches are dangerous**, presenting full load to an under-provisioned source.
- **Invalidation logic is scattered** through the application wherever writes happen.
- **Stampedes on hot keys** require explicit coalescing to prevent.
- **The cache becomes a tier-0 dependency** once the source is sized for miss traffic.

**Trade-offs**

**Cache-aside versus the alternatives**

|  | Cache-aside | Read-through | Write-through | Write-behind |
|---|---|---|---|---|
| Who populates | Application | Cache library/layer | Cache on write | Cache on write, async to source |
| Cache holds | Only what was read | Only what was read | Everything written | Everything written |
| Write consistency | Invalidate; racy | Invalidate; racy | Strong-ish | Weak — buffered |
| Cache failure | Degrades to source | Degrades to source | Degrades to source | Risk of data loss |
| Complexity | In the application | In the cache layer | Moderate | High |

Cache-aside dominates in practice because it requires nothing special and degrades gracefully. Read-through is the same pattern with the fetch logic moved into a library, which is mostly an ergonomics improvement — the races are identical.

**How it fails**

**Cache-aside failure modes**

| Failure | Mechanism | Fix |
|---|---|---|
| Stale value persists | Slow reader wrote an old value after invalidation | TTL as backstop; versioned keys; delayed double-delete |
| Stampede on expiry | Many concurrent misses on one hot key | Single-flight coalescing; jittered TTL; stale-while-revalidate |
| Cold-start collapse | Cache restart presents 100% miss rate to the source | Concurrency limit at the source; warm before shifting traffic |
| Negative results not cached | Repeated misses for keys that do not exist | Cache the negative result with a short TTL |
| Synchronised expiry | Many keys written together expire together | Add jitter to TTLs |
| Cache and source diverge silently | Invalidation missed on some write path | One write path; TTL; periodic comparison sampling |
| Cache outage becomes a database outage | Unbounded fall-through | Concurrency cap plus shedding on the source path |

**Limits**

> **Sizing and tuning numbers**
>
> - **Hit rate versus working set**: with Zipfian access, caching the hot 10–30% typically yields 90–97% hit rate; beyond that, returns diminish sharply.
> - **Origin load = request rate × (1 − hit rate).** Going from 95% to 99% cuts origin load fivefold.
> - **TTL jitter**: ±10–20% prevents synchronised expiry of keys written together.
> - **Negative TTL** should be much shorter than positive TTL — typically seconds — so a newly created entity becomes visible quickly.
> - **Fall-through concurrency limit** should be sized to the source's actual capacity, not to the request rate.

**Alternatives**

| Pattern | Best for | Trade |
|---|---|---|
| Cache-aside | General read-heavy caching | Application owns invalidation; races |
| Read-through | Same, with fetch logic centralised | Cache layer must know how to load |
| Write-through | Read-after-write consistency matters | Write latency; caches data never read |
| Write-behind | Write-heavy with tolerable loss risk | Durability risk; complexity |
| Materialised view | Complex derived reads | Maintenance cost; staleness |
| No cache, scale the source | Low read/write ratio, strict freshness | Cost |

**In real systems**

- **Facebook's memcache deployment** documented leases and other mechanisms specifically to address the stale-set race and the thundering herd at scale.
- **Redis and Memcached in front of relational databases** is the single most common production caching topology in existence.
- **HTTP caching with `ETag` and `Cache-Control`** is cache-aside at the protocol layer, with the same staleness and invalidation trade-offs.
- **CDNs** apply the same pattern geographically, where origin shields exist precisely to coalesce misses and prevent stampedes.
- **ORM second-level caches** implement cache-aside automatically, which is convenient and hides the invalidation subtleties from developers who then meet them in production.

**Common mistakes**

- **Caching without a TTL**, making every invalidation bug permanent.
- **Using `set` instead of `delete` on write**, creating a writer-ordering hazard.
- **Invalidating before the database commit**, leaving a window for stale repopulation.
- **No coalescing on hot keys**, so expiry produces a stampede.
- **Unbounded fall-through**, turning a cache outage into a database outage.
- **Uniform TTLs on bulk-loaded keys**, causing synchronised expiry.
- **Caching data whose staleness has business consequences**, such as entitlements or balances at the point of use.

**The staff-level view**

Caching is the easiest large performance win available, and the fastest way to introduce a correctness problem that appears only under concurrency.

- **Require a TTL on every cached entry**, enforced in the cache client wrapper rather than by convention.
- **Re-do the capacity analysis after adding a cache.** The source is now sized for miss traffic, so the cache's failure is a load event that must be explicitly bounded.
- **Standardise single-flight coalescing in the cache library**, because stampedes are a library-level problem that every team otherwise rediscovers.
- **Prefer versioned keys for data where staleness is expensive**, since it converts invalidation from a race into a non-event.
- **Ask what must never be stale.** Balances at the point of spending, permissions at the point of enforcement and inventory at the point of allocation should read the source of truth regardless of how attractive caching them looks.

**Go deeper**

Cache-aside puts the application in charge: read from the cache, and on a miss read the source, populate the cache with a TTL, and return. On write, update the source and delete the cache entry — delete rather than set, because two concurrent writers could otherwise apply cache updates in the opposite order to their commits, leaving the cache permanently wrong.

Three failure modes define the pattern. A slow reader can write a stale value into the cache after a writer invalidated it, which nothing detects — so a TTL on every entry is mandatory as a backstop, and versioned keys eliminate the race entirely by making old entries unreachable rather than requiring deletion. A hot key expiring produces a stampede unless concurrent misses are coalesced through single-flight. And a cold or failed cache presents the source with the full request rate, typically 20–100× what it was provisioned for after caching succeeded.

That last point is the one teams miss: once the cache absorbs 97% of reads, the database is sized for miss traffic, so the cache has become a tier-0 dependency. A concurrency limit on the source path plus load shedding is what converts a cache outage into degraded latency rather than a database collapse. And some reads — permissions at enforcement, balances at spending, inventory at allocation — should never be cached at all.

Cache-aside is the most widely deployed caching pattern because it requires nothing from the cache or the database and degrades gracefully. Its subtleties are worth precision because nearly every system contains it.

**The pattern.** Read: check cache, on miss read source, populate with TTL, return. Write: commit to the source, then delete the cache entry. Deletion is strictly better than updating, because updating introduces a second ordering problem — two writers can commit in one order and update the cache in the opposite order, leaving a permanently wrong value with nothing to detect it. Deletion has a single outcome: the next reader repopulates from the authority.

**The stale-set race is inherent.** A reader that misses, then reads an old value from a slow query, can write that value into the cache *after* a concurrent writer has committed and invalidated. The delete removed nothing because the entry was not yet there. This cannot be prevented by ordering, only bounded — which is why a TTL on every entry is not hygiene but correctness, and should be enforced by the cache client rather than left to callers. The stronger fix is versioned keys: embed a version from the row itself in the cache key, so a write makes the old key unreachable and invalidation becomes a non-event, at the cost of one cheap version lookup per read.

**Stampedes and expiry synchronisation.** A popular key expiring under high concurrency sends every concurrent request to the source simultaneously. Single-flight coalescing — the first miss fetches while others wait on its result — turns thousands of queries into one, and belongs in the shared cache library rather than in each service. Jittering TTLs prevents keys populated together from expiring together, which otherwise produces periodic load spikes that look mysterious. Stale-while-revalidate goes further, serving the expired value while refreshing in the background, removing the miss from the critical path.

**The cache changes the failure model.** A successful cache means the source is now provisioned for miss traffic, perhaps 3% of the real request rate. Its failure therefore presents 30 times its capacity, and a cold cache after restart or failover does the same. The design obligation is a concurrency limit on the source client sized to actual database capacity, load shedding above it so excess requests fail fast rather than queueing, and a warming procedure before restoring traffic. This analysis must be re-done *after* the cache is deployed, and it routinely is not.

**What should not be cached.** The useful test is whether a read authorises an action or merely displays information. A permission check at the point of enforcement, a balance at the point of spending, an inventory count at the point of allocation — a few seconds of staleness in these produces a security incident, an overdraft, or an oversell, and the failure is silent. Those reads should go to the source with a conditional write. Everything feeding a display can be cached aggressively. Making this distinction explicit at design time matters because the temptation to cache a hot authorisation check is strong and the consequences are not visible in testing.

**Prove it — interview questions**

1. **[Basic] Describe the cache-aside read and write paths.**

   <details><summary>Model answer</summary>

   On read, check the cache; on a hit return it, on a miss read the source, write the value into the cache with a TTL, and return it. On write, update the source and then delete the cache entry. Deletion is preferred over updating because two concurrent writers could otherwise apply their cache updates in the opposite order to their database commits, leaving the cache permanently wrong — whereas deletion has only one outcome, which is that the next reader repopulates from the source of truth.

   </details>

2. **[Basic] Why does every cached entry need a TTL?**

   <details><summary>Model answer</summary>

   Because invalidation is racy and some of it will be missed. The classic case is a reader that misses, reads an old value from the database, and then — after a writer has committed and invalidated — writes that stale value into the cache. Nothing detects this, so without a TTL the stale value persists indefinitely. The TTL is the backstop that bounds the damage from every invalidation bug you have not found yet, which is why it should be enforced by the cache client rather than left to each caller.

   </details>

3. **[Senior] How do you prevent a cache stampede?**

   <details><summary>Model answer</summary>

   Single-flight coalescing: when a key misses, the first request fetches from the source while concurrent requests for the same key wait on that result, so one source query serves all of them. Complementary techniques are jittering TTLs so keys written together do not expire together, and stale-while-revalidate, where an expired value is served while a background refresh runs — which removes the miss from the critical path entirely. Without these, a popular key expiring at 40,000 requests per second sends 40,000 simultaneous queries to a database sized for a few thousand.

   </details>

4. **[Senior] What happens when the cache tier goes down?**

   <details><summary>Model answer</summary>

   Hit rate goes to zero and the source receives the full request rate, which after a successful caching deployment is typically 20 to 100 times what it is provisioned for. That is not a latency event, it is an outage unless it was designed for. The mitigations are a concurrency limit on the source client so only as many requests as the database can handle are admitted, load shedding above that so the excess fails fast rather than queueing, and a warming strategy before shifting traffic back — because a cold cache presents the same problem as no cache. Teams routinely add a cache and never revisit the capacity analysis, which is where this catches them.

   </details>

5. **[Staff] How would you eliminate the stale-set race rather than merely bounding it?**

   <details><summary>Model answer</summary>

   Versioned keys. Include a version — an updated-at timestamp or a counter from the row itself — in the cache key, so a write that changes the row automatically makes the old key unreachable rather than requiring a deletion that might race. Reads fetch the version first, which is a cheap lookup and can itself be cached briefly, then read the versioned key. Old entries are never invalidated; they simply age out via TTL. The cost is one extra lookup per read, which is usually negligible against the cost of a source query, and in exchange invalidation stops being a concurrency problem entirely. Where that extra lookup is too expensive, the pragmatic alternative is delayed double-delete — delete on write and again after a short delay — which is crude but clears stale values written by in-flight readers.

   </details>

6. **[Principal] How do you decide what should never be cached?**

   <details><summary>Model answer</summary>

   By asking where staleness has consequences that money or trust cannot absorb. The test I use is: if this value were a few seconds out of date at the moment it is acted upon, what happens? For a product description, nothing. For a permission check at the point of enforcement, someone retains access they should have lost — which is a security incident, not a latency issue. For an account balance at the point of spending, you permit an overdraft. For inventory at the point of allocation, you oversell. The common thread is that reads used to *authorise an action* must reflect the current state, while reads used to *display information* usually need not. So the architectural rule I would set is that the authorisation path and the allocation path read the source of truth with a conditional write, while everything feeding a display can be cached aggressively — and that distinction should be explicit in the design, because the temptation to cache a hot permission check is very strong and the failure is silent.

   </details>

---

### Write-through caching

*Write to the source and the cache together so the cache is never stale — at the cost of write latency and caching data nobody reads.*

**Flow:** `Write request` → `Source commit` → `Cache update` → `Acknowledgment`

> **The 30-second version**  
> Update the source and the cache together on every write, so readers immediately see what was written — paying cache latency per write and caching data that may never be read.

**The problem**

Cache-aside leaves a window: between the database commit and the cache invalidation, and again between invalidation and the next read, the cache can hold or repopulate stale data. For most workloads that is fine. For some — a user immediately viewing what they just saved, a configuration change that must take effect at once — it is not.

Write-through closes the window by making the cache part of the write path: every write updates both the source and the cache before acknowledging. The cache is therefore never behind the source.

> **What it actually buys**  
> Not general consistency — **read-after-write consistency**. Anyone reading immediately after a write sees the new value, because the cache was updated synchronously. It does not protect against a stale value written by a racing reader, and it does not help with data written by another system entirely.

**Mental model**

The cache sits in the write path rather than beside it. A write is not complete until both the source and the cache reflect it, which makes the cache an authority-adjacent component rather than a hint.

1. **Write path** — Commit to source, update cache, acknowledge. Ordering matters: source first, always.
2. **Read path** — Usually read-through — a miss loads from source and populates, same as cache-aside.
3. **Consistency** — Read-after-write within the system doing the writing. Not global consistency.
4. **Cost** — Every write pays a cache round trip, and the cache fills with data that may never be read.
5. **Failure handling** — What happens if the cache update fails after the source commit is the design decision that defines the pattern.

> **The cache write must not be able to fail the transaction**  
> If the source commit succeeds and the cache update fails, you cannot un-commit. Treat the cache update as best-effort: on failure, delete the key instead (so the next read repopulates) and never let a cache error surface as a write error. A write-through cache that can reject writes is a cache that has become a single point of failure for your write path.

**How it works**

**Write-through, with the failure path**

```text
WRITE
  db.write(k, v)                     -- source of truth first
  try:
      cache.set(k, v, ttl)
  except CacheError:
      try: cache.delete(k)           -- best effort
      except: pass                   -- TTL will fix it
  return success                     -- NEVER fail on cache error

READ (read-through)
  v = cache.get(k)
  if v is None:
      v = db.read(k); cache.set(k, v, ttl)
  return v

WHY SOURCE FIRST
  cache first, then db:
    if the db write fails, the cache holds a value that
    was never committed -> a phantom read of data that
    does not exist. Strictly worse than staleness.
```

1. **Write to the source first, always** — Committing to the cache before the source can publish data that never gets committed, which is worse than any staleness.
2. **Never fail a write because the cache failed** — Degrade to deletion, or to nothing plus TTL. The cache must not become a write-path dependency.
3. **Expect a lower hit rate than cache-aside** — Write-through populates on write, so it caches data that may never be read. In a write-heavy workload this wastes cache capacity on cold data.
4. **Combine with read-through for the miss path** — Write-through only covers what has been written since the cache started. Everything else still needs a read path that populates.
5. **It still does not give cross-system consistency** — If another service or a migration script writes the source directly, the cache is stale and write-through did nothing. Only TTL or a change stream covers that.
6. **Consider it for configuration and small hot datasets** — Where the whole dataset fits in cache and read-after-write matters, write-through is a clean fit with predictable behaviour.

**Where write-through helps and where it does not**

```text
HELPS
  user updates their profile, then immediately views it
    cache-aside: delete, then read repopulates -> fine
                 UNLESS a racing reader refilled with old data
    write-through: cache already has the new value -> correct

  feature flag toggled, must take effect immediately
    write-through: every reader sees it on the next read

DOES NOT HELP
  another service writes the same row directly
    -> the cache is stale; write-through never saw the write
  a batch job updates a million rows
    -> either the job must write through too (slow), or
       invalidate in bulk, or you rely on TTL
  a racing reader writes a stale value after your update
    -> same race as cache-aside; TTL still required

CONCLUSION: write-through improves the WRITE path's
consistency. It does not make the cache authoritative.
```

> **Write-through plus read-through is the clean combination**  
> Write-through alone only caches what has been written recently. Pairing it with read-through — where a miss loads and populates — gives you both: the cache is warm for read-heavy keys and never stale for recently written ones. Most library implementations offer both together for this reason.

**Worked example**

A feature flag service: small dataset, extremely read-heavy, changes must take effect immediately.

**Why write-through fits perfectly here**

```text
CHARACTERISTICS
  dataset:       ~5,000 flags x 200 B = 1 MB   -> fits entirely
  reads:         500,000/s across the fleet
  writes:        ~50/day
  requirement:   a disabled flag must stop being served
                 within seconds, everywhere

DESIGN
  write path:
    db.write(flag)                 -- source of truth
    cache.set(flag)                -- write-through
    publish invalidation on pub/sub -- fan out to local caches
  read path:
    local in-process cache (TTL 5s)
      -> shared cache
        -> database
  every layer write-through on the local update

WHY NOT CACHE-ASIDE
  with 500k reads/s, a flag expiry produces a stampede,
  and the invalidation race could leave a KILL SWITCH
  in the "enabled" state -> unacceptable

WHY THIS WORKS
  tiny dataset -> 100% hit rate, no eviction
  rare writes  -> write-through cost is irrelevant
  pub/sub fan-out bounds propagation to ~1s
  TTL 5s bounds any missed invalidation
```

| Metric | Value | Note |
|---|---|---|
| Dataset | 1 MB | fits entirely |
| Reads | 500k/s | 100% hit rate |
| Writes | 50/day | cost irrelevant |
| Propagation | ~1 s | **bounded by TTL** |

> **Write-through shines when writes are rare and freshness matters**  
> The pattern's cost is on the write path and its benefit is on the read path, so its value is highest when the read/write ratio is extreme *and* staleness has consequences. Configuration, feature flags, entitlement rules and routing tables fit this profile precisely. A high-write transactional table fits it poorly.

**When to use it**

- **Read-after-write consistency requirements**, where a user must immediately see what they just changed.
- **Small, hot datasets** that fit entirely in cache, so nothing is wasted on unread data.
- **Configuration, feature flags and routing rules**, where writes are rare and freshness matters.
- **Wherever the read/write ratio is extreme**, so the write-path cost is amortised over enormous read volume.
- **Combined with read-through**, to cover keys not written recently.

**When to avoid it**

- **Do not use it for write-heavy workloads** — you pay cache latency on every write and fill the cache with unread data.
- **Do not let a cache failure fail the write.** Degrade to delete-plus-TTL.
- **Do not write to the cache before the source**, which can publish uncommitted data.
- **Do not assume it makes the cache authoritative** — other writers bypass it entirely.
- **Do not use it for large datasets** where cached-on-write data will simply be evicted before it is read.

**Advantages**

- **Read-after-write consistency** for writes going through the path, which is the main reason to choose it.
- **No invalidation logic to get wrong** on the write path — the cache is updated, not deleted.
- **Predictable cache contents** for datasets small enough to fit entirely.
- **Reduced miss rate for recently written data**, which is often the hottest data.
- **Simple mental model**: the cache mirrors the source for anything written through it.

**Disadvantages**

- **Every write pays a cache round trip**, adding latency to the write path.
- **Caches data that may never be read**, wasting capacity in write-heavy workloads.
- **Does not cover writes from other systems**, migrations or batch jobs.
- **Still requires a TTL**, since racing readers can still write stale values.
- **Two things to keep consistent** on every write, with a failure path that must be designed explicitly.

**Trade-offs**

**Choosing between the write patterns**

|  | Cache-aside (invalidate) | Write-through | Write-behind |
|---|---|---|---|
| Write latency | One delete | One cache write | None (buffered) |
| Read-after-write | Racy | Consistent | Consistent locally |
| Cache contents | What was read | What was written | What was written |
| Durability risk | None | None | Real — buffered writes can be lost |
| Write-heavy fit | Good | Poor | Excellent |
| Cross-system writes | TTL only | TTL only | TTL only |

> **Framing the choice**  
> “The dataset is 1 MB and gets fifty writes a day against half a million reads per second, and a stale kill-switch is unacceptable — so write-through plus a short TTL is the right fit. For our main product catalogue, which is gigabytes with steady writes, cache-aside is better because write-through would fill the cache with items nobody views.”

**How it fails**

**Write-through failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Phantom data in cache | Cache written before the source commit failed | Always commit to the source first |
| Writes fail when the cache is down | Cache treated as a required dependency | Best-effort cache update; never fail the write |
| Cache full of cold data | Write-heavy workload populating on write | Use cache-aside instead; or cache only hot keys |
| Stale despite write-through | Another system wrote the source directly | TTL; change-data-capture invalidation |
| Stale after a batch job | Bulk update bypassed the write path | Bulk invalidation; or route batch writes through the same path |
| Write latency regression | Synchronous cache write on a hot path | Asynchronous update with delete-on-failure, accepting a small window |
| Cache and source diverge | Partial failure with no reconciliation | TTL as backstop; periodic sampled comparison |

**Limits**

> **Guidance**
>
> - **Added write latency**: typically 0.5–2 ms for an in-region cache write; significant only for very high write rates.
> - **Best fit** when the dataset fits entirely in cache, so nothing written is evicted before being read.
> - **Read/write ratio** should be high — write-through's cost scales with writes and its benefit with reads.
> - **TTL still required** as a backstop against racing readers and external writers.
> - **Cache update must be best-effort**: a failed cache write should never fail a committed database write.

**Alternatives**

| Approach | Gives | Costs |
|---|---|---|
| Cache-aside + invalidate | Simplicity, cache holds read data | Racy read-after-write |
| Write-through | Read-after-write consistency | Write latency; caches unread data |
| Write-behind | Fast writes, batched to source | Durability risk; complexity |
| Versioned keys | Race-free invalidation | Extra lookup per read |
| Change-data-capture invalidation | Covers writes from any source | Pipeline to build and operate |
| Read your own writes via session pinning | Consistency without touching the cache | Routing complexity |

Change-data-capture invalidation is the strongest general answer: by driving cache invalidation from the database's replication log, every writer — including migrations and other services — triggers invalidation, which is the one gap write-through cannot close.

**In real systems**

- **Feature flag and configuration services** almost universally use write-through with local caches plus pub/sub fan-out, because the dataset is small and staleness is unacceptable.
- **CPU caches** implement write-through as a hardware policy, with the same trade-off against write-back (the hardware name for write-behind).
- **Hibernate and similar ORM second-level caches** offer write-through as a strategy for entities that must be immediately consistent after update.
- **CDN purge-on-publish workflows** are write-through at the edge: the publish step pushes the new content rather than waiting for expiry.
- **Session stores** frequently use write-through so that a request immediately following a login sees the session, with a TTL bounding everything else.

**Common mistakes**

- **Writing to the cache before committing to the source**, publishing data that may never exist.
- **Failing the write when the cache write fails**, making the cache a write-path dependency.
- **Using it on write-heavy workloads**, paying latency and filling the cache with cold data.
- **Dropping the TTL** because “the cache is always current now”.
- **Assuming it covers batch jobs and other services**, which bypass the write path entirely.
- **Applying it to a dataset far larger than the cache**, so written entries are evicted before being read.

**The staff-level view**

Write-through is a narrow tool with a clear fit, and the common error is adopting it for the wrong workload because “never stale” sounds strictly better.

- **Check the read/write ratio and dataset size first.** Write-through's costs scale with writes and its benefits with reads; a write-heavy large dataset is the worst case.
- **Make the cache update best-effort in the shared client**, so no team can accidentally make the cache a write-path dependency.
- **Keep a TTL regardless.** Write-through does not cover external writers, batch jobs or racing readers.
- **For cross-system freshness, drive invalidation from the change log**, which is the only mechanism that catches every writer.
- **Be explicit that write-through gives read-after-write, not consistency.** Teams over-trust it and stop thinking about the other paths.

**Go deeper**

Write-through puts the cache in the write path: commit to the source, then update the cache, then acknowledge. The result is read-after-write consistency for anything written through that path, with no invalidation logic to get wrong. Reads are typically read-through, loading and populating on a miss.

Two rules define a correct implementation. Write to the source first — updating the cache before a commit that then fails publishes a value that was never committed, which is worse than staleness. And treat the cache update as best-effort: a failed cache write must never fail a committed database write, degrading instead to a delete or to the TTL, or the cache becomes a single point of failure for writes.

The fit is narrow. Costs scale with write volume and benefits with read volume, so it excels for small hot datasets with extreme read/write ratios — feature flags, configuration, routing rules — and is a poor choice for write-heavy or large datasets where written entries are evicted before being read. It also does not cover writes from batch jobs, migrations or other services, so a TTL remains mandatory, and change-log-driven invalidation is the only mechanism that catches every writer.

Write-through moves the cache from beside the write path to inside it, exchanging write latency for read-after-write consistency. Understanding precisely what it guarantees — and what it does not — is what keeps it from being over-trusted.

**What it guarantees.** Any read following a write that went through this path sees the new value. That is genuinely useful for the common product requirement of a user immediately viewing what they just saved, and for control-plane data like feature flags where a change must take effect promptly. What it does not guarantee is that the cache is authoritative: a batch job, a migration, an admin tool or another service writing the same rows bypasses the path entirely, and a racing reader can still write a stale value. A TTL therefore remains mandatory, exactly as in cache-aside.

**Two ordering and failure rules.** The source must be committed before the cache is updated; reversing this publishes values that may never be committed, and a phantom read is worse than a stale one because nothing corrects it. And the cache update must be best-effort — on failure, attempt a delete so the next read repopulates, and otherwise rely on TTL. A write-through implementation that can reject a write because the cache is unavailable has converted a performance optimisation into a write-path dependency, which is precisely the property a source of truth exists to avoid. Enforcing this in the shared cache client rather than in each service is the practical safeguard.

**Why the fit is narrow.** The cost is a cache round trip per write; the benefit is realised only if the written value is subsequently read. With a small dataset that fits entirely in cache and an extreme read/write ratio, every write is amortised across enormous read volume and nothing is wasted — configuration, flags and routing tables are the archetype. With a large dataset and steady writes, most written entries are evicted before anyone reads them, so you pay the latency and gain nothing, and cache-aside's property of caching only what was actually requested is strictly better.

**Composition.** Write-through covers only keys written since the cache started, so it is paired with read-through to populate everything else. In multi-tier deployments — an in-process cache backed by a shared cache — write-through at each layer plus a pub/sub invalidation fan-out bounds propagation to roughly a second, with a short TTL as the backstop. That combination is what feature flag services actually run, and the reason is a specific correctness case rather than performance: a kill switch being turned off must take effect promptly, and an invalidation race that leaves it enabled is unacceptable at exactly the moment it matters.

**The general answer to the coverage gap.** Neither write-through nor cache-aside invalidation sees writes that did not go through the instrumented path, and that blind spot is invisible until someone finds stale data nobody can explain. Driving invalidation from the database's replication log via change data capture catches every committed change regardless of origin, making freshness a property of the data rather than of the code path. It costs a pipeline to build and operate and adds propagation delay measured in hundreds of milliseconds, so the mature design uses it as the universal backstop while keeping synchronous write-through or invalidation for the read-after-write case — leaving TTL as a third line of defence rather than the only one.

**Prove it — interview questions**

1. **[Basic] What is write-through caching?**

   <details><summary>Model answer</summary>

   Every write updates both the source of truth and the cache before acknowledging, so the cache is never behind for data written through that path. Reads are usually read-through: a miss loads from the source and populates. The benefit is read-after-write consistency — someone reading immediately after a write sees the new value — and the cost is a cache round trip on every write plus caching data that may never be read.

   </details>

2. **[Basic] Why must you write to the source before the cache?**

   <details><summary>Model answer</summary>

   Because if the cache is updated first and the source commit then fails, the cache holds a value that was never committed — readers see data that does not exist anywhere authoritative. That is strictly worse than staleness, because staleness is eventually corrected while a phantom value is simply wrong. Committing to the source first means the worst case is a brief staleness window, which a TTL bounds.

   </details>

3. **[Senior] What should happen if the cache update fails after a successful commit?**

   <details><summary>Model answer</summary>

   The write must still succeed. You cannot un-commit the database, and failing the request would tell the caller the write did not happen when it did. The correct degradation is to attempt a delete instead, so the next read repopulates from the source, and if that also fails, to rely on the TTL. The general rule is that a cache must never be able to fail a write path; if it can, it has become a single point of failure for writes, which defeats the purpose of having a source of truth.

   </details>

4. **[Senior] When is write-through a poor choice?**

   <details><summary>Model answer</summary>

   When the workload is write-heavy or the dataset is much larger than the cache. Every write pays a cache round trip whose benefit is only realised if the value is subsequently read, and in a large dataset most written entries are evicted before anyone reads them — so you pay the cost and get nothing. It is also a poor choice when other systems, batch jobs or migrations write the same data, because those bypass the write path entirely and the cache is stale anyway, which means you needed a TTL or change-log-driven invalidation regardless.

   </details>

5. **[Staff] Design caching for a feature flag service serving 500,000 reads per second.**

   <details><summary>Model answer</summary>

   The dataset is tiny — a few thousand flags, around a megabyte — and writes are rare, so write-through fits almost perfectly: the cost is on the write path, which is negligible, and the benefit is on the read path, which is enormous. I would use a two-tier cache with an in-process cache in every service instance backed by a shared cache, both updated write-through, plus a pub/sub invalidation fan-out so a flag change propagates in about a second rather than waiting for expiry. A short TTL of a few seconds remains as the backstop for missed invalidations. The reason cache-aside would be wrong here is not performance but correctness of a specific case: a kill switch being disabled must take effect promptly, and cache-aside's invalidation race could leave it in the enabled state until TTL expiry, which for a kill switch is precisely the moment it matters.

   </details>

6. **[Principal] How would you get cache freshness guarantees that cover every writer, not just your own?**

   <details><summary>Model answer</summary>

   Drive invalidation from the database's change log rather than from application code. Write-through, and cache-aside invalidation for that matter, only cover writes that go through the path you instrumented — a migration script, an admin tool, a batch job, or another service writing the same table all bypass it silently, and that gap is invisible until someone notices stale data nobody can explain. Change-data-capture reading the replication log sees every committed change regardless of origin, so publishing invalidations from it makes freshness a property of the data rather than a property of the code path. The costs are real: a pipeline to build and operate, ordering and replay semantics to get right, and a propagation delay measured in hundreds of milliseconds rather than being synchronous. So I would combine them — write-through or invalidate-on-write for the synchronous read-after-write case, and change-log-driven invalidation as the universal backstop that makes the TTL a third line of defence rather than the only one.

   </details>

---

### Write-behind caching

*Acknowledge the write from the cache and persist it asynchronously — enormous throughput, at the price of a durability window you must design for.*

**Flow:** `Update` → `Durable buffer` → `Coalescing` → `Flush worker` → `Source`

> **The 30-second version**  
> Acknowledge from the cache and persist asynchronously in coalesced batches. Huge throughput gains on hot keys, paid for with a data-loss window unless the buffer is durable.

**The problem**

A metrics service receives two million counter increments per second. Each one is a tiny change to a row, and almost all of them are to the same few thousand rows. Writing each increment to a database individually is absurd: the same row is rewritten thousands of times per second, each write paying a full transaction cost for a change that will be superseded in a millisecond.

Write-behind — write-back, in hardware terminology — acknowledges the write immediately from the cache and persists it later, in batches, with intermediate values coalesced away. The hot row is written once per flush interval instead of thousands of times.

> **The trade is durability, and it is not negotiable away**  
> Between acknowledgement and flush, the write exists only in the cache. If the cache process dies, that data is gone — and the client was told it succeeded. Every write-behind design must answer: how much can we lose, and what does the user believe happened?

**Mental model**

A buffer with a flush policy. Writes land in memory, are merged where possible, and are pushed to the source periodically or when the buffer fills. The acknowledgement is decoupled from durability.

1. **Buffer** — In-memory (or ideally durably-logged) pending writes, keyed so that repeated writes to the same key coalesce.
2. **Coalescing** — The main performance win: a thousand increments to one counter become one write. Only works for keys written repeatedly.
3. **Flush trigger** — Time-based (every N ms), size-based (buffer full), or both. This interval is your data-loss window.
4. **Durability gap** — The time between acknowledging and persisting. Everything about the risk profile follows from its size.
5. **Read path** — Reads must see buffered writes, which means reading from the cache, not the source — the cache is now authoritative for recent data.

> **The cache becomes the source of truth, temporarily**  
> This is the property that makes write-behind categorically different from other caching patterns. For the duration of the buffer window, the cache holds state that exists nowhere else. It is no longer a cache — it is a write buffer that happens to serve reads, and it must be operated with the durability and availability expectations of a database, not of a cache.

**How it works**

**Write-behind with a durable buffer**

```text
NAIVE (memory only) - acceptable only for loss-tolerant data
  write:  buffer[k] = v; return OK      <- acknowledged
  flush:  every 100ms, db.write_batch(buffer); buffer.clear()
  crash:  up to 100ms of acknowledged writes VANISH

DURABLE BUFFER - the pattern for anything that matters
  write:  append to a replicated log (Kafka/WAL/AOF)
          update in-memory state
          return OK                    <- durable in the log
  flush:  consumer reads the log, batches, writes to the source,
          commits its offset
  crash:  replay from the last committed offset; nothing lost

NOW THE TRADE IS DIFFERENT
  you have not removed durability, you have moved it to a
  cheaper, append-only, sequential store — and gained
  coalescing and batching on the way to the expensive one.
```

1. **Decide the loss budget explicitly, in the requirements** — “We may lose up to 200 ms of counter increments” is an acceptable statement. Discovering it during an incident is not.
2. **Prefer a durable buffer to a memory-only one** — Appending to a replicated log before acknowledging keeps the throughput benefit while removing the data-loss window. This is the single most important design upgrade available.
3. **Coalescing is the actual win** — If keys are written once each, write-behind only batches — useful but modest. If keys are written repeatedly, coalescing can reduce source writes by orders of magnitude.
4. **Bound the buffer and apply backpressure** — If the source cannot keep up, the buffer grows without limit and then you lose all of it. Cap it, and reject or block writers above the cap.
5. **Reads must be served from the buffer** — Otherwise a read immediately after a write returns the old value from the source. This makes the cache a required read dependency too.
6. **Watch ordering across keys** — Batched flushes do not preserve the relative order of writes to different keys, so anything with cross-key invariants is unsuitable.

**Coalescing: where the orders of magnitude come from**

```text
WORKLOAD  2,000,000 counter increments/s over 5,000 counters

WRITE-THROUGH
  2,000,000 database writes/s        -> impossible

WRITE-BEHIND, flush every 200ms
  each counter accumulates ~80 increments per window
  flush writes 5,000 rows every 200ms
  = 25,000 database writes/s         -> 80x reduction
  and each write is a batch, not an individual transaction
  -> effective reduction is larger still

WHEN COALESCING DOES NOT APPLY
  2,000,000 DISTINCT keys written once each
  -> no coalescing; only batching
  -> still valuable (batch transactions) but ~5-10x, not 80x
```

> **Backpressure is mandatory, not optional**  
> A write-behind buffer that grows when the source is slow is a bomb: it absorbs load beautifully until memory runs out, at which point every buffered write is lost simultaneously — the maximum possible damage at the worst possible moment. Bound the buffer and apply backpressure to writers, so overload produces slow or rejected writes rather than a catastrophic loss.

**Worked example**

View counters on a content platform: two billion views a day, exact counts unnecessary, displayed everywhere.

**Two designs, different risk profiles**

```text
REQUIREMENT
  display a view count on every item
  2B views/day = ~23,000/s average, 70,000/s peak
  accuracy: approximate is fine; losing a few seconds
            of counts is acceptable
  BUT: counts must never go BACKWARDS (user-visible)

DESIGN A - memory buffer (simplest)
  increment in Redis, flush aggregated deltas to the
  database every 10s
  risk: a Redis failure loses up to 10s of counts
        -> counts do not go backwards (DB is behind, not ahead)
        -> acceptable given the stated requirement
  cost: Redis is now holding un-persisted state

DESIGN B - durable buffer (recommended at scale)
  append view events to a partitioned log
  stream processor aggregates per item per window
  writes aggregates to the database
  serves current counts from the processor's state store
  risk: none on loss - log is replicated
  cost: a streaming pipeline to operate

DECIDING
  A is right if the loss budget genuinely covers 10s and
    Redis is replicated with AOF persistence.
  B is right if views feed billing, ranking, or anything
    where "approximately" stops being acceptable.
```

| Metric | Value | Note |
|---|---|---|
| Peak rate | 70k/s | increments |
| DB writes | ~500/s | **140× reduction** |
| Loss window | 10 s | design A |
| Loss window | 0 | design B, durable log |

> **Ask what the number is used for, not just how accurate it must be**  
> A view count displayed on a page can lose ten seconds of increments harmlessly. The same number feeding a creator payout or a ranking algorithm cannot, because the loss is systematic rather than random — it always occurs during incidents, which correlate with traffic spikes. The question that decides the architecture is not “how accurate?” but “what decisions does this number drive?”

**When to use it**

- **Very high write rates to a small number of keys**, where coalescing produces order-of-magnitude reductions: counters, metrics, gauges, last-seen timestamps.
- **Loss-tolerant data** where a bounded window of missing writes is genuinely acceptable.
- **Smoothing bursty writes** against a source with limited write capacity.
- **With a durable buffer** — a replicated log — where you want the throughput benefit without the loss window.
- **Session and presence data**, where the value is short-lived anyway and exact persistence has little meaning.

**When to avoid it**

- **Do not use it for financial, inventory or identity data** without a durable buffer; a lost acknowledged write is a lost transaction.
- **Do not use a memory-only buffer** for anything a user believes was saved.
- **Do not use it where cross-key ordering matters**, since batched flushes reorder relative to each other.
- **Do not let the buffer grow unbounded**; cap it and apply backpressure.
- **Do not use it when writes are to distinct keys** — you get batching but no coalescing, and the risk may not be worth it.

**Advantages**

- **Write latency drops to cache latency**, since the source is not on the critical path.
- **Coalescing can reduce source writes by orders of magnitude** for repeatedly written keys.
- **Batching amortises transaction overhead**, further multiplying the reduction.
- **Absorbs bursts**, smoothing spiky write traffic against a source with fixed capacity.
- **Protects a limited source**, which may be the only way to serve the write rate at all.

**Disadvantages**

- **Data loss window** between acknowledgement and persistence, unless a durable buffer is used.
- **The cache becomes a required dependency for reads**, since it holds the only current state.
- **Cross-key ordering is not preserved**, ruling out anything with multi-key invariants.
- **Unbounded buffers turn overload into catastrophic loss**, so backpressure is mandatory.
- **Complex failure handling**: what happens to buffered writes when the source rejects them?
- **Harder to reason about**, because acknowledgement no longer means persistence.

**Trade-offs**

**Buffer design choices**

| Buffer | Loss window | Throughput gain | Operational cost |
|---|---|---|---|
| In-memory only | Full flush interval | Highest | Low |
| In-memory + replication | Replica lag only | High | Moderate |
| Append to local WAL | Bounded by fsync policy | High | Low; single-node risk |
| Replicated log (Kafka-style) | None | High | Significant — a pipeline to run |
| Write-through instead | None | None | Lowest |

The middle rows are where most production systems should sit. A memory-only buffer is tempting because it is trivial, but the difference between “we lose 200 ms of writes if a process dies” and “we lose nothing” is usually worth the operational cost, and it is almost always cheaper to add at design time than after the first incident.

**How it fails**

**Write-behind failure modes**

| Failure | Consequence | Mitigation |
|---|---|---|
| Cache process dies | All buffered acknowledged writes lost | Durable buffer; replication; bounded flush interval |
| Source becomes slow | Buffer grows; eventually all of it is lost | Bound the buffer; backpressure; shed writes |
| Source rejects a batch | Whole batch fails; retry semantics unclear | Idempotent writes; dead-letter path; per-item fallback |
| Read returns stale value | Reader hit the source instead of the buffer | Serve reads from the buffer; make it a read dependency explicitly |
| Counts go backwards after a failure | Flushed aggregate lost, source shows older value | Monotonic aggregates; never decrease a displayed counter |
| Ordering violated across keys | Cross-key invariant broken | Do not use write-behind for such data |
| Silent divergence | Some flushes failed and were dropped | Track flush failures as a first-class metric; reconcile |

> **Acknowledged-then-lost is the worst kind of data loss**  
> It is invisible to the client, which has already displayed a success message, and invisible to the source, which never saw the write. There is no error, no retry, and no record that anything was missing. This is why the loss budget must be a stated requirement rather than an implementation detail — someone outside engineering needs to have agreed to it.

**Limits**

> **Design numbers**
>
> - **Flush interval** sets the loss window directly: 100 ms–10 s typical, with shorter meaning less loss and less coalescing.
> - **Coalescing ratio** = writes per key per flush window; it is the entire performance argument, so measure it.
> - **Buffer cap** should be sized so that losing it is survivable — if losing the whole buffer is unacceptable, it needs to be durable, not smaller.
> - **Source write rate** after coalescing is distinct keys per flush window ÷ interval; size the source for that.
> - **Backpressure threshold** should trigger well before the buffer cap, so writers slow down rather than failing abruptly.

**Alternatives**

| Approach | Durability | Throughput | When |
|---|---|---|---|
| Write-through | Full | Limited by source | Default; correctness matters |
| Write-behind, memory buffer | Loss window | Very high | Loss-tolerant counters |
| Write-behind, durable log | Full | Very high | High volume that also matters |
| Batched writes in the application | Full (sync batch) | High | Simple bulk ingest |
| Atomic increment in the source | Full | Moderate | When the source supports it and volume permits |
| Approximate counters (HLL, sampling) | Full for the approximation | Very high | Cardinality and rough counts |

The last row deserves consideration whenever the requirement is “roughly how many”. Probabilistic structures give bounded-error answers at a fraction of the storage and write cost, and they sidestep the durability question entirely.

**In real systems**

- **CPU write-back caches** are the original instance of this pattern, with the same trade: enormous throughput gain, and data in cache that memory has not yet seen.
- **View, like and impression counters** on large platforms are almost universally write-behind, typically through a durable log rather than a memory buffer.
- **Kafka plus a stream processor writing aggregates** is write-behind with a durable buffer, and is the standard architecture for high-volume counting at scale.
- **Redis with AOF persistence** used as a write buffer occupies the middle ground: bounded loss depending on fsync policy.
- **Database group commit** is a bounded form of the same idea — batching commits to amortise fsync, without acknowledging before durability.

**Common mistakes**

- **Memory-only buffering of data users believe was saved.**
- **Unbounded buffers**, so overload becomes total loss.
- **No backpressure**, letting the buffer absorb load it cannot flush.
- **Reading from the source instead of the buffer**, returning stale values after a write.
- **Using it for data with cross-key ordering requirements.**
- **Not measuring the coalescing ratio**, so the performance justification is assumed rather than known.
- **Treating flush failures as background noise** instead of a first-class alert.

**The staff-level view**

Write-behind is the caching pattern with genuine data-loss consequences, so the Staff-level work is making the risk explicit and usually eliminating it.

- **Make the loss budget a written requirement with a business owner.** “We may lose up to N seconds of X” must be agreed outside engineering, because the failure is silent and the client was told it succeeded.
- **Push for a durable buffer.** Appending to a replicated log before acknowledging keeps the throughput and coalescing benefits while removing the loss window; this is nearly always the right upgrade.
- **Require bounded buffers and backpressure in review.** An unbounded buffer converts overload into total loss at the worst moment.
- **Ask what the data is used for, not how accurate it must be.** A count that feeds billing or ranking has a different risk profile from one that decorates a page, even though both are “approximate”.
- **Treat the buffer as a database**, not a cache: it holds state that exists nowhere else, so it needs the availability, replication and monitoring of a datastore.

**Go deeper**

Write-behind acknowledges a write immediately from the cache and flushes to the source later, in batches. The performance win comes chiefly from coalescing: a counter incremented eighty times within a flush window becomes one source write, which is where order-of-magnitude reductions originate. If every write is to a distinct key you get only batching, worth perhaps five to ten times and rarely worth the risk.

The risk is that between acknowledgement and flush, the data exists only in the buffer — so a crash loses writes the client was told had succeeded. That is the most insidious kind of data loss, since there is no error and no record of what is missing. The design upgrade that removes it is a durable buffer: append to a replicated log before acknowledging, then have a consumer batch and apply to the source. You keep the latency, batching and coalescing while moving durability to a cheap sequential store.

Two further constraints. The buffer must be bounded with backpressure, because an unbounded buffer absorbs overload beautifully until memory is exhausted and then loses everything at once — maximum damage at the worst moment. And reads must be served from the buffer, since it holds the only current state, which makes it a read dependency and means it should be operated like a datastore rather than a cache.

Write-behind is the only caching pattern with genuine data-loss consequences, which makes precision about its trade-offs unusually important.

**Where the gain comes from.** Two mechanisms, of very different magnitude. Batching amortises per-transaction overhead across many writes, typically worth five to ten times. Coalescing collapses repeated writes to the same key within a flush window into one — a counter incremented eighty times per window produces one write instead of eighty — and this is where order-of-magnitude reductions originate. The coalescing ratio is therefore the number that justifies the pattern, and it must be measured rather than assumed: a workload writing two million distinct keys once each gets batching only, and probably should not accept the durability risk for it.

**The durability gap.** Between acknowledgement and flush, the data exists solely in the buffer. If the process dies, those writes vanish — and the client has already been told they succeeded. This is the worst class of data loss because it is silent on both sides: no error reaches the client, and the source never saw the write, so nothing indicates that anything is missing. The loss budget must therefore be a written requirement with a business owner, not an implementation detail an engineer chose.

**The upgrade that removes the trade.** Appending to a replicated log before acknowledging, with a consumer batching into the source and committing offsets as it goes, retains the low write latency, the batching and the coalescing while eliminating the loss window entirely — a crash simply replays from the last committed offset. Framed correctly, this is not “write-behind with extra steps”: it is recognising that durability was moved to a cheap sequential append-only store, and that the buffer's role is reducing load on the expensive one. Since the performance benefit comes almost entirely from coalescing and batching rather than from skipping durability, the memory-only variant usually takes a real risk for a marginal additional gain.

**Backpressure is not optional.** A buffer that grows when the source is slow absorbs overload beautifully — latency unaffected, writes succeeding — right up to memory exhaustion, at which point everything buffered is lost simultaneously. That is maximum damage during an already-degraded state. Bounding the buffer and applying backpressure well below the cap converts overload into slow or rejected writes, which are visible, recoverable, and honest to the caller.

**Two further constraints.** Reads must be served from the buffer, because it holds the only current state — which makes it a read dependency and means it must be replicated, monitored and capacity-planned like a datastore rather than treated as a disposable cache. And batched flushes do not preserve ordering across keys, so any data with cross-key invariants is unsuitable regardless of durability arrangements. Finally, when the requirement is genuinely “roughly how many,” probabilistic structures such as sketches and sampling deserve consideration first: they give bounded-error answers at a fraction of the write cost and sidestep the durability question entirely.

**Prove it — interview questions**

1. **[Basic] What is write-behind caching and what does it trade?**

   <details><summary>Model answer</summary>

   Writes are acknowledged from the cache immediately and persisted to the source asynchronously, in batches, with repeated writes to the same key coalesced. The gain is very low write latency and a potentially enormous reduction in source writes. The trade is durability: between acknowledgement and flush, the data exists only in the buffer, so a crash loses writes the client was told had succeeded.

   </details>

2. **[Basic] Where does the big performance win come from?**

   <details><summary>Model answer</summary>

   Coalescing, more than batching. If the same key is written many times within a flush window — a view counter incremented eighty times — all of those collapse into one source write. That is where order-of-magnitude reductions come from. If every write is to a distinct key, you only get batching, which is worth perhaps five to ten times and may not justify the durability risk. So the coalescing ratio is the number that decides whether the pattern is worth using at all.

   </details>

3. **[Senior] How do you make write-behind safe for data that matters?**

   <details><summary>Model answer</summary>

   Give the buffer real durability. Instead of holding pending writes in memory, append them to a replicated log before acknowledging, then have a consumer batch and apply them to the source, committing its offset as it goes. You keep the low write latency, the batching and the coalescing, but a crash replays from the last committed offset and nothing is lost. That reframes the pattern: you have not removed durability, you have moved it to a cheap sequential append-only store and used the buffer to reduce load on the expensive one.

   </details>

4. **[Senior] Why is an unbounded buffer dangerous?**

   <details><summary>Model answer</summary>

   Because it absorbs overload gracefully right up to the moment it fails completely. If the source slows down, the buffer grows, which looks fine — latency is unaffected and writes keep succeeding — until memory is exhausted, at which point every buffered write is lost at once. That is the maximum possible damage at the worst possible time, during an existing incident. The correct behaviour is to cap the buffer and apply backpressure well below the cap, so overload manifests as slower or rejected writes, which is visible and recoverable.

   </details>

5. **[Staff] Design counting for two billion page views a day.**

   <details><summary>Model answer</summary>

   First I would establish what the count is used for, because that determines the durability requirement more than the volume does. If it decorates a page, a bounded loss window is fine; if it feeds creator payouts or ranking, it is not, because losses are systematic — they happen during incidents, which correlate with traffic spikes, so the error is biased rather than random. Assuming it matters, I would append view events to a partitioned log, have a stream processor aggregate per item per time window, serve current counts from the processor's state store, and write aggregates to the database periodically. That gives write-behind's coalescing — roughly seventy thousand increments per second becoming a few hundred database writes — with no loss window, because the log is replicated. I would also make displayed counts monotonic, so that no failure mode can ever show a user a number that went backwards, which is the one visible symptom people actually notice.

   </details>

6. **[Principal] How do you govern write-behind so its risks stay visible?**

   <details><summary>Model answer</summary>

   By making the loss budget a stated, owned requirement rather than an implementation detail. Acknowledged-then-lost is the most insidious kind of data loss: the client has already displayed success, the source never saw the write, there is no error and no record that anything is missing — so nothing surfaces it unless someone decided in advance what was acceptable and instrumented for it. I would require that any write-behind component document its loss window, name a business owner who has agreed to it, and export flush failures and buffer depth as first-class alerting metrics rather than debug logs. Structurally, I would push the default toward durable buffers, because the throughput benefit comes almost entirely from coalescing and batching rather than from skipping durability, so the memory-only variant usually takes a real risk for a marginal additional gain. And I would insist the buffer be operated like a datastore — replicated, monitored, capacity-planned — since for the duration of the window it holds state that exists nowhere else.

   </details>

---

### TTL and invalidation

*Two ways to bound staleness: expire on a timer, or actively invalidate on change. Use both — one is a guarantee, the other an optimisation.*

**Flow:** `Source change` → `Invalidation event` → `Cache entry` → `TTL deadline` → `Refresh`

> **The 30-second version**  
> TTL bounds staleness unconditionally; invalidation reduces it typically but can be lost. Use TTL as the guarantee, invalidation as the optimisation, and jitter everything.

**The problem**

Every cached value is a claim about the past. The question is how far in the past it may be, and how you bound that. There are exactly two mechanisms: wait for a timer to expire the entry, or actively tell the cache the entry is now wrong.

Teams reach for invalidation because it sounds precise, and then discover that invalidation is delivery over a network — it can be lost, delayed, reordered, or simply never sent because someone wrote the data through a path nobody instrumented. TTL is the only mechanism that cannot fail to fire.

> **The correct framing**  
> **TTL is the guarantee; invalidation is the optimisation.** TTL bounds staleness unconditionally, because it requires no message to arrive. Invalidation reduces typical staleness from the TTL to milliseconds, but it can be missed — and a design that depends on it is one lost message from permanent inconsistency.

**Mental model**

Think of TTL as the expiry date printed on the package and invalidation as a recall notice. The expiry date always works. The recall notice works only if it reaches you, but when it does, it works much faster.

1. **TTL** — A deadline attached to the entry. Requires no coordination, cannot be lost, and bounds worst-case staleness exactly.
2. **Invalidation** — An active message — delete this key. Reduces typical staleness dramatically, but is a distributed message with all the usual failure modes.
3. **Refresh-ahead** — Proactively reload before expiry so no request pays the miss. Reduces latency, not staleness.
4. **Stale-while-revalidate** — Serve the expired value immediately while refreshing in the background. Trades a little extra staleness for removing the miss from the critical path.
5. **Versioned keys** — Change the key when the data changes, so old entries become unreachable. Invalidation without messages.

> **Invalidation only covers the writers you instrumented**  
> A migration script, an admin tool, a batch job, or another service writing the same table produces no invalidation. The cache is stale and nothing will notice. This is why TTL must exist even in a system with perfect invalidation, and why change-data-capture — which sees every committed write regardless of origin — is the only invalidation mechanism with full coverage.

**How it works**

**Choosing a TTL from the staleness requirement**

```text
START FROM THE PRODUCT REQUIREMENT, NOT THE HIT RATE

  "a price change must be visible within 60 seconds"
    -> TTL <= 60s   (with invalidation making it typically <1s)

  "a revoked permission must take effect within 5 seconds"
    -> TTL <= 5s, OR do not cache this at all

  "a product description can be a day old"
    -> TTL = hours; invalidation for editor experience

THEN CHECK THE COST
  origin load = rate x (1 - hit rate)
  hit rate falls as TTL falls, but NOT linearly:
    for a key read 100x/minute, a 60s TTL still yields
    ~99% hit rate. Short TTLs are cheaper than people assume
    for hot keys, and expensive for long-tail keys.

ADD JITTER
  ttl = base * (0.9 + random()*0.2)
  prevents keys populated together from expiring together
```

1. **Set TTL from the staleness requirement, then verify the cost** — Working backwards from hit rate produces a number nobody can defend to a product owner. Working forwards from “how stale may this be” produces one you can.
2. **Always jitter** — Keys populated in the same batch — after a deploy, a cache warm, or a bulk import — will otherwise expire simultaneously and produce a synchronised stampede.
3. **Use short negative TTLs** — Caching “not found” prevents repeated lookups for missing keys, but a long negative TTL hides a newly created entity. Seconds, not minutes.
4. **Prefer stale-while-revalidate on hot keys** — Serving the stale value while refreshing in the background removes the miss latency entirely and eliminates the stampede, at the cost of a slightly wider staleness window.
5. **Consider versioned keys where correctness matters** — Embedding a version in the key makes stale entries unreachable rather than requiring a message to arrive — invalidation with no delivery problem.
6. **Drive invalidation from the change log for full coverage** — Change data capture sees every committed write, including from tools and other services, which application-level invalidation structurally cannot.

**The four staleness strategies compared**

```text
TTL ONLY
  staleness: up to TTL, always
  complexity: none
  use: data with a genuine tolerance window

TTL + INVALIDATE ON WRITE
  staleness: typically <1s, worst case TTL
  complexity: invalidation call on every write path
  use: the common default

TTL + CDC INVALIDATION
  staleness: typically <1s, worst case TTL, covers ALL writers
  complexity: a change-capture pipeline
  use: multiple writers, or correctness-sensitive data

VERSIONED KEYS
  staleness: zero for the version you looked up
  complexity: version lookup per read
  use: where a stale read is expensive and races unacceptable
```

> **Stale-while-revalidate is usually the best default for hot reads**  
> It converts the expensive event — a cache miss on a hot key — into a background refresh nobody waits for. The reader gets an answer immediately, the origin sees one request instead of thousands, and the only cost is that the served value may be slightly older than the strict TTL. For anything where a second of extra staleness is harmless, it strictly dominates plain expiry.

**Worked example**

A pricing service: prices change a few times a day but must propagate quickly, and the read volume is very high.

**Layering the mechanisms**

```text
REQUIREMENT
  price reads:   200,000/s
  price changes: ~500/day
  propagation:   a price change must be live within 30s
                 (contractual)
  never:         serve a price the customer will not be charged

DESIGN
  cache key:  price:{sku}:{price_version}
              version comes from the product row
  TTL:        300s with +/-10% jitter  (the guarantee)
  invalidation: CDC on the prices table publishes to pub/sub,
                every cache tier drops the key   (the optimisation)
  read path:  stale-while-revalidate on the shared cache

STALENESS ANALYSIS
  typical:     <1s   (CDC invalidation)
  if pub/sub is degraded: <=300s  (TTL)
  contractual requirement is 30s
  -> TTL of 300s DOES NOT meet the requirement on its own
  -> either lower TTL to 30s, or treat pub/sub as tier-0

DECISION
  lower TTL to 25s with jitter.
  cost: hit rate for a SKU read 200x/s is still ~99.98%
  -> the guarantee now meets the contract without depending
     on message delivery.
```

| Metric | Value | Note |
|---|---|---|
| Typical staleness | <1 s | CDC invalidation |
| Guaranteed staleness | 25 s | **TTL alone** |
| Hit rate | 99.98% | hot keys unaffected |
| Requirement | 30 s | met without messages |

> **Test the guarantee, not the typical case**  
> The question that reveals a broken design is: *if invalidation stopped working entirely tomorrow, would we still meet our requirement?* If the answer is no, the requirement is being met by an optimisation rather than by a guarantee, and one degraded message bus turns a performance issue into a contractual or safety failure.

**When to use it**

- **TTL on every cached entry, always** — it is the only unconditional bound on staleness.
- **Invalidation on top**, where typical staleness matters for user experience or correctness.
- **Stale-while-revalidate** for hot keys, to remove miss latency and stampedes.
- **Negative caching with short TTLs**, to absorb repeated lookups for missing keys.
- **Versioned keys** where a stale read is expensive and invalidation races are unacceptable.

**When to avoid it**

- **Do not cache without a TTL**, even with invalidation — one missed message becomes permanent.
- **Do not derive TTL from a desired hit rate**; derive it from the staleness requirement and then check the cost.
- **Do not use uniform TTLs on bulk-populated keys**, which causes synchronised expiry.
- **Do not use long negative TTLs**, which hide newly created entities.
- **Do not rely on application-level invalidation alone** when multiple systems write the same data.

**Advantages**

- **TTL is unconditional** — no message, no coordination, no delivery guarantee needed.
- **Invalidation reduces typical staleness by orders of magnitude** over TTL alone.
- **Stale-while-revalidate removes miss latency and stampedes** simultaneously.
- **Versioned keys eliminate invalidation races**, converting a distributed problem into a lookup.
- **Layering gives defence in depth**: fast typical behaviour with a hard worst-case bound.

**Disadvantages**

- **TTL alone means serving stale data for the whole window**, even when the source changed immediately after caching.
- **Invalidation can be lost, delayed or reordered**, and cannot cover writers you did not instrument.
- **Short TTLs increase origin load** for long-tail keys, though hot keys are barely affected.
- **Change-data-capture invalidation requires a pipeline** with its own operational burden.
- **Multiple cache tiers multiply the problem**, since each must be invalidated and each has its own TTL.

**Trade-offs**

**Staleness mechanisms**

| Mechanism | Typical staleness | Worst case | Coverage |
|---|---|---|---|
| TTL only | Half the TTL on average | The TTL | Complete |
| Invalidate on write | Milliseconds | The TTL, if the message is lost | Only instrumented writers |
| CDC invalidation | Hundreds of ms | The TTL | Every committed write |
| Versioned keys | Zero | Zero for that version | Every write that bumps the version |
| Stale-while-revalidate | TTL + refresh time | TTL + refresh time | Complete |

> **What separates a strong answer**  
> “I'd set the TTL from the contractual staleness requirement — 25 seconds, jittered — so the guarantee holds even if invalidation is completely broken. Then I'd add CDC-driven invalidation to bring typical staleness under a second, treating it as an optimisation rather than as the thing meeting the requirement.”

**How it fails**

**Staleness failures**

| Failure | Cause | Fix |
|---|---|---|
| Stale data persists indefinitely | Invalidation missed and no TTL | TTL on every entry, without exception |
| Synchronised expiry spike | Uniform TTLs on bulk-populated keys | Jitter every TTL by ±10–20% |
| New entity invisible for minutes | Long negative TTL cached the miss | Short negative TTL; invalidate on create |
| Some writers never invalidate | Batch jobs and admin tools bypass the code path | CDC-driven invalidation; TTL backstop |
| Invalidation arrives before the commit is visible | Race between message and replication | Invalidate after commit; delayed double-delete; versioned keys |
| One tier stale, another fresh | Multi-tier caches invalidated inconsistently | Invalidate all tiers; shorter TTL on local tiers |
| Requirement met only when the bus is healthy | Guarantee resting on an optimisation | Set TTL to the requirement; treat invalidation as a bonus |

**Limits**

> **Numbers to work with**
>
> - **Hit rate for a hot key** barely changes with TTL: a key read 200 times per second retains ~99.98% hit rate even at a 25-second TTL.
> - **Long-tail keys** are where short TTLs cost — they may be read once per TTL window, giving near-zero hit rate.
> - **Jitter** of ±10–20% is enough to break synchronised expiry.
> - **Negative TTL** should be seconds, typically 1–30, versus minutes or hours for positive entries.
> - **Invalidation propagation** through a healthy pub/sub tier is typically under a second; assume it can be minutes when degraded.

**Alternatives**

| Approach | Best for | Cost |
|---|---|---|
| TTL only | Data with a genuine tolerance window | Serving stale for the full window |
| TTL + write invalidation | The common default | Invalidation on every write path |
| TTL + CDC invalidation | Multiple writers; correctness-sensitive | A change-capture pipeline |
| Versioned keys | Races unacceptable | Version lookup per read |
| No caching | Authorisation, balances, allocation | Full source load |
| Push-based subscription | Small hot datasets (flags, config) | Connection management; fan-out |

**In real systems**

- **HTTP `Cache-Control` with `max-age` and `stale-while-revalidate`** is this exact model standardised at the protocol level, including the background refresh behaviour.
- **CDN purge APIs** are invalidation with all the usual delivery problems, which is why CDNs also enforce a maximum TTL.
- **Debezium and similar CDC tools** driving cache invalidation give the coverage that application-level invalidation cannot.
- **DNS TTLs** are the classic demonstration that the guarantee is only as good as the layer that honours it — caches above you may clamp or ignore it.
- **Feature flag services** use push-based invalidation with a short TTL backstop, because a kill switch that fails to propagate is precisely the wrong failure.

**Common mistakes**

- **No TTL because invalidation is “reliable”.**
- **Uniform TTLs**, producing synchronised expiry storms.
- **Long negative TTLs**, hiding newly created records.
- **Invalidating before the write is visible**, so a reader repopulates the old value.
- **Deriving TTL from a target hit rate** rather than from a staleness requirement.
- **Relying on application-level invalidation** when batch jobs and other services write the same data.
- **Forgetting local in-process caches**, which have their own TTLs and no invalidation at all.

**The staff-level view**

Staleness is a product requirement wearing a technical costume, and the most common architectural error is meeting it with an optimisation.

- **Ask the guarantee question in review**: if invalidation stopped working entirely, would we still meet the requirement? If not, the TTL is wrong.
- **Derive TTLs from stated staleness tolerances**, recorded alongside the requirement, so they can be defended and revisited.
- **Enforce a TTL on every entry in the cache client**, so it cannot be omitted by accident.
- **Prefer CDC-driven invalidation over application-level** wherever multiple systems write the data, since coverage is the failure that is invisible.
- **Default hot-read paths to stale-while-revalidate**, which removes miss latency and stampedes for a small, bounded increase in staleness.

**Go deeper**

There are two ways to bound how stale a cached value can be: let a timer expire it, or actively invalidate it on change. The crucial asymmetry is that TTL requires nothing to arrive, while invalidation is a network message that can be lost, delayed, or never sent because a migration script or another service wrote the data through an uninstrumented path. So TTL is the guarantee and invalidation is the optimisation.

Derive the TTL from the staleness requirement rather than from a target hit rate — “a price change must be live within thirty seconds” gives a defensible number, and the cost is usually lower than expected since a hot key keeps a 99.98% hit rate even at a short TTL. Always jitter by ±10–20%, or keys populated together will expire together and produce a stampede. Keep negative TTLs to seconds, or a newly created entity stays invisible.

For coverage, change-data-capture invalidation sees every committed write regardless of which system made it, which application-level invalidation structurally cannot. For hot reads, stale-while-revalidate serves the expired value while refreshing in the background, removing both miss latency and stampedes. And the review question that catches broken designs is: if invalidation stopped working entirely, would we still meet the requirement?

Staleness bounding has exactly two mechanisms, and conflating their roles is the most common structural error in caching design.

**TTL is unconditional.** An entry with a deadline expires whether or not any message arrives, any service is healthy, or any code path was instrumented. That makes it the only mechanism that can carry a guarantee. Invalidation, by contrast, is a distributed message: it can be lost, delayed, reordered, or simply never emitted because a bulk import, a migration, an admin tool or another service wrote the underlying data. Therefore the correct architecture is TTL as the guarantee and invalidation as the optimisation that reduces typical staleness from the TTL to milliseconds.

**Deriving the TTL.** Start from the product or contractual requirement — how out of date may this value be at the moment someone acts on it — rather than from a target hit rate, which produces a number nobody can defend. Then check the cost, which is asymmetric: a key read two hundred times per second retains a 99.98% hit rate at a twenty-five second TTL, while a long-tail key read once per window gets almost no benefit at all. Short TTLs are cheap for hot data and expensive for cold data, which is the opposite of most people's intuition. Always apply ±10–20% jitter, because keys populated together — after a deploy, a warm, or a bulk import — otherwise expire simultaneously and produce a periodic stampede that looks inexplicable.

**Coverage is the invisible failure.** Application-level invalidation covers only the write paths someone instrumented, and the gaps grow silently as more writers appear. Change-data-capture invalidation, published from the database's replication log, sees every committed change regardless of origin, which is the only mechanism with complete coverage. It costs a pipeline to operate and adds a few hundred milliseconds, so TTL remains the backstop — but it converts “we think all writers invalidate” from an assumption into a property.

**Refinements worth defaulting to.** Stale-while-revalidate serves an expired value immediately while refreshing in the background, which removes miss latency from the critical path and eliminates stampedes in one move, at the cost of the TTL plus refresh time as the staleness bound. Negative caching prevents repeated lookups for missing keys but needs a short TTL, or a newly created entity remains invisible. And versioned keys — embedding a version from the row in the cache key — eliminate the invalidation race entirely by making stale entries unreachable, which is invalidation with no delivery problem, at the cost of one extra lookup per read.

**The review question.** If invalidation stopped working completely tomorrow, would the system still meet its staleness requirement? If the answer is no, the requirement is being met by an optimisation, and a degraded message bus becomes a contractual or safety failure rather than a performance one. The corollary that teams most often miss is in-process local caches: they have their own TTLs, usually no invalidation at all, and they silently extend the worst-case staleness of the entire chain — so the local TTL, not the shared cache's, is frequently the number that actually determines whether the guarantee holds.

**Prove it — interview questions**

1. **[Basic] Why is TTL necessary even with invalidation?**

   <details><summary>Model answer</summary>

   Because invalidation is a message and messages can be lost, delayed, or never sent — a migration script or another service writing the same data produces no invalidation at all. TTL requires nothing to arrive: the entry expires on its own. So TTL is the guarantee that bounds worst-case staleness, and invalidation is the optimisation that reduces typical staleness from the TTL to milliseconds. A design with no TTL is one lost message away from permanent inconsistency.

   </details>

2. **[Basic] How should you choose a TTL?**

   <details><summary>Model answer</summary>

   From the staleness requirement, not from a target hit rate. Ask how out of date this value may be when someone acts on it — a contractual thirty seconds for a price, a day for a description — and set the TTL at or below that. Then verify the cost, which is usually lower than people expect for hot keys: a key read two hundred times a second keeps a 99.98% hit rate even at a twenty-five second TTL. Short TTLs are expensive for long-tail keys, not for hot ones.

   </details>

3. **[Senior] What is stale-while-revalidate and when would you use it?**

   <details><summary>Model answer</summary>

   When an entry expires, you serve the stale value immediately and trigger a background refresh, so no request waits for the origin. It removes miss latency from the critical path and eliminates stampedes, because only the background refresh touches the source. The cost is that the served value can be slightly older than the strict TTL — the TTL plus the refresh time. For any data where a second of additional staleness is harmless, it strictly dominates plain expiry, which is why it is standardised in HTTP caching.

   </details>

4. **[Senior] Why is application-level invalidation insufficient in many systems?**

   <details><summary>Model answer</summary>

   Because it only covers the write paths someone instrumented. A bulk import, a database migration, an admin tool, or a second service writing the same table produces no invalidation, so the cache is stale and nothing detects it — the failure is silent and often discovered months later. Change-data-capture invalidation, driven from the database's replication log, sees every committed write regardless of origin, which is the only mechanism with complete coverage. It costs a pipeline to operate and adds a few hundred milliseconds of propagation, which is why TTL still sits underneath it as the final guarantee.

   </details>

5. **[Staff] A requirement says price changes must be live within 30 seconds. How do you design for it?**

   <details><summary>Model answer</summary>

   The TTL must be at or below thirty seconds, with jitter, because that is the only mechanism that holds without a message arriving. I would then add invalidation — ideally driven from the change log so it covers every writer — to bring typical propagation under a second, but explicitly as an optimisation rather than as the thing meeting the requirement. The test I would apply in review is: if the message bus were completely down tomorrow, would we still meet thirty seconds? If the answer depends on invalidation working, then a degraded pub/sub tier turns a performance problem into a contractual breach, and that is the wrong coupling. The cost of the shorter TTL is negligible here, because prices are read hot enough that hit rate barely moves.

   </details>

6. **[Principal] How do you manage staleness across multiple cache tiers and many teams?**

   <details><summary>Model answer</summary>

   By making the guarantee explicit and the mechanisms uniform. Each cached dataset should carry a documented staleness tolerance owned by whoever set the requirement, and the TTL derived from it, so the number is defensible and revisitable rather than folklore. The cache client library should enforce that every entry has a TTL and should apply jitter automatically, because these are the two failures every team otherwise rediscovers. For invalidation, I would standardise on change-log-driven publication rather than per-service calls, since coverage gaps are invisible and grow as more writers appear. And I would pay particular attention to in-process local caches, which are the tier people forget: they have their own TTLs, usually no invalidation, and they silently extend the worst-case staleness of the whole chain — so the local TTL, not the shared cache's, is often the number that actually determines whether the requirement is met.

   </details>

---

### Cache stampede protection

*When a hot key expires, every concurrent request misses at once. Coalesce them so one request refills and the rest wait — or serve stale and refresh behind.*

**Flow:** `Concurrent misses` → `Per-key coalescing` → `One refill` → `Shared result`

> **The 30-second version**  
> A hot key's expiry sends every concurrent request to the origin at once. Coalesce so one refill serves all, or serve the stale value while refreshing behind it.

**The problem**

A key serving 30,000 requests per second expires. In the next 50 milliseconds, 1,500 requests all find an empty cache and all issue the same expensive query. The database, provisioned for a few hundred queries per second, receives 1,500 identical ones simultaneously.

It gets worse from there. The database slows under the load, so each of those queries takes longer, so more requests arrive and miss, so more queries are issued. The refill that was supposed to take 20 milliseconds now takes seconds, during which the stampede grows.

> **Three names, one shape**  
> **Thundering herd**: many waiters wake and act simultaneously. **Cache stampede**: many concurrent misses on one key. **Dogpile**: the same, emphasising the self-amplification. All describe a single expiry event turning into a load multiplier, and all are solved by the same idea: make one request do the work.

**Mental model**

Between the moment a key expires and the moment it is repopulated, every request is a miss. The number of wasted duplicate queries is `request_rate × refill_duration`. Both terms are attackable: coalesce so only one query happens, or eliminate the window so no request ever finds it empty.

1. **Single-flight** — The first miss starts the fetch; concurrent misses for the same key wait on that in-flight result. Reduces N queries to one.
2. **Stale-while-revalidate** — Never let a request see an empty cache. Serve the expired value and refresh in the background.
3. **Probabilistic early expiry** — Each reader independently decides, with increasing probability as expiry approaches, to refresh early. Spreads refills over time with no coordination.
4. **Jittered TTL** — Prevents keys populated together from expiring together, which turns one stampede into thousands.
5. **Locking with a fallback** — Take a short lock to refill; others serve stale or wait. Effective but introduces a lock to reason about.

> **Single-flight is per-process; the cluster still stampedes**  
> In-process coalescing reduces 1,500 requests to one *per instance*. With 100 instances, the origin still sees 100 simultaneous queries rather than one. For genuinely hot keys you need either a shared lock in the cache, a request-coalescing proxy, or stale-while-revalidate — which sidesteps the problem entirely because no instance ever needs to block.

**How it works**

**Single-flight, and what it actually saves**

```text
WITHOUT COALESCING
  30,000 rps on key K, refill takes 40ms
  duplicate queries per expiry = 30,000 x 0.040 = 1,200

WITH IN-PROCESS SINGLE-FLIGHT (100 instances)
  each instance issues 1 query
  duplicate queries = 100
  -> 12x better, still 100x more than necessary

WITH STALE-WHILE-REVALIDATE
  expired value served immediately by all instances
  one background refresh per instance, but nobody WAITS
  -> origin sees 100 queries spread over the refresh window,
     and zero user-visible latency

WITH A SHARED LOCK IN THE CACHE
  SET lock:K token NX EX 10
  winner refills; losers serve stale or wait briefly
  -> origin sees exactly 1 query
  -> cost: a lock, a token, and a fallback if the winner dies
```

1. **Implement single-flight in the cache client, not per service** — It is a library concern. Every team otherwise rediscovers it after an incident, and implementations differ in subtle ways around timeouts and error propagation.
2. **Prefer stale-while-revalidate for read-hot keys** — It removes the miss from the critical path entirely, which means the stampede has no user-visible cost even if it partly occurs.
3. **Use probabilistic early refresh for smooth behaviour** — Each reader refreshes early with probability rising as expiry approaches, so refills spread naturally with no coordination and no lock.
4. **Jitter every TTL** — Without it, keys warmed together expire together and you get a fleet-wide synchronised stampede rather than one key's.
5. **Bound the fall-through concurrency** — Whatever coalescing you use, put a concurrency limit in front of the origin so a coalescing failure cannot present unlimited load.
6. **Handle the failed-refill case** — If the refill fails, do not let every waiter then attempt its own. Cache the failure briefly, or serve stale for longer, with an explicit maximum.

**Probabilistic early expiry (XFetch)**

```text
Store with each value: the time it took to compute (delta)
and its expiry time.

on read:
  if now - delta * beta * ln(random()) >= expiry:
      refresh now          <- probabilistically early
  else:
      serve cached value

BEHAVIOUR
  far from expiry: essentially never refreshes early
  near expiry:     increasing chance each reader refreshes
  expensive values (large delta): refresh earlier

WHY IT IS ELEGANT
  no locks, no coordination, no waiting.
  refills are spread over a window before expiry,
  so the cache is never empty and the origin never spikes.
```

> **The refill failure loop is the second-order stampede**  
> Coalescing works until the refill fails. If the single flight returns an error and every waiter then independently retries, you get the stampede you were preventing, now against a failing dependency. Propagate the error to all waiters, cache the failure for a short interval, and apply a circuit breaker — otherwise the protection inverts under exactly the conditions it exists for.

**Worked example**

A homepage feed query costing 400 ms, served 50,000 times per second, cached for 60 seconds.

**Progressive mitigation, measured**

```text
BASELINE (plain TTL, no protection)
  misses per expiry = 50,000 x 0.400 = 20,000 duplicate queries
  each expiry event: a 20,000-query burst at a database
  sized for ~300 qps
  -> the refill takes longer under load -> burst grows
  -> the cache effectively never repopulates cleanly

+ JITTERED TTL
  stops many keys expiring together, but this single hot key
  still stampedes -> no help for the hot-key case

+ IN-PROCESS SINGLE-FLIGHT (200 instances)
  200 duplicate queries per expiry
  -> survivable, but a 200x burst every 60s is still visible
     as a latency spike

+ STALE-WHILE-REVALIDATE
  expired value served instantly; refresh in background
  user-visible latency: unchanged, always ~1ms
  origin: 200 refreshes spread over the refresh window
  -> no spike, no waiting

+ SHARED LOCK (cache-level)
  exactly 1 refresh per expiry across the fleet
  -> origin sees 1 query per 60s for this key
  cost: lock ownership, expiry, and a fallback if the
        holder dies mid-refresh
```

| Metric | Value | Note |
|---|---|---|
| No protection | 20,000 queries | per expiry |
| Single-flight | 200 | per-instance |
| Stale-while-revalidate | 200 spread | **0 user latency** |
| Shared lock | 1 | most complex |

> **Optimise for user-visible latency first, origin load second**  
> Stale-while-revalidate leaves 200 background refreshes but removes every millisecond of user-visible impact. A shared lock reduces origin load to one query but leaves users waiting behind the lock if the refresh is slow. For a read path, the first is usually the better trade — and the two compose, if the origin genuinely cannot absorb 200 queries.

**When to use it**

- **Any hot key with an expensive refill**, where request rate times refill duration is more than a handful.
- **Fan-out and aggregation queries**, whose refill cost is high and whose results are widely shared.
- **After a cache restart or deploy**, when the entire keyspace is cold simultaneously.
- **Anywhere TTLs are short**, since frequent expiry means frequent stampede opportunities.
- **Behind CDNs and edge caches**, where origin shields coalesce misses across many edge locations.

**When to avoid it**

- **Do not rely on in-process single-flight alone** for a key hot enough to matter across a large fleet.
- **Do not add a distributed lock without a fallback** — if the holder dies, everyone waits for the lock TTL.
- **Do not let every waiter retry independently** when the refill fails.
- **Do not use stale-while-revalidate where staleness is unacceptable**, such as authorisation checks.
- **Do not treat jitter as stampede protection for a single hot key** — it only helps with correlated expiry across many keys.

**Advantages**

- **Enormous reduction in origin load** for a small amount of library code.
- **Stale-while-revalidate removes miss latency entirely**, which is often worth more than the load reduction.
- **Probabilistic refresh needs no coordination**, making it trivially scalable across a fleet.
- **Protects against the self-amplifying case**, where a slow refill widens the miss window and grows the herd.
- **Composable**: jitter, coalescing and stale-serving stack, each addressing a different part of the problem.

**Disadvantages**

- **In-process coalescing does not scale across instances**, leaving a fleet-sized burst.
- **Distributed locks add a failure mode** — a dead lock holder blocks everyone until expiry.
- **Stale-while-revalidate widens the staleness window** beyond the nominal TTL.
- **Refill failure handling is subtle**, and getting it wrong inverts the protection.
- **Extra bookkeeping** — in-flight maps, compute-time tracking, lock tokens — that must be correct under concurrency.

**Trade-offs**

**Stampede mitigations compared**

| Technique | Origin queries per expiry | User latency | Complexity |
|---|---|---|---|
| None | rate × refill time | Full refill on miss | None |
| Jittered TTL | Unchanged for one key | Unchanged | Trivial |
| In-process single-flight | One per instance | Waits for refill | Low |
| Distributed lock | One | Waits, or serves stale | Moderate; failure modes |
| Stale-while-revalidate | One per instance, spread | **Zero** | Low |
| Probabilistic early refresh | Spread, never zero-cache | Zero | Low; needs compute-time tracking |

The bottom two rows are usually the right answer, and they combine well: probabilistic early refresh keeps the cache from ever being empty, and stale-while-revalidate covers the case where it is anyway.

**How it fails**

**Stampede-related failures**

| Symptom | Cause | Fix |
|---|---|---|
| Periodic latency spikes at TTL boundaries | Synchronised expiry of keys warmed together | Jitter TTLs by ±10–20% |
| Database overload on one popular item | Hot-key stampede with no coalescing | Single-flight; stale-while-revalidate; shared lock |
| Cache never repopulates cleanly | Refill slowed by the stampede it caused | Coalesce; bound fall-through concurrency |
| Everyone blocked for seconds | Distributed lock holder died mid-refresh | Short lock TTL; fallback to stale; fencing token |
| Stampede against a failing dependency | Waiters retried independently after a failed refill | Propagate the error to all waiters; cache failures briefly; circuit break |
| Cold start after deploy overwhelms origin | Entire keyspace empty simultaneously | Warm before shifting traffic; concurrency limit; gradual ramp |
| Coalescing helps in one service, not overall | In-process only, many instances | Shared lock or edge-level coalescing |

**Limits**

> **Sizing the problem**
>
> - **Duplicate queries per expiry ≈ request rate × refill duration.** 30,000 rps × 40 ms = 1,200.
> - **In-process single-flight** divides that by instance count, not to one.
> - **Hedge on refill**: keep lock TTLs short (seconds) so a dead holder blocks briefly.
> - **Jitter ±10–20%** breaks correlated expiry; it does nothing for a single hot key.
> - **Fall-through concurrency limit** should equal the origin's real capacity, since it is the last line of defence.

**Alternatives**

| Approach | Removes | Cost |
|---|---|---|
| Single-flight coalescing | Duplicate concurrent work | Waiters block on one fetch |
| Stale-while-revalidate | Miss latency and waiting | Wider staleness window |
| Probabilistic early refresh | The empty-cache window | Compute-time tracking |
| Refresh-ahead (scheduled) | Expiry entirely for known hot keys | Refreshing keys nobody requests |
| Never expire; invalidate only | Expiry stampedes | Unbounded staleness if invalidation is lost |
| Origin concurrency limit | Unbounded origin load | Some requests fail fast |

Scheduled refresh-ahead deserves mention for a small set of extremely hot keys: a background job refreshing them on a timer means they never expire from the readers' perspective, and the origin sees a steady trickle rather than bursts.

**In real systems**

- **Go's `singleflight` package** is the canonical in-process implementation and is widely embedded in cache clients.
- **Facebook's memcache leases** grant one client the right to refill a key while others wait or serve stale — a shared-lock approach at scale.
- **HTTP's `stale-while-revalidate` directive** standardises serving an expired response while refreshing behind it.
- **CDN origin shields** exist specifically to coalesce misses from many edge locations into a single origin request.
- **The XFetch paper** formalised probabilistic early expiry, which several cache libraries now implement directly.

**Common mistakes**

- **Relying on in-process single-flight** for a key hot across hundreds of instances.
- **Uniform TTLs**, producing synchronised expiry across the keyspace.
- **No fallback when a distributed lock holder dies**, blocking everyone until the lock expires.
- **Independent retries after a failed refill**, stampeding a failing dependency.
- **No concurrency limit on the origin**, so any gap in protection is unbounded.
- **Treating cold start as a separate problem** rather than the same one at fleet scale.
- **Adding stale-while-revalidate to a path where staleness is unsafe**, such as authorisation.

**The staff-level view**

Stampede protection is a library-level concern that every team otherwise learns from an outage, which makes it a high-leverage place to intervene once.

- **Ship coalescing and stale-while-revalidate as cache client defaults**, so protection is inherited rather than remembered.
- **Enforce jittered TTLs in the client**, since correlated expiry is invisible until it produces a periodic spike nobody can explain.
- **Put a concurrency limit in front of every origin**, as the backstop that holds when the coalescing logic has a gap.
- **Design the refill-failure path explicitly.** Protection that inverts under dependency failure is worse than none, because it arrives precisely when the system is already degraded.
- **Treat cold start as the same problem.** A deploy or cache restart is a fleet-wide stampede, and it needs warming plus a traffic ramp rather than optimism.

**Go deeper**

When a hot key expires, every request arriving before it is repopulated misses, so duplicate origin queries equal request rate times refill duration — thirty thousand requests per second with a forty-millisecond refill produces twelve hundred identical queries at once. It is self-amplifying: the load slows the refill, which widens the window and grows the herd.

Single-flight coalescing makes the first miss fetch while concurrent misses wait on its result, reducing N queries to one *per process* — with two hundred instances the origin still sees two hundred. Stale-while-revalidate is usually the better default because it attacks user-visible latency: the expired value is served immediately and refreshed behind, so nobody waits and the cache is never empty. Probabilistic early refresh spreads refills before expiry with no coordination at all.

Two details decide whether protection holds. Jitter every TTL, or keys warmed together expire together and turn one stampede into thousands. And design the refill-failure path: if a failed single flight lets every waiter retry independently, the protection inverts exactly when the dependency is already failing. Underneath everything, a concurrency limit on the origin is the backstop that makes any remaining gap survivable.

A cache stampede is a single expiry event turning into a load multiplier, and it is one of the few caching failures that is actively self-amplifying — which is why it produces outages rather than merely slow responses.

**The arithmetic.** Between expiry and repopulation, every arriving request is a miss, so duplicate origin queries are approximately request rate multiplied by refill duration. Both terms are attackable, and the second is worse than it looks: the burst slows the origin, extending the refill, which widens the miss window and increases the burst further. Past a threshold the cache never repopulates cleanly at all, and the system sits in permanent stampede.

**Coalescing and its limit.** Single-flight — the first miss fetches while concurrent misses for the same key wait on that result — is the textbook answer and reduces the burst to one query per process. The limitation that surprises people is that it is per-process: with a large fleet, the origin still receives one query per instance, which for a genuinely hot key is still a hundredfold burst. Closing that gap requires either a shared lock in the cache, a coalescing proxy or origin shield, or a technique that removes the empty-cache window entirely.

**The better defaults.** Stale-while-revalidate serves the expired value immediately and refreshes in the background, so no request ever waits and the cache is never empty — it attacks user-visible latency rather than merely origin load, which for a read path is usually the more valuable target. Probabilistic early expiry has each reader independently decide to refresh early with a probability that rises as expiry approaches and scales with how expensive the value was to compute, spreading refills over a window with no locks and no coordination. Together these mean the empty window largely does not occur, and when it does, nobody waits in it.

**Jitter, and why it is not enough.** Keys populated together — after a deploy, a warm, or a bulk import — expire together without jitter, producing periodic fleet-wide spikes that look inexplicable on a dashboard. Jitter of ten to twenty per cent breaks that correlation cheaply. But it does nothing for a single hot key, which is a different problem: correlated expiry across many keys versus concurrent misses on one. Conflating them leads teams to add jitter, see no improvement, and conclude stampede protection does not work.

**The failure path decides everything.** Protection that works when healthy and disappears under dependency failure is worse than none, because it evaporates precisely when the system is already degraded. If a coalesced refill fails and every waiter independently retries, the stampede is recreated against a failing origin. The correct design propagates the single attempt's error to all waiters, caches the failure briefly with a short negative TTL, and sits behind a circuit breaker. Similarly, a distributed lock needs a short TTL and a stale-serving fallback, or a holder that dies mid-refresh blocks the entire fleet until the lock expires.

**Why it belongs in the platform.** Each of these details is individually easy to omit and individually invisible until an incident, so independent per-service implementations diverge exactly where it matters. Shipping coalescing, stale-while-revalidate, jitter and an origin concurrency limit as cache client defaults means protection is inherited rather than remembered. The same argument covers cold start, which is this problem at fleet scale — a deploy or cache restart empties the whole keyspace at once — and which needs warming and a traffic ramp that no individual team will build for itself.

**Prove it — interview questions**

1. **[Basic] What is a cache stampede?**

   <details><summary>Model answer</summary>

   When a cached key expires and many concurrent requests all miss simultaneously, each independently issuing the same expensive refill. The number of duplicate queries is roughly the request rate times the refill duration, so a key served thirty thousand times a second with a forty-millisecond refill produces around twelve hundred identical queries at once. It is self-amplifying, because the resulting load slows the refill, which widens the miss window and increases the herd.

   </details>

2. **[Basic] What is single-flight?**

   <details><summary>Model answer</summary>

   Per-key coalescing: the first request that misses starts the fetch, and any concurrent request for the same key waits on that in-flight result rather than starting its own. It reduces N duplicate queries to one within a process. The important limitation is that it is per-process, so with two hundred service instances the origin still sees two hundred simultaneous queries — better, but not the single query people assume.

   </details>

3. **[Senior] Why is stale-while-revalidate often better than coalescing?**

   <details><summary>Model answer</summary>

   Because it attacks user-visible latency rather than just origin load. With coalescing, the waiters still wait for the refill — a four-hundred-millisecond query means every concurrent request takes four hundred milliseconds. With stale-while-revalidate, the expired value is served immediately and the refresh happens behind it, so no request ever waits and the cache is never empty. The origin still sees some refresh traffic, but it is spread rather than bursty. The cost is that the served value may be older than the nominal TTL, which is unacceptable only where staleness has consequences.

   </details>

4. **[Senior] What happens if the refill fails while requests are coalesced?**

   <details><summary>Model answer</summary>

   This is the failure that inverts the protection. If the single flight returns an error and every waiter then independently retries, you produce exactly the stampede you were preventing, now directed at a dependency that is already failing. The correct handling is to propagate the error to all waiters from the single attempt, cache the failure briefly with a short negative TTL so the next wave does not immediately retry, and put a circuit breaker in the path so sustained failure stops generating load at all. Getting this wrong means the protection works in healthy conditions and disappears in exactly the conditions it exists for.

   </details>

5. **[Staff] Design stampede protection for a homepage query costing 400 ms at 50,000 requests per second.**

   <details><summary>Model answer</summary>

   Unprotected, each expiry produces about twenty thousand duplicate queries against a database sized for hundreds — which is not a spike but a failure, because the refill slows under the load and the window widens. I would layer the mitigations. Jittered TTLs first, which costs nothing and prevents correlated expiry across the keyspace, though it does not help this single hot key. In-process single-flight next, bringing it to one query per instance. Then stale-while-revalidate, which is the important one: the expired value is served instantly so no user ever waits, and refreshes spread out behind. If the origin still cannot absorb a few hundred refreshes, I would add a cache-level lock so exactly one refresh happens per expiry, with a short lock TTL and a stale-serving fallback so a dead lock holder cannot block the fleet. Underneath all of it, a concurrency limit on the origin client as the backstop, because every one of these mechanisms has a gap and the limit is what makes the gap survivable.

   </details>

6. **[Principal] Why should stampede protection live in shared infrastructure rather than in each service?**

   <details><summary>Model answer</summary>

   Because it is subtle enough that independent implementations diverge in the places that matter, and because the failure only appears under production concurrency, so every team learns it from an outage rather than from review. The specific subtleties — propagating errors from a single flight rather than letting waiters retry, keeping lock TTLs short with a stale fallback, jittering every TTL, applying a concurrency limit as the last line — are each easy to omit and individually invisible until the incident. Shipping them as defaults in the cache client means protection is inherited rather than remembered, and it means an improvement to the logic benefits every service at once. The same argument applies to cold start, which is the identical problem at fleet scale: a deploy or cache restart empties the entire keyspace simultaneously, and the platform is the right place to own warming and traffic ramping, because no individual team will build it for themselves.

   </details>

---

### Cache eviction policies

*When the cache is full, something must go. Which one you evict — and more importantly, what you refuse to admit — decides your hit rate.*

**Flow:** `Access trace` → `Admission` → `Working set` → `Eviction policy` → `Miss cost`

> **The 30-second version**  
> When the cache is full, the policy decides what leaves — but the bigger lever is admission: refusing to cache one-off items so a single scan cannot flush your working set.

**The problem**

A cache has finite memory and the dataset does not. Every insertion beyond capacity displaces something. Choosing badly means evicting an item that was about to be requested while keeping one that never will be — and the difference between a good and bad policy on the same workload can be tens of percentage points of hit rate.

The subtler problem is that the classic policies have pathological cases that appear in ordinary workloads: a single large scan can flush an entire LRU cache, and a frequency-based policy can hold onto items that were popular last week and are now irrelevant.

> **Admission matters more than eviction**  
> The question “which item should I remove?” presupposes the new item deserves a place. Often it does not — a one-off scan, a crawler, a batch job. **Admission policies** ask whether the candidate is likely to be reused before deciding to displace something. Modern high-performing caches spend most of their intelligence on admission, not eviction.

**Mental model**

A cache is a bet that recent or frequent access predicts future access. Each policy encodes a different bet, and the right one depends on whether your workload's popularity is stable or shifting.

1. **LRU — least recently used** — Bets on recency. Excellent for workloads with temporal locality; destroyed by a single large scan.
2. **LFU — least frequently used** — Bets on frequency. Robust to scans; slow to adapt when popularity shifts, and needs aging to avoid cache pollution.
3. **FIFO / CLOCK** — Approximates LRU cheaply with no per-access bookkeeping. Slightly worse hit rate, much lower overhead.
4. **TTL-driven** — Eviction by expiry rather than by pressure — correctness first, capacity second.
5. **Admission (TinyLFU)** — Before inserting, estimate whether the candidate is more valuable than the victim. Rejects one-off items entirely.

> **The scan that empties your cache**  
> An analytics query, a crawler, a backup job or a paginated export reads a million items once. Under LRU, each read displaces something useful, and by the end the cache contains only data nobody will request again. Hit rate collapses for everyone, and it recovers only as the working set is slowly re-read. This single failure is why admission control exists.

**How it works**

**How the policies behave on the same trace**

```text
WORKLOAD  hot set of 100 items read constantly,
          plus a scan of 10,000 items read once

LRU
  the scan displaces all 100 hot items
  hit rate after the scan: ~0% until the hot set is re-read
  classic pathology

LFU (with aging)
  scan items have frequency 1; hot items have frequency 1000s
  scan items are evicted immediately
  hit rate: unaffected
  but: if the hot set CHANGES, old winners linger

W-TinyLFU (admission + LRU window)
  small LRU window admits new items briefly
  a frequency sketch decides whether a candidate
  deserves to displace the eviction victim
  scan items fail admission -> never enter the main cache
  hit rate: unaffected by the scan, AND adapts to shifts
  -> this is why modern caches use it
```

1. **Measure your access distribution before choosing** — Zipfian workloads with a stable hot set favour frequency and admission; workloads with strong recency and shifting popularity favour LRU or adaptive policies.
2. **Protect against scans explicitly** — Either use an admission policy, or route scan-like workloads through a separate cache or a bypass flag so they never pollute the shared one.
3. **Age frequency counters** — Pure LFU never forgets, so yesterday's popular items squat forever. Periodic halving of counters — as TinyLFU does — keeps it responsive.
4. **Size by working set, not by dataset** — With Zipfian access, caching the hot 10–30% typically captures 90–97% of requests. Sizing beyond that buys very little.
5. **Watch the eviction rate, not just the hit rate** — A high eviction rate with a decent hit rate means the cache is thrashing at the boundary and a modest size increase may pay off disproportionately.
6. **Distinguish eviction from expiry** — TTL expiry is about correctness; eviction is about capacity. A cache evicting before TTL is undersized, and that is a different problem from a staleness problem.

**Sizing from the access distribution**

```text
ZIPFIAN ACCESS (typical for user-facing systems)
  item rank r gets traffic proportional to 1/r^alpha

  cache size (% of dataset)   expected hit rate
  1%                          ~60-70%
  5%                          ~80-85%
  10%                         ~88-92%
  30%                         ~95-97%
  60%                         ~98%
  100%                        100%

-> sharply diminishing returns. Doubling the cache from
   30% to 60% of the dataset buys ~1-2 points of hit rate
   and doubles cost.

THE CORRECT QUESTION
  origin load = rate x (1 - hit rate)
  going 92% -> 97% cuts origin load by 2.7x
  going 97% -> 98% cuts it by 1.5x
  -> value the change in MISS rate, not in hit rate
```

> **Always reason about miss rate, not hit rate**  
> “We improved hit rate from 97% to 98%” sounds like a one-percent improvement. It halved the miss rate and therefore halved origin load. Conversely, a drop from 99% to 97% triples origin traffic. Hit rate is a compressed scale that hides exactly the changes that matter operationally.

**Worked example**

A 64 GB cache in front of a 2 TB product catalogue, degraded by nightly batch jobs.

**Diagnosing and fixing a pollution problem**

```text
SYMPTOM
  hit rate 94% during the day, 61% between 02:00 and 04:00,
  and takes until ~07:00 to recover

CAUSE
  a nightly export reads every product once.
  under LRU, 2 TB of one-time reads displace the entire
  64 GB working set.

OPTIONS
  1  BYPASS
     export path sets a no-cache flag; reads go direct
     effect: hit rate flat at 94% all night
     cost: export is slower (it was benefiting slightly)

  2  SEPARATE CACHE
     export uses its own small cache instance
     effect: same, with some benefit for the export
     cost: another cache to run

  3  ADMISSION POLICY (W-TinyLFU)
     candidates must beat the eviction victim on estimated
     frequency; single-read items fail admission
     effect: hit rate flat, no application change
     cost: requires a cache supporting it

  4  BIGGER CACHE
     effect: none. 2 TB of scan will flush any size
             that is not the whole dataset.

CHOSEN: 1 for immediate relief, 3 as the durable fix
```

| Metric | Value | Note |
|---|---|---|
| Day hit rate | 94% | healthy |
| Night hit rate | 61% | **6.5× origin load** |
| Recovery | 3 hours | slow re-warm |
| Bigger cache | no help | scans defeat size |

> **You cannot buy your way out of a pollution problem**  
> The instinct on seeing a hit-rate collapse is to add memory. Against a scan, that is money spent for nothing: any cache smaller than the scanned dataset will be flushed. The fix is categorical — keep the scan out, either by bypassing, isolating, or refusing admission — and recognising that distinction saves both the budget and the incident.

**When to use it**

- **Whenever the cache is smaller than the dataset**, which is essentially always.
- **When choosing or configuring a cache**, since policy is one of the few knobs that materially changes hit rate.
- **When hit rate is unstable over time**, which usually indicates pollution or a shifting working set.
- **When sizing**, to know where the diminishing returns begin for your access distribution.
- **When mixed workloads share a cache**, where one tenant's scans can destroy another's hit rate.

**When to avoid it**

- **Do not use plain LRU where scans occur**, unless you can route them around the cache.
- **Do not use pure LFU without aging**, or stale winners will squat indefinitely.
- **Do not size a cache from the dataset size**; size it from the working set and the access distribution.
- **Do not share a cache between interactive and batch workloads** without isolation.
- **Do not tune eviction when the real problem is TTL** — eviction before expiry means undersized, which is a different fix.

**Advantages**

- **A good policy can be worth tens of points of hit rate** on the same memory, which is free capacity.
- **Admission policies eliminate the scan pathology** without any application changes.
- **CLOCK-style approximations give near-LRU behaviour** at a fraction of the bookkeeping cost.
- **Frequency-based policies resist pollution** from one-off traffic naturally.
- **Understanding the distribution turns sizing from guesswork into arithmetic.**

**Disadvantages**

- **Every policy has a pathological workload**, and yours may hit it.
- **LRU is scan-vulnerable**; LFU is slow to adapt; admission policies add complexity and memory for the frequency sketch.
- **Policy choice is often not configurable** in managed caches, limiting your options to isolation and bypass.
- **Hit rate hides the operationally relevant change**, since miss rate is what drives origin load.
- **Multi-tenant caches make one workload's behaviour another's problem**, and policies alone rarely fix that.

**Trade-offs**

**Policy characteristics**

| Policy | Strength | Pathology | Overhead |
|---|---|---|---|
| LRU | Strong temporal locality | A single scan flushes it | Per-access list update |
| LFU | Resists scans and one-off traffic | Slow to adapt; stale winners without aging | Counters per item |
| FIFO / CLOCK | Near-LRU quality, minimal cost | Slightly worse hit rate | Very low |
| ARC | Balances recency and frequency adaptively | Patent history limited adoption | Moderate |
| W-TinyLFU | Scan-resistant and adaptive | More complex; sketch memory | Low (approximate counters) |
| Random | Surprisingly acceptable; trivial | No locality exploitation | None |

Random eviction is worth knowing about: at high hit rates it performs far closer to LRU than intuition suggests, and it has no bookkeeping and no pathological case. Several production systems use it deliberately for exactly that predictability.

**How it fails**

**Eviction-related failures**

| Symptom | Cause | Fix |
|---|---|---|
| Hit rate collapses during batch windows | Scan pollution under LRU | Bypass, isolate, or use an admission policy |
| Cache thrashing at the size boundary | Working set slightly exceeds capacity | Modest size increase; the returns here are steep |
| Stale items never evicted | LFU without aging | Age or halve counters periodically |
| One tenant destroys another's hit rate | Shared cache, no isolation | Per-tenant quotas or separate instances |
| Eviction before TTL expiry | Cache undersized for the working set | Increase size, or reduce what is cached |
| Memory pressure causes unpredictable evictions | No explicit policy or limit configured | Set a max memory and an explicit policy |
| Hit rate fine, origin still overloaded | Small hit-rate change, large miss-rate change | Reason in miss rate; recompute origin load |

**Limits**

> **Sizing arithmetic**
>
> - **Zipfian rule of thumb**: hot 10% of items ≈ 90% hit rate; hot 30% ≈ 95–97%.
> - **Origin load = rate × (1 − hit rate)**; always evaluate changes in miss rate.
> - **A scan larger than the cache will flush any cache**, regardless of size — size is not a remedy.
> - **CLOCK approximations** typically land within 1–3 points of true LRU at far lower cost.
> - **Frequency sketches** (as in TinyLFU) cost a few bytes per tracked item, far less than storing the items themselves.

**Alternatives**

| Approach | Addresses | Cost |
|---|---|---|
| Better eviction policy | Which item to remove | Complexity; may not be configurable |
| Admission policy | Whether to admit at all | Sketch memory; complexity |
| Cache partitioning by workload | Cross-workload pollution | Lower utilisation per partition |
| Bypass flag for scans | Pollution from known batch traffic | Application awareness |
| Bigger cache | Working set slightly too large | Cost; no help against scans |
| Tiered caching | Different locality at different layers | Coherence between tiers |

**In real systems**

- **Caffeine (Java)** popularised W-TinyLFU and demonstrates hit rates well above LRU on standard traces, which is why it became a default in many stacks.
- **Redis** offers several policies including `allkeys-lru`, `allkeys-lfu` and `volatile-ttl`, making the choice an explicit operational decision.
- **Memcached's segmented LRU** splits hot and cold segments to resist the scan pathology without full admission control.
- **CDNs** implement admission heuristics so that content requested once from one location does not displace globally popular assets.
- **Database buffer pools** use CLOCK or segmented LRU variants precisely because a sequential table scan would otherwise evict the entire working set.

**Common mistakes**

- **LRU in front of a workload that includes scans.**
- **Adding memory to fix pollution**, which cannot work against a scan larger than the cache.
- **Pure LFU with no aging**, letting stale winners squat.
- **Sizing from dataset size** rather than from the access distribution.
- **Sharing one cache between interactive and batch traffic** with no isolation.
- **Judging changes by hit rate** instead of by the miss rate that drives origin load.
- **Leaving the policy unset**, so behaviour under memory pressure is unspecified.

**The staff-level view**

Eviction policy is a rare case where a configuration choice, not a code change, moves a headline metric — and where the instinct to spend money is usually wrong.

- **Reason in miss rate, and insist others do.** “Hit rate improved from 97% to 98%” hides that origin load halved; the reverse hides that it tripled.
- **Treat scan pollution as a categorical problem**, fixed by bypass, isolation or admission — never by buying more memory.
- **Isolate batch and interactive workloads** in shared caches, since one tenant's access pattern silently becomes another's incident.
- **Prefer caches with admission control** for shared infrastructure, because it removes an entire failure class without application cooperation.
- **Check whether eviction is happening before TTL.** If it is, the cache is undersized, which is a capacity conversation rather than a policy one.

**Go deeper**

LRU bets on recency and is excellent for temporal locality, but a single large scan makes every scanned item the most recently used and flushes the entire working set. LFU bets on frequency and resists scans, but without aging it holds onto items that were popular last week. FIFO and CLOCK approximate LRU at far lower bookkeeping cost, usually within a few points of hit rate.

The more powerful idea is admission: before inserting, estimate from a compact frequency sketch whether the candidate deserves to displace the eviction victim. Items read once fail admission and never enter the cache, which eliminates the scan pathology without any application changes. This is why modern caches such as Caffeine use W-TinyLFU rather than plain LRU.

Two numerical habits matter. Size from the access distribution, not the dataset — with Zipfian access, the hot 10% of items gives roughly 90% hit rate and 30% gives 95–97%, with sharply diminishing returns. And evaluate changes in miss rate, because a drop from 99% to 97% triples origin load while sounding like a two-point change. Crucially, pollution is categorical: no amount of extra memory helps against a scan larger than the cache.

Eviction policy is one of the few configuration choices that moves a headline metric without a code change, and one where the intuitive remedy — buy more memory — is frequently the wrong answer.

**Each policy is a bet.** LRU bets that recency predicts reuse, which is true for most interactive workloads and catastrophically false during a scan: a job reading a million items once makes each of them most-recently-used, displacing the working set entirely. LFU bets on frequency, which resists scans naturally, but without periodic aging of counters, items popular last week squat indefinitely and the cache stops adapting. CLOCK and FIFO approximate LRU with almost no per-access bookkeeping, typically landing within one to three points of its hit rate — a good trade at high throughput. Random eviction, counter-intuitively, performs respectably at high hit rates and has no pathological case at all.

**Admission is the larger lever.** Asking which item to evict presupposes the candidate deserves entry. Frequently it does not — crawler traffic, paginated exports, analytics scans, one-off lookups. W-TinyLFU keeps a compact frequency sketch and admits a candidate only if it beats the prospective victim, so single-read items never enter the main cache. This eliminates the entire scan-pollution failure class without any application cooperation, which matters because bypass flags depend on every future batch job remembering to set one, and they will not.

**Sizing is arithmetic, not intuition.** Real access distributions are heavy-headed, so caching the hot ten per cent of a dataset typically yields around ninety per cent hit rate and thirty per cent yields ninety-five to ninety-seven, with very sharp diminishing returns beyond that. Doubling the cache from thirty to sixty per cent of the dataset buys a point or two. The correct evaluation frame is miss rate: origin load is request rate times miss rate, so ninety-two to ninety-seven per cent cuts origin traffic nearly threefold, while ninety-nine to ninety-seven triples it. Hit rate is a compressed scale that conceals precisely the changes that matter operationally, and reporting in it causes real misjudgements.

**Pollution is categorical.** When a scan larger than the cache runs, any cache smaller than that scan will be flushed, so adding memory is money spent for nothing. The remedies are different in kind: bypass the cache for known batch paths, isolate batch traffic in a separate instance, or use admission control so the cache refuses the traffic itself. Recognising that the problem is categorical rather than quantitative is what prevents an expensive and ineffective capacity purchase.

**Shared caches make one tenant's pattern another's incident.** The victim sees a hit-rate collapse with no local explanation, and the cause sees nothing unusual at all. Managing this needs isolation — separate instances or per-tenant quotas, accepting lower utilisation for predictability — plus admission control so sharing is safe by default rather than by agreement, plus per-tenant hit and eviction metrics so attribution is possible. And because policy interactions between tenants cannot be reasoned about locally, the policy for a shared cache should be platform-owned rather than individually tunable.

**Prove it — interview questions**

1. **[Basic] Why does LRU perform badly with scans?**

   <details><summary>Model answer</summary>

   Because LRU assumes recent access predicts future access, and a scan violates that assumption for every item it touches. A job reading a million items once makes each of them the most recently used, displacing the genuinely hot working set. By the end, the cache holds only items nobody will request again, hit rate collapses, and it recovers only as the real working set is slowly re-read. Adding memory does not help, because any cache smaller than the scanned dataset will be flushed.

   </details>

2. **[Basic] What is an admission policy?**

   <details><summary>Model answer</summary>

   A check performed before inserting a new item, asking whether it is likely to be reused more than the item it would displace. Typically it uses a compact frequency sketch to estimate how often the candidate and the eviction victim have been seen. Items read once — scan traffic, crawlers, one-off lookups — fail admission and never enter the cache at all, which eliminates pollution without any cooperation from the application.

   </details>

3. **[Senior] How do you size a cache?**

   <details><summary>Model answer</summary>

   From the access distribution rather than the dataset size. For typical Zipfian workloads, caching the hot ten per cent of items yields around ninety per cent hit rate and thirty per cent yields ninety-five to ninety-seven, with sharply diminishing returns beyond that. The right way to evaluate a size change is in miss rate, because that is what drives origin load: moving from ninety-two to ninety-seven per cent hit rate cuts origin traffic by nearly a factor of three, while ninety-seven to ninety-eight cuts it by half again — both of which the hit-rate numbers make look trivial.

   </details>

4. **[Senior] Your hit rate drops from 99% to 97%. How bad is that?**

   <details><summary>Model answer</summary>

   Three times worse, not two per cent worse. Origin load is the request rate times the miss rate, so going from one per cent misses to three per cent triples the traffic reaching the source. If the database was sized against a ninety-nine per cent hit rate, it is now receiving three times its provisioned load, which may well be an outage rather than a degradation. This is why I always convert hit-rate changes into miss-rate ratios before deciding whether something matters.

   </details>

5. **[Staff] Nightly batch jobs are destroying your cache hit rate. What do you do?**

   <details><summary>Model answer</summary>

   First, recognise it as pollution rather than capacity, because that rules out the most tempting response — adding memory achieves nothing against a scan larger than the cache. The immediate fix is to keep the scan out: a bypass flag on the batch path so those reads go direct to the source, or a separate cache instance for batch traffic if it benefits from any caching at all. The durable fix is an admission policy such as W-TinyLFU, where a candidate must beat the eviction victim on estimated frequency, so single-read items never enter the main cache and no application awareness is needed. I would take the bypass immediately for relief and move to admission control as the structural fix, because bypass flags depend on every future batch job remembering to set them, which they will not.

   </details>

6. **[Principal] How do you manage cache behaviour in a shared, multi-tenant cache?**

   <details><summary>Model answer</summary>

   The core problem is that one workload's access pattern becomes another's incident, invisibly and without any code change on the victim's side. Three mechanisms address it. Isolation first: separate instances or explicit per-tenant memory quotas, so a scan-heavy tenant cannot displace an interactive one, accepting slightly lower overall utilisation as the price of predictability. Admission control second, so that even within a shared pool, one-off traffic cannot displace frequently used data — this is the mechanism that makes sharing safe by default rather than by agreement. And per-tenant hit and eviction metrics third, because without attribution the victim sees a hit-rate drop with no explanation and the cause sees nothing at all. Beyond mechanism, I would set the expectation that a shared cache's policy is platform-owned and not individually tunable, since policy interactions between tenants are exactly the kind of thing that cannot be reasoned about locally.

   </details>

---

### Negative caching

*Cache the absence of a result so repeated lookups for missing keys stop reaching the source — with a short TTL, because absence is temporary.*

**Flow:** `Lookup` → `Missing result` → `Negative entry` → `Repeated miss suppressed`

> **The 30-second version**  
> Cache “not found” so repeated lookups for missing keys stop hitting the source — with a short TTL, invalidation on create, and a Bloom filter when the key space is unbounded.

**The problem**

Caches only help with results that exist. A request for a key that is not in the source finds nothing in the cache, queries the source, gets nothing back, and caches nothing — so the next identical request repeats the whole sequence. Every lookup for a missing key is a guaranteed cache miss, forever.

This is not a theoretical inefficiency. Scanners probing for valid identifiers, clients requesting deleted resources, mistyped usernames, and bots enumerating a namespace all generate high volumes of lookups for keys that do not exist — and every one reaches the database.

> **Why it becomes a security problem**  
> An attacker who can generate requests for random keys bypasses your cache entirely by construction. Every request is a miss, so the full request rate hits the source. A cache achieving 99% hit rate on legitimate traffic offers 0% protection against this, which makes it an unusually effective denial-of-service vector against a database that was sized assuming the cache was there.

**Mental model**

“Not found” is a legitimate answer and can be cached like any other. The only difference is that absence is much more likely to change than presence — a missing key may be created at any moment, while an existing value usually persists.

1. **Negative entry** — A marker meaning “the source has no value for this key”, stored under the same key as a positive result would be.
2. **Short TTL** — Because a key can be created at any time, the negative entry's staleness window must be small — seconds, not hours.
3. **Invalidate on create** — When the entity is created, delete the negative entry so it becomes visible immediately rather than after the TTL.
4. **Bloom filter** — A probabilistic pre-filter answering “definitely not present” with no storage per key, which handles unbounded key spaces that negative caching cannot.
5. **Distinguish absence from failure** — A source error is not a negative result. Caching an error as “not found” hides a real problem and returns wrong answers.

> **The asymmetry that sets the TTL**  
> A positive entry becomes wrong when someone updates the value — an event you often control and can invalidate on. A negative entry becomes wrong when someone creates the key — which may be the very user who just got the 404 and is now retrying. Negative TTL should therefore be an order of magnitude shorter than positive TTL, and creation should invalidate it explicitly.

**How it works**

**Negative caching done correctly**

```text
READ
  v = cache.get(k)
  if v is NEGATIVE_MARKER:
      return NotFound                 <- no source query
  if v is None:                       <- true cache miss
      v = db.read(k)
      if v is None:
          cache.set(k, NEGATIVE_MARKER, ttl=10)   <- SHORT
          return NotFound
      cache.set(k, v, ttl=300)
  return v

ON CREATE
  db.insert(k, v)
  cache.delete(k)        <- clears any negative entry

WHAT MUST NOT HAPPEN
  db.read raises a timeout
  -> DO NOT cache that as "not found"
  -> a transient failure would become a cached wrong answer
     for every subsequent reader
```

1. **Use a distinct marker, not a null value** — Many cache clients cannot distinguish “stored null” from “not in cache”. Store an explicit sentinel so the two cases are unambiguous.
2. **Keep negative TTL short** — Seconds. Long enough to absorb a burst of repeated lookups, short enough that a newly created entity appears promptly.
3. **Invalidate on creation** — The user who just received a 404 and then created the resource should see it immediately, not after the TTL.
4. **Never cache errors as absence** — A timeout, a connection failure or a 5xx from the source is not a negative result. Caching it turns a transient fault into a persistent wrong answer.
5. **Use a Bloom filter for unbounded key spaces** — If an attacker can generate arbitrary keys, negative caching fills memory with useless entries. A Bloom filter of existing keys rejects non-members with fixed memory and no per-key storage.
6. **Rate-limit by client for adversarial traffic** — Negative caching reduces source load but not cache load. An enumeration attack still consumes cache capacity and bandwidth, so pair it with rate limiting.

**Bloom filter as the scalable pre-filter**

```text
PROBLEM WITH PLAIN NEGATIVE CACHING
  attacker requests 10 million random keys
  each one creates a negative cache entry
  -> the cache fills with negative entries and EVICTS
     your real working set
  -> the defence becomes the attack

BLOOM FILTER
  maintain a filter containing every key that EXISTS
  on lookup:
    if not filter.might_contain(k): return NotFound
                                     (no cache, no db, ~0 cost)
    else: proceed to cache, then db

  properties
    no false negatives  -> never hides an existing key
    ~1% false positives -> those fall through to the db,
                           which is correct, just not free
    ~10 bits per key    -> 10M keys = 12 MB
  maintenance
    adds are easy; deletes are not (use a counting variant
    or rebuild periodically)
```

> **Negative caching can be the vector, not the defence**  
> Without a bound, an enumeration attack fills your cache with negative entries for keys nobody will ever request again, evicting the working set and collapsing hit rate for legitimate traffic. Either put negative entries in a separate bounded region, use a Bloom filter so they are never stored, or rate-limit aggressively at the edge.

**Worked example**

A public profile lookup API under enumeration by a scraper.

**Layered defence, measured**

```text
TRAFFIC
  legitimate:  20,000 lookups/s, 98% cache hit rate
               -> 400 db queries/s
  scraper:     30,000 lookups/s for random usernames,
               essentially all non-existent

WITHOUT NEGATIVE CACHING
  scraper generates 30,000 db queries/s
  database sized for ~1,000 -> saturated
  legitimate traffic degrades because the db is busy

WITH NAIVE NEGATIVE CACHING (TTL 300s)
  repeated keys are absorbed, but the scraper uses
  RANDOM keys -> almost no repeats
  -> 30,000 db queries/s still, PLUS 30,000 useless
     cache entries/s evicting the real working set
  -> hit rate for legitimate traffic collapses
  -> STRICTLY WORSE

WITH BLOOM FILTER
  filter holds 50M existing usernames -> ~60 MB
  scraper keys fail the filter -> rejected in memory
  db queries from scraper: ~1% false positives = 300/s
  cache pollution: none
  legitimate hit rate: unchanged

PLUS RATE LIMITING
  per-IP and per-token limits cut the scraper's volume
  at the edge before it reaches any of this
```

| Metric | Value | Note |
|---|---|---|
| Scraper load | 30,000/s | random keys |
| Naive negative cache | worse | **pollutes cache** |
| Bloom filter | 300/s | 1% false positives |
| Memory | 60 MB | for 50M keys |

> **Negative caching helps with repeats, not with randomness**  
> The distinction that decides the design: negative caching absorbs *repeated* lookups for the same missing key — a deleted resource that clients keep requesting, a mistyped URL that gets shared. It does nothing for randomly generated keys, where every request is unique, and there it actively harms you by consuming cache capacity. Bloom filters handle the random case; negative caching handles the repeated one.

**When to use it**

- **Repeated lookups for the same missing key**: deleted resources, expired links, stale references from other systems.
- **Expensive miss paths**, where determining that something does not exist costs a full query or a fan-out.
- **Protecting a source** that is sized assuming a high cache hit rate.
- **DNS, permissions and metadata lookups**, where absence is a common and legitimate answer.
- **With a Bloom filter**, when the key space is large or attacker-controlled.

**When to avoid it**

- **Do not cache source errors as absence.** A timeout is not a 404, and conflating them turns a transient fault into a persistent wrong answer.
- **Do not use long negative TTLs**, which hide newly created entities from the user who just created them.
- **Do not negative-cache unbounded or attacker-controlled key spaces** without a Bloom filter or a bounded region.
- **Do not store a null value** where the client cannot distinguish it from a cache miss.
- **Do not rely on it as a denial-of-service defence** — it helps only with repeated keys, and rate limiting is the actual control.

**Advantages**

- **Removes a guaranteed-miss class of traffic** from the source entirely.
- **Very cheap to implement**, needing only a sentinel value and a shorter TTL.
- **Protects against reference storms**, where one broken link is shared widely and generates identical failing lookups.
- **Improves latency for the client**, since a cached 404 returns in microseconds.
- **Composes with Bloom filters**, covering both the repeated and the random cases.

**Disadvantages**

- **Newly created entities are invisible for the negative TTL** unless creation invalidates explicitly.
- **Consumes cache capacity** with entries that hold no value, which matters when the key space is large.
- **Can be turned into an attack vector** by enumeration if unbounded.
- **Error-versus-absence confusion** is an easy and damaging implementation mistake.
- **Does nothing for random key spaces**, which is where the largest volumes usually come from.

**Trade-offs**

**Handling lookups for missing keys**

| Approach | Handles repeats | Handles random keys | Memory |
|---|---|---|---|
| Nothing | No | No | None |
| Negative cache | Yes | No — actively harmful | One entry per distinct missing key |
| Bloom filter | Yes | Yes | ~10 bits per existing key |
| Rate limiting | Partially | Yes | Counters per client |
| Bloom + negative cache + rate limit | Yes | Yes | Modest |

The bottom row is the production answer for a public API: rate limiting bounds abusive volume at the edge, the Bloom filter rejects non-existent keys in memory, and negative caching absorbs the residual repeats for genuinely deleted resources.

**How it fails**

**Negative caching failures**

| Failure | Cause | Fix |
|---|---|---|
| New resource returns 404 for minutes | Long negative TTL, no invalidation on create | Short TTL; delete the key on creation |
| Transient outage becomes persistent 404s | Source error cached as absence | Only cache confirmed absence; never cache errors |
| Cache hit rate collapses under scraping | Negative entries evicting the working set | Bloom filter; bounded negative region; rate limiting |
| Null confused with miss | Cache client cannot distinguish stored null | Use an explicit sentinel marker |
| Bloom filter says present for a deleted key | Standard filters do not support deletion | Counting Bloom filter, or periodic rebuild |
| Filter false-positive rate rising | More keys inserted than the filter was sized for | Monitor fill ratio; resize and rebuild |
| Negative entries never expire | TTL omitted on the negative path | Enforce TTL in the cache client for all entries |

**Limits**

> **Practical numbers**
>
> - **Negative TTL**: typically 1–30 seconds, an order of magnitude below positive TTL.
> - **Bloom filter memory**: ~10 bits per key for a 1% false-positive rate — 50 million keys in about 60 MB.
> - **False positives fall through** to the normal path, so they cost a query, not a wrong answer. False negatives cannot occur.
> - **Bloom filters cannot delete**; use a counting variant or rebuild on a schedule.
> - **Negative entries should be bounded** — either a separate region or a hard cap — so they cannot evict positive entries.

**Alternatives**

| Technique | Best for | Cost |
|---|---|---|
| Negative cache | Repeated lookups for the same missing key | Cache capacity; staleness on create |
| Bloom filter | Large or adversarial key spaces | Memory; deletion difficulty |
| Cuckoo filter | Same, with deletion support | Slightly more memory and complexity |
| Rate limiting | Abusive volume regardless of key | Legitimate bursts may be limited |
| Authenticated access | Removing anonymous enumeration entirely | Product constraint |
| Nothing | Low volume, cheap misses | Source load |

**In real systems**

- **DNS** caches negative responses using the SOA minimum TTL, which is why a newly created record can be invisible for minutes after a failed lookup.
- **LSM-tree storage engines** put a Bloom filter on every file specifically so a point lookup can skip files that definitely do not contain the key.
- **CDNs** cache 404 responses with short TTLs to stop broken links generating origin traffic.
- **Bigtable and similar systems** use Bloom filters to avoid disk reads for row keys that do not exist in a given file.
- **API gateways** combine per-client rate limiting with negative caching, since neither alone handles both repeated and random invalid requests.

**Common mistakes**

- **Caching a source timeout as “not found”**, persisting a wrong answer past the outage.
- **Long negative TTLs**, hiding entities from the user who just created them.
- **No invalidation on create**, so a new resource is invisible until expiry.
- **Negative caching an unbounded key space**, letting enumeration evict the working set.
- **Storing a null rather than an explicit marker**, so miss and absence are indistinguishable.
- **Treating it as a denial-of-service defence** rather than pairing it with rate limiting.
- **Using a Bloom filter without a deletion strategy**, so deleted keys keep passing the filter.

**The staff-level view**

Negative caching is a small technique with two disproportionate failure modes — hiding new data, and becoming an amplifier for enumeration — so the value is in getting the defaults right once.

- **Enforce a short TTL for negative entries in the cache client**, distinct from the positive default, so no team has to remember the asymmetry.
- **Make “never cache an error as absence” an explicit rule**, because the failure is silent and turns a brief outage into persistent incorrect responses.
- **Bound negative entries** in a separate region or with a cap, so enumeration cannot evict the real working set.
- **Reach for a Bloom filter when the key space is attacker-controlled**, since negative caching is the wrong tool there and can make things worse.
- **Pair with edge rate limiting**, which is the actual denial-of-service control; negative caching is a load optimisation, not a defence.

**Go deeper**

Lookups for keys that do not exist are guaranteed cache misses forever, because there is no result to store. Negative caching fixes that by storing an explicit marker meaning “the source has no value for this key”, so repeats are served from cache. It is cheap and effective for deleted resources, broken links and stale references.

Two rules make it safe. The negative TTL must be short — seconds, an order of magnitude below positive TTL — because absence changes when someone creates the key, often the very user who just received the 404; creation should also invalidate the entry explicitly. And a source error must never be cached as absence: a timeout stored as “not found” turns a brief outage into confident wrong answers for the whole TTL.

The important limitation is that it helps with *repeated* missing keys only. Against enumeration with random identifiers there are no repeats, so it absorbs nothing while creating an entry per request that evicts the real working set — the defence becomes the attack. There the right tool is a Bloom filter of existing keys, roughly ten bits each, which rejects non-members in memory with false positives merely falling through to the normal path, paired with edge rate limiting as the actual abuse control.

Negative caching addresses a structural gap: a cache can only store results, so requests for keys with no result bypass it entirely and hit the source every time, forever.

**The mechanism and its asymmetry.** Store an explicit sentinel — not a null, which many clients cannot distinguish from a miss — meaning the source confirmed absence. The critical difference from positive caching is how the entry becomes wrong. A positive entry is invalidated by an update, which is usually an event under your control. A negative entry is invalidated by a *creation*, and the person creating the key is frequently the one who just received the 404 and is now retrying. Hence a short TTL measured in seconds, plus explicit invalidation on create, so the newly created resource is visible immediately rather than after expiry.

**Errors are not absence.** This is the highest-consequence implementation mistake. Caching a timeout, connection failure or 5xx as “not found” converts a transient fault into confident wrong answers for every subsequent reader, lasting for the TTL rather than for the outage. Only a successful query returning no rows justifies a negative entry; failures should propagate, retry, or trip a circuit breaker.

**It helps with repeats, not with randomness.** The value of a negative entry depends entirely on the same key being requested again — a widely shared broken link, a deleted resource clients keep polling, a stale reference from another system. Against enumeration with randomly generated identifiers there are almost no repeats, so nothing is absorbed, while one cache entry is created per request. Those entries evict the genuine working set and collapse hit rate for legitimate traffic, making the defence strictly worse than doing nothing. Recognising this distinction is what separates a correct design from an amplifier.

**Bloom filters cover the unbounded case.** A filter containing every key that exists answers “definitely not present” in memory with no per-key storage for absent keys — roughly ten bits per existing key for a one per cent false-positive rate, so fifty million keys fit in about sixty megabytes. There are no false negatives, so an existing key is never hidden, and false positives merely fall through to the normal path, costing a query rather than producing a wrong answer. The operational caveat is deletion: standard Bloom filters cannot remove entries, so a counting variant or periodic rebuild is needed, and fill ratio must be monitored because the false-positive rate degrades as more keys are added than it was sized for.

**Layering is the production answer.** Edge rate limiting bounds abusive volume regardless of key and is the actual denial-of-service control — negative caching is a load optimisation, not a defence. A Bloom filter rejects non-existent keys cheaply and is indifferent to how many distinct ones are probed. Negative caching with a short TTL, in a bounded region so it cannot evict positive entries, absorbs the residual repeats. Public-facing systems need all three because real traffic contains all of these patterns simultaneously, and none substitutes for the others.

**Prove it — interview questions**

1. **[Basic] What is negative caching and why is it needed?**

   <details><summary>Model answer</summary>

   It is caching the fact that a key has no value in the source, so repeated lookups for the same missing key stop reaching the database. Without it, every request for a non-existent key is a guaranteed cache miss forever, because there is no result to store. That matters because missing-key traffic can be substantial — deleted resources, broken links, mistyped identifiers — and it bypasses the cache entirely by construction.

   </details>

2. **[Basic] Why should negative TTLs be shorter than positive ones?**

   <details><summary>Model answer</summary>

   Because absence is much more likely to change than presence. A positive entry becomes wrong when someone updates the value, which is often an event you control and can invalidate on. A negative entry becomes wrong when someone creates the key — and frequently that someone is the very user who just received the 404 and is now creating the resource. Seconds rather than minutes is the right order, and creation should explicitly delete the negative entry.

   </details>

3. **[Senior] Why must you never cache a source error as absence?**

   <details><summary>Model answer</summary>

   Because they are different answers and conflating them converts a transient fault into a persistent wrong response. If a database query times out and you store “not found”, then every subsequent reader gets a confident 404 for data that exists, for as long as the entry lives — and the original outage may last seconds while the incorrect answers last for the TTL. Only a confirmed absence from a successful query should be cached; errors should propagate or trigger a retry, and ideally a circuit breaker.

   </details>

4. **[Senior] When does negative caching make things worse?**

   <details><summary>Model answer</summary>

   When the key space is large or attacker-controlled. A scraper requesting random identifiers generates almost no repeated keys, so negative caching absorbs nothing — but it does create a cache entry per request, which evicts the genuine working set and collapses hit rate for legitimate traffic. In that scenario the defence is strictly worse than doing nothing. The right tool is a Bloom filter holding the keys that exist, which rejects non-members in memory with no per-key storage, plus rate limiting at the edge to bound the volume.

   </details>

5. **[Staff] Design protection for a public lookup API facing enumeration.**

   <details><summary>Model answer</summary>

   Three layers, because each handles a case the others do not. Edge rate limiting per client and per token bounds abusive volume regardless of what keys are requested, and that is the actual denial-of-service control. A Bloom filter containing every existing identifier then rejects non-existent keys in memory at essentially zero cost — around sixty megabytes for fifty million keys at a one per cent false-positive rate, where false positives simply fall through to the normal path and cost a query rather than producing a wrong answer. Finally, negative caching with a short TTL absorbs the residual repeats for genuinely deleted resources, in a bounded region so it cannot evict positive entries. I would also make sure creation invalidates the negative entry, so a user who gets a 404 and then registers that identifier sees it immediately.

   </details>

6. **[Principal] How do you decide between a Bloom filter and negative caching in general?**

   <details><summary>Model answer</summary>

   By the shape of the missing-key traffic, specifically whether it repeats. Negative caching stores one entry per distinct missing key, so its value depends entirely on those keys being requested again — a widely shared broken link, a deleted resource that clients keep polling. If the keys are largely unique, it consumes capacity proportional to the traffic and returns nothing, and against adversarial traffic it inverts into an amplifier. A Bloom filter is the opposite: memory is proportional to the number of keys that *exist*, which is bounded and known, and it is indifferent to how many distinct non-existent keys are probed. So the rule I use is that bounded, repeating absence favours negative caching, and unbounded or adversarial absence requires a filter. In practice public-facing systems need both, plus rate limiting, because the traffic contains all of these patterns simultaneously — and the important architectural point is that none of them is a substitute for the others.

   </details>

---

### Multi-level caching

*Local memory, shared cache, then origin — each layer faster and smaller than the next, with coherence between them as the real design problem.*

**Flow:** `Local memory` → `Shared cache` → `Origin` → `Version check` → `Response`

> **The 30-second version**  
> Local memory, then shared cache, then origin: four orders of magnitude of latency across the hierarchy. The catch is that the fastest layer's TTL becomes your staleness guarantee.

**The problem**

A shared cache lookup costs a network round trip — perhaps half a millisecond in-region, plus serialisation and connection handling. For a service handling 50,000 requests per second where each request needs five cached values, that is 250,000 network round trips per second spent retrieving data that fits comfortably in local memory.

An in-process cache eliminates that entirely, returning in nanoseconds. But now every instance has its own copy, they can disagree, and an invalidation must reach all of them. Multi-level caching is the decision to accept that coherence problem in exchange for the latency.

> **Each layer trades coherence for speed**  
> The origin is authoritative and slow. The shared cache is one step removed, fast, and coherent across the fleet. The local cache is a thousand times faster and coherent with nothing. As you move up the hierarchy, latency falls by orders of magnitude and the staleness window grows — and the top layer's TTL, not the bottom layer's, determines your actual worst-case staleness.

**Mental model**

A hierarchy of decreasing size and increasing speed, exactly like CPU caches. A request descends until it finds a value, and the value is populated back up on the way out.

1. **L1 — in-process** — Nanoseconds, megabytes, per-instance. No network, no serialisation. Coherent with nothing.
2. **L2 — shared cache** — Sub-millisecond, gigabytes, fleet-wide. One network hop. Coherent across instances.
3. **L3 — origin** — Milliseconds to seconds. Authoritative. The thing you are protecting.
4. **Population** — A miss at L1 checks L2; a miss at L2 checks the origin; the value is written back into both on the way up.
5. **Coherence** — The hard part. An invalidation must reach every L1 in the fleet, and there is no reliable way to know it did.

> **The L1 TTL is your real staleness bound**  
> Teams carefully design invalidation for the shared cache and then add a local cache with a five-minute TTL “for performance”. Worst-case staleness is now five minutes fleet-wide, regardless of how good the L2 invalidation is — and it is invisible, because the shared cache dashboard shows correct data the whole time.

**How it works**

**The hierarchy, and where the time goes**

```text
REQUEST for key K

L1  in-process map            ~50 ns     hit rate 80%
    miss ->
L2  shared cache (network)    ~500 us    hit rate 95% of L1 misses
    miss ->
L3  origin (database)         ~20 ms

EFFECTIVE LATENCY
  0.80 x 50ns
+ 0.20 x 0.95 x 500us
+ 0.20 x 0.05 x 20ms
= ~0.095 ms + 0.2 ms = ~0.3 ms average

WITHOUT L1
  0.95 x 500us + 0.05 x 20ms = ~1.5 ms
-> L1 cuts average latency ~5x and removes 80% of
   shared-cache traffic entirely

ORIGIN LOAD
  with both layers: 1% of requests reach the origin
  L2 alone:         5%
```

1. **Keep L1 small and short-lived** — It exists for the hottest handful of keys. A large L1 with a long TTL buys little extra hit rate and enormously widens the staleness window.
2. **Push invalidations to L1 via pub/sub** — A broadcast channel that every instance subscribes to, deleting the key locally. Best-effort — the TTL remains the guarantee.
3. **Use versioned keys to sidestep coherence** — If the key contains a version, a stale L1 entry is simply unreachable rather than wrong. This removes the fleet-wide invalidation problem entirely.
4. **Size L1 by the hottest keys, not by capacity** — With Zipfian access, a few thousand entries often capture most of the L1-eligible traffic; beyond that the return collapses.
5. **Protect the origin at the bottom** — A concurrency limit belongs at the L2-to-origin boundary, since that is the last line before the authoritative store.
6. **Instrument each layer separately** — Per-layer hit rates diagnose problems that aggregate hit rate hides — an L1 that is too small, or an L2 whose working set no longer fits.

**Coherence strategies for the local layer**

```text
1  SHORT TTL ONLY
   L1 TTL = 5s. Worst-case staleness 5s, always.
   + no coordination, cannot fail
   - 5s of staleness even when nothing changed

2  SHORT TTL + PUB/SUB INVALIDATION
   typical staleness <100ms, worst case 5s
   + fast in the normal case
   - message delivery is best-effort; instances that were
     restarting or disconnected miss it

3  VERSIONED KEYS
   key = "user:42:v" + version
   a stale L1 entry is unreachable, not wrong
   + no invalidation needed at all
   - requires a cheap way to learn the current version
     (often a short-TTL L1 entry for the version itself,
      which moves the staleness to that lookup)

4  WRITE-THROUGH TO ALL LAYERS
   works for the writing instance only; other instances
   still need 1, 2 or 3
```

> **Local caches make deploys a coherence event**  
> Rolling deploys mean instances start at different times with empty L1 caches while others hold populated ones. During the rollout, different users can see different data depending on which instance served them — which produces user reports of values flickering between old and new. Short L1 TTLs bound this; versioned keys eliminate it.

**Worked example**

A permissions service: 200,000 checks per second, changes must take effect quickly, and the consequences of staleness are security-relevant.

**Designing the hierarchy around the guarantee**

```text
REQUIREMENT
  a revoked permission must stop being honoured within 5s
  200,000 checks/s across 300 service instances

NAIVE DESIGN
  L1: 5 min TTL   L2: 30s TTL   pub/sub invalidation
  typical staleness: <1s
  WORST CASE: 5 minutes  <- FAILS the requirement
  and the failure is invisible: L2 is correct throughout

CORRECTED DESIGN
  L1 TTL = 3s (the guarantee)
  L2 TTL = 30s
  pub/sub invalidation to both layers (the optimisation)
  L1 size = 10,000 entries (the hot principals)

COST OF THE SHORT L1 TTL
  a principal checked 50x/s still hits L1 ~99.3% of the time
  at a 3s TTL -> the hit rate barely moves
  L2 traffic rises modestly; origin traffic unchanged

ALTERNATIVE: VERSIONED KEYS
  key = "perm:{principal}:v{policy_version}"
  policy_version cached with a 2s TTL
  -> permission entries can be cached for minutes,
     because a policy change makes them unreachable
  -> one short-TTL lookup instead of many
```

| Metric | Value | Note |
|---|---|---|
| Checks | 200k/s | 300 instances |
| Naive worst case | 5 min | **fails requirement** |
| L1 TTL 3 s | 99.3% hit | cost is negligible |
| Versioned keys | one short lookup | best structure |

> **Short TTLs are cheap for hot keys — that is what makes this work**  
> The reflex against short local TTLs is that they will destroy hit rate. For genuinely hot keys they do not: a key accessed fifty times per second retains over 99% hit rate at a three-second TTL. The cost of a short L1 TTL falls almost entirely on cold keys, which were contributing little anyway. This asymmetry is what lets you have both the latency and the guarantee.

**When to use it**

- **Very high read rates** where the shared cache round trip is a meaningful share of request latency.
- **Small hot datasets** — permissions, feature flags, configuration, routing tables — where a few thousand entries cover most traffic.
- **Reducing load on a shared cache tier**, which can itself become a bottleneck or a hot-key victim.
- **Latency-critical paths**, where nanoseconds versus microseconds changes the p99.
- **With versioned keys**, which make the local layer safe without fleet-wide invalidation.

**When to avoid it**

- **Do not add a local cache without recomputing the staleness guarantee** — the L1 TTL becomes the bound.
- **Do not use large local caches with long TTLs** for data that changes; the hit-rate gain is small and the staleness cost is large.
- **Do not rely on pub/sub invalidation as the guarantee**; instances restarting or disconnected will miss it.
- **Do not cache locally where per-instance divergence is user-visible and unacceptable**, such as anything where two adjacent requests must agree.
- **Do not add layers without per-layer metrics**, or you cannot diagnose which one is misbehaving.

**Advantages**

- **Order-of-magnitude latency reduction** for hits at the local layer.
- **Large reduction in shared-cache traffic**, which protects that tier from becoming the bottleneck.
- **Resilience**: a local cache keeps serving during a brief shared-cache outage.
- **Hot-key protection**, since the hottest keys are absorbed locally and never reach a single shared-cache shard.
- **Composable with versioned keys**, which removes the coherence problem rather than managing it.

**Disadvantages**

- **Coherence is genuinely hard** — invalidation must reach every instance, and you cannot verify that it did.
- **The weakest layer sets the staleness bound**, which is usually the one added last for performance.
- **Per-instance divergence** means two adjacent requests can see different data.
- **Memory cost per instance**, multiplied across the fleet.
- **Deploys become coherence events**, with mixed populated and empty caches during a rollout.
- **More layers means more places for a bug to hide**, and aggregate metrics conceal which layer is at fault.

**Trade-offs**

**Layer characteristics**

|  | L1 in-process | L2 shared | L3 origin |
|---|---|---|---|
| Latency | ~50 ns | ~0.5 ms | ~20 ms |
| Capacity | MB per instance | GB fleet-wide | Everything |
| Coherence | None | Fleet-wide | Authoritative |
| Invalidation | Broadcast, best-effort | Direct, reliable | n/a |
| Survives instance restart | No | Yes | Yes |
| Cost | Memory × instances | Cache tier | The thing being protected |

> **The framing that shows judgement**  
> “I'd add a local cache with a three-second TTL for the hot principals, which takes 80% of traffic off the shared tier at negligible cost to hit rate. The important consequence is that our worst-case staleness is now three seconds rather than thirty — so I'd confirm three seconds satisfies the revocation requirement before shipping it.”

**How it fails**

**Multi-level cache failures**

| Symptom | Cause | Fix |
|---|---|---|
| Stale data despite correct shared cache | Local cache TTL too long; invalidation missed | Short L1 TTL; versioned keys |
| Users see values flickering | Instances disagree; requests hit different ones | Short L1 TTL; session affinity for reads; versioned keys |
| Revocation not taking effect | L1 TTL exceeds the security requirement | Set L1 TTL from the requirement, not from performance |
| Shared cache still overloaded | L1 too small or TTL too short for the hot set | Size L1 to the hot keys; check per-layer hit rates |
| Memory pressure across the fleet | Large L1 in every instance | Bound L1 size explicitly; it should be small by design |
| Inconsistent behaviour during deploys | Mixed empty and populated local caches | Short L1 TTL; warm on start; versioned keys |
| Cannot diagnose which layer is wrong | Only aggregate hit rate measured | Per-layer hit, miss and eviction metrics |

**Limits**

> **Design numbers**
>
> - **Latency ratios**: local ~50 ns, shared ~500 µs, origin ~20 ms — roughly four orders of magnitude across the hierarchy.
> - **L1 TTL** of 1–10 s is typical; it is the staleness guarantee, so derive it from the requirement.
> - **L1 size**: a few thousand entries usually captures most eligible traffic under Zipfian access.
> - **Hot-key hit rate at short TTL**: a key read 50 times per second still hits ~99% at a 3 s TTL.
> - **Pub/sub invalidation** propagates in well under a second when healthy; assume it can be minutes when degraded.

**Alternatives**

| Approach | Gives | Costs |
|---|---|---|
| Shared cache only | Coherent, simple | Network hop per lookup |
| Local + shared | Very low latency; less shared load | Coherence; staleness bound rises |
| Local + versioned keys | Low latency with no coherence problem | A version lookup per read |
| Push-based local state | Near-zero staleness for small datasets | Connection management; fan-out |
| No cache on the hot path | Perfect freshness | Full origin load |

Push-based distribution deserves consideration for small, critical datasets: rather than caching with a TTL, every instance subscribes to a stream and maintains a full local copy. That gives local-memory latency with sub-second propagation — the approach feature flag and configuration services typically take.

**In real systems**

- **CPU cache hierarchies (L1/L2/L3)** are the model, including the coherence protocols that distributed caches informally imitate.
- **CDN tiers** — browser, edge PoP, regional shield, origin — are multi-level caching applied geographically.
- **Feature flag SDKs** keep a full local copy updated by streaming, which is the push-based end of this spectrum.
- **Caffeine or Guava in front of Redis** is the standard JVM pattern for this, and the standard place where local TTLs quietly become the staleness bound.
- **Database buffer pools in front of the storage layer** are the same hierarchy inside a single process.

**Common mistakes**

- **Adding a local cache without recomputing worst-case staleness.**
- **Long local TTLs** chosen for hit rate rather than derived from a requirement.
- **Treating pub/sub invalidation as a guarantee**, when restarting or disconnected instances miss it.
- **Large per-instance caches**, multiplying memory cost across the fleet for marginal hit-rate gain.
- **Only measuring aggregate hit rate**, so you cannot tell which layer is failing.
- **Ignoring deploy-time divergence**, producing flickering values users report and engineers cannot reproduce.
- **Caching security-relevant decisions locally** with a TTL longer than the revocation requirement.

**The staff-level view**

Adding a local cache is an easy performance win that silently changes a system-wide guarantee, which makes it exactly the kind of change worth catching in review.

- **Require the staleness analysis to be redone whenever a layer is added.** The newest, fastest layer usually sets the worst case, and the dashboards of the older layers will look perfectly healthy.
- **Keep local caches small and short-lived by default**, since the hit-rate cost of a short TTL falls almost entirely on cold keys that were contributing little.
- **Prefer versioned keys for multi-layer designs**, because they convert fleet-wide invalidation — an unsolvable delivery problem — into a lookup.
- **Insist on per-layer metrics.** Aggregate hit rate cannot tell you whether L1 is undersized or L2's working set no longer fits.
- **Check the deploy behaviour.** Mixed populated and empty local caches during a rollout produce user-visible flicker that is hard to reproduce afterwards.

**Go deeper**

A local in-process cache returns in nanoseconds; a shared cache costs a network round trip of around half a millisecond; the origin costs tens of milliseconds. Placing them in a hierarchy — check L1, then L2, then the origin, populating back up — cuts average latency several-fold and removes most traffic from the shared tier, which also protects it from hot keys.

The cost is coherence. Each instance has its own local copy, they can disagree, and an invalidation broadcast may not reach instances that are restarting or briefly disconnected. The consequence teams miss is that the *local* TTL becomes the fleet-wide staleness bound: a five-minute local cache added for performance silently overrides a carefully designed thirty-second guarantee, and the shared cache dashboards look perfectly healthy throughout.

Two things make this safe. Short local TTLs, which are far cheaper than people expect — a key read fifty times per second still hits over 99% at a three-second TTL, because the cost falls on cold keys that contributed little. And versioned keys, which embed a version so a stale local entry is unreachable rather than wrong, removing fleet-wide invalidation from the problem entirely.

Multi-level caching applies the CPU memory hierarchy to distributed systems: each layer is faster and smaller than the one below, and each is one step further from authoritative truth.

**The latency argument.** Local in-process access is around fifty nanoseconds; a shared cache lookup is roughly half a millisecond including serialisation and connection handling; the origin is tens of milliseconds. Four orders of magnitude separate the ends. For a service making several cached lookups per request at high throughput, an eighty per cent local hit rate cuts average latency several-fold and removes eighty per cent of the traffic from the shared tier — which matters independently, because the shared cache can itself become a bottleneck or suffer hot-key concentration on a single shard.

**The coherence cost, and where it hides.** Every instance holds its own copy with no mechanism guaranteeing agreement. An invalidation must be broadcast to the whole fleet, and instances restarting, deploying or briefly disconnected will miss it, with no way to detect that they did. The consequence that teams consistently overlook is that the *local* TTL becomes the system's worst-case staleness bound. A five-minute local cache added for latency silently overrides a thirty-second guarantee designed into the shared tier — and the failure is invisible because the shared cache is correct the whole time while users see stale data from whichever instance served them.

**Short local TTLs are cheaper than intuition suggests.** The reflexive objection is that a three-second TTL will destroy hit rate. For hot keys it does not: a key accessed fifty times per second is re-read roughly a hundred and fifty times within each window, so hit rate stays above ninety-nine per cent. The cost falls almost entirely on cold keys, which were contributing little. This asymmetry is what allows a design to have both local-memory latency and a tight staleness guarantee, and recognising it resolves most arguments about local caching.

**Versioned keys are the structural answer.** Embedding a version from the underlying data in the cache key means a stale local entry is unreachable rather than wrong, which eliminates fleet-wide invalidation — a delivery problem that cannot be solved reliably — and replaces it with a lookup. The version itself becomes the one short-TTL entry, while the values it guards can be cached for minutes. This is why it is the right default for multi-layer designs, particularly for security-relevant data where per-instance divergence is unacceptable.

**Operational details that bite.** Rolling deploys create mixed populated and empty local caches, so different users see different values during the rollout — producing flicker reports that are impossible to reproduce afterwards; short local TTLs bound this and versioned keys remove it. Per-layer metrics are essential, because aggregate hit rate cannot distinguish an undersized L1 from an L2 whose working set no longer fits. And the concurrency limit protecting the origin belongs at the L2-to-origin boundary, since that is the last line before the authoritative store. Organisationally, the practice worth enforcing is that the staleness analysis is re-done whenever a layer is added, because the team adding the layer is optimising latency and has no reason to look at the guarantee they just changed.

**Prove it — interview questions**

1. **[Basic] Why add an in-process cache in front of a shared one?**

   <details><summary>Model answer</summary>

   To remove the network round trip. A shared cache lookup costs perhaps half a millisecond plus serialisation and connection handling, while an in-process map returns in nanoseconds. For a service making several cached lookups per request at high throughput, that difference dominates request latency and also removes most of the traffic from the shared tier, which protects it from becoming the bottleneck.

   </details>

2. **[Basic] What is the main cost of a local cache layer?**

   <details><summary>Model answer</summary>

   Coherence. Every instance has its own copy, so they can disagree, and an invalidation must reach all of them with no reliable way to confirm it did. The practical consequence is that the local TTL becomes your worst-case staleness bound fleet-wide — which is easy to miss, because the shared cache will show correct data throughout while users see stale values from whichever instance served them.

   </details>

3. **[Senior] How do you keep a local cache coherent?**

   <details><summary>Model answer</summary>

   Three options, in increasing robustness. A short TTL alone is the guarantee that cannot fail, because it needs no message to arrive. Adding pub/sub invalidation brings typical staleness down to milliseconds, but instances that were restarting or briefly disconnected miss the message, so it is an optimisation rather than a guarantee. Versioned keys are the strongest: embedding a version in the key means a stale local entry is simply unreachable rather than wrong, which removes fleet-wide invalidation from the problem entirely — at the cost of needing a cheap way to learn the current version, which usually becomes the one short-TTL lookup.

   </details>

4. **[Senior] Won't a short local TTL destroy the hit rate?**

   <details><summary>Model answer</summary>

   Not for hot keys, which is where local caching earns its value. A key accessed fifty times per second still hits over ninety-nine per cent of the time at a three-second TTL, because it is re-read many times within each window. The cost of a short TTL falls almost entirely on cold keys, which were contributing little hit rate anyway. That asymmetry is what makes it possible to have both very low latency and a tight staleness guarantee, and it is why the reflexive objection to short local TTLs is usually wrong.

   </details>

5. **[Staff] Design caching for a permissions service where revocation must take effect within 5 seconds.**

   <details><summary>Model answer</summary>

   The binding constraint is that the local TTL sets the worst case, so it must be under five seconds regardless of how good the invalidation is — I would use three seconds with jitter, which for principals checked frequently costs almost nothing in hit rate. Above that, a shared cache with a thirty-second TTL, and pub/sub invalidation to both layers as the optimisation that brings typical propagation under a second. The design I would actually prefer is versioned keys: cache permissions under a key containing the policy version, and cache the policy version itself with a two-second TTL. Then permission entries can live for minutes because a policy change makes them unreachable, and there is exactly one short-TTL lookup rather than many. Either way, the thing I would insist on is that the guarantee is stated and tested: if pub/sub were completely down, would revocation still take effect within five seconds?

   </details>

6. **[Principal] What organisational practice prevents multi-level caching from eroding guarantees?**

   <details><summary>Model answer</summary>

   Make the staleness analysis a required artifact that is re-done whenever a layer is added, because the failure is structurally invisible: the team adding a local cache is optimising latency, the shared cache dashboards stay green, and the guarantee silently degrades from thirty seconds to five minutes with nobody noticing until a security or correctness incident. Concretely, I would require each cached dataset to document its staleness tolerance, the layer-by-layer TTLs, and the resulting worst case, reviewed when any layer changes. The cache client library should enforce a short default TTL for in-process caches specifically — distinct from the shared-cache default — so the safe choice is the easy one. And I would push versioned keys as the standard pattern for multi-layer caching, because fleet-wide invalidation delivery is a problem that cannot be solved reliably, whereas making stale entries unreachable sidesteps it entirely.

   </details>

---
