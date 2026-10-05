# Curriculum · Database Internals

[← System Design index](../README.md)

> 8 lessons in **Database Internals**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Database Internals** (8): [B-tree indexes](#b-tree-indexes) · [LSM trees and compaction](#lsm-trees-and-compaction) · [Write-ahead logging](#write-ahead-logging) · [MVCC snapshots](#mvcc-snapshots) · [Transaction isolation levels](#transaction-isolation-levels) · [Optimistic concurrency control](#optimistic-concurrency-control) · [Deadlocks and lock ordering](#deadlocks-and-lock-ordering) · [Query planning and statistics](#query-planning-and-statistics)

## Database Internals

### B-tree indexes

*A shallow balanced tree of sorted pages that turns a full scan into three or four page reads — and quietly taxes every write you make.*

**Flow:** `Predicate` → `Root page` → `Internal page` → `Leaf entries` → `Rows`

> **The 30-second version**  
> A shallow balanced tree of sorted pages: three or four page reads for any row, sequential range scans, and free sort order — paid for with a write tax on every insert and update.

**The problem**

Without an index, finding a row means reading every row. At 100 million rows that is a full table scan: gigabytes of IO for a query that returns one record. The obvious fix — keep the data sorted — fails because you can only sort by one thing, and inserting into the middle of a sorted array is O(n).

A B-tree solves both: it maintains sorted order under insertion in logarithmic time, and because each node holds hundreds of keys, the tree stays three or four levels deep even at billions of rows. Every lookup is a handful of page reads.

> **The number that makes B-trees work**  
> A 8 KB page holding ~200 keys gives a fanout of 200. Three levels addresses 200³ = 8 million entries; four levels addresses 1.6 billion. So **any row in a billion-row table is three or four page reads away**, and the top levels are almost always in memory — meaning the real cost is usually one disk read.

**Mental model**

Picture a sorted phone book with an index at the front pointing to page ranges, and those pages having their own sub-indexes. You never scan; you descend.

1. **Root and internal pages** — Hold separator keys and child pointers. Almost always cached in memory because every query touches them.
2. **Leaf pages** — Hold the actual index entries in sorted order, linked to their siblings so range scans are sequential.
3. **Fanout** — Keys per page. Higher fanout means a shallower tree, which is why smaller keys are materially better.
4. **Clustered versus secondary** — A clustered index stores the row itself in the leaf; a secondary index stores a pointer, so it may require a second lookup.
5. **Write amplification** — Every insert, update of an indexed column, and delete must maintain every relevant index — plus page splits when a page fills.

> **Indexes are not free, and the cost lands on writes**  
> Each index roughly adds one random write per row modification, plus space, plus buffer-pool pressure that evicts useful data. A table with twelve indexes writes thirteen structures per insert. The most common database performance problem is not a missing index — it is a table carrying indexes nobody queries.

**How it works**

**Descending the tree, and why range scans are cheap**

```text
SELECT * FROM orders WHERE customer_id = 4711;

  [root]            keys: 1000 | 5000 | 9000
     |                           |
     v                     (4711 < 5000)
  [internal]        keys: 4000 | 4500 | 4800
     |                           |
     v                     (4711 >= 4500)
  [leaf]  ... 4709 4710 4711 4712 ...  -> row pointers

3 page reads. Top two levels are cached -> ~1 disk read.

RANGE SCAN: WHERE customer_id BETWEEN 4711 AND 4900
  descend once, then follow leaf-page sibling pointers
  -> SEQUENTIAL reads, not repeated descents
  -> this is why B-trees beat hash indexes for ranges
```

1. **Composite index column order is the whole game** — An index on `(a, b, c)` serves predicates on `a`, on `(a, b)`, and on `(a, b, c)` — but not on `b` alone. Put equality predicates leftmost, then the range or sort column.
2. **Covering indexes avoid the second lookup** — If the index contains every column the query needs, the engine never touches the table. Adding an `INCLUDE` column can turn two IOs into one across millions of rows.
3. **Selectivity decides whether the index is used at all** — If a predicate matches 30% of rows, descending the index and then fetching each row randomly is slower than a sequential scan — and the planner will correctly ignore your index.
4. **Beware wrapping the indexed column** — `WHERE lower(email) = ?` cannot use an index on `email`. Either index the expression or store the normalised value.
5. **Watch for page splits and fragmentation** — Random-order inserts (UUID v4 primary keys) split pages constantly, wasting space and causing random IO. Monotonic keys append to the rightmost page instead.
6. **Audit unused indexes regularly** — Most databases track index usage. Indexes with zero scans are pure write tax and should be dropped.

**Composite index: what it can and cannot serve**

```text
CREATE INDEX ON orders (customer_id, status, created_at);

SERVED (leftmost prefix rule)
  WHERE customer_id = 1
  WHERE customer_id = 1 AND status = 'paid'
  WHERE customer_id = 1 AND status = 'paid'
        AND created_at > '2024-01-01'
  WHERE customer_id = 1 ORDER BY status, created_at

NOT SERVED (efficiently)
  WHERE status = 'paid'                       -- skips customer_id
  WHERE created_at > '2024-01-01'             -- skips two columns
  WHERE customer_id = 1 AND created_at > ...  -- gap at status:
                                                 only customer_id
                                                 narrows the scan

RULE: equality columns first, then ONE range column last.
```

> **UUID primary keys are a real cost**  
> Random UUIDs scatter inserts across the whole tree, causing page splits, poor cache locality and index bloat. Time-ordered identifiers — UUIDv7, ULID, or a snowflake-style id — insert at the right edge like an auto-increment key while keeping global uniqueness. On a write-heavy table this is frequently a 2–3× difference in write throughput.

**Worked example**

A slow query on a 200-million-row orders table, worked through properly.

**Diagnosing and fixing with the leftmost-prefix rule**

```text
QUERY
  SELECT id, total FROM orders
  WHERE customer_id = 4711 AND status = 'paid'
  ORDER BY created_at DESC LIMIT 20;

EXISTING INDEXES
  (customer_id)          -- matches 3,000 rows
  (status)               -- matches 80,000,000 rows, never used
  (created_at)           -- used for the sort, scans everything

WHAT THE PLANNER DOES
  use (customer_id) -> 3,000 rows
  fetch each row from the heap (3,000 random IOs)
  filter status = 'paid'
  sort 900 surviving rows
  return 20
  -> ~3,000 random reads for 20 results

THE FIX
  CREATE INDEX ON orders (customer_id, status, created_at DESC)
                 INCLUDE (total);
  -> descend to (4711,'paid'), read the first 20 leaf entries
     in already-sorted order, all columns present
  -> ~4 page reads, no heap access, no sort

DROP (status): 80M-row selectivity means it is never chosen,
               and it taxes every write.
```

| Metric | Value | Note |
|---|---|---|
| Before | ~3,000 IOs | plus a sort |
| After | ~4 IOs | **covering, pre-sorted** |
| Write cost | one fewer index | dropped `(status)` |
| Key insight | order matters | equality, then range |

> **Three wins from one index definition**  
> The composite index did three things simultaneously: it narrowed to the matching rows, it delivered them already sorted so the `ORDER BY` disappeared, and the `INCLUDE` made it covering so the heap was never touched. Recognising that a single well-ordered index can eliminate the filter, the sort and the lookup is the core skill in index design.

**When to use it**

- **Equality and range predicates** on columns with reasonable selectivity.
- **Sort and pagination**, where an index in the right order removes the sort entirely.
- **Uniqueness enforcement**, where a unique index is both a constraint and an access path.
- **Foreign key columns**, which are otherwise scanned on every parent delete or update.
- **Covering frequent queries**, where including a few columns eliminates heap access.

**When to avoid it**

- **Do not index low-selectivity columns alone** — a boolean or a two-value status is rarely worth it, except as a partial index.
- **Do not index write-heavy tables speculatively**; each index is a per-row write tax.
- **Do not index columns you always wrap in a function** unless you index the expression itself.
- **Do not use random UUIDs as clustered primary keys** on high-insert tables.
- **Do not create single-column indexes for every column** hoping the planner will combine them; a composite index in the right order is usually far better.

**Advantages**

- **Logarithmic lookup** that stays three or four levels deep even at billions of rows.
- **Range scans are sequential** across linked leaf pages, which hash indexes cannot do.
- **Sorted order comes free**, so `ORDER BY` and pagination can avoid sorting entirely.
- **Self-balancing under insert and delete**, with no periodic rebuild required for correctness.
- **Universally implemented and well understood**, with mature planner and tooling support.

**Disadvantages**

- **Every write pays** for every index on the table, including page splits.
- **Storage overhead** of roughly the indexed columns plus pointers, often 10–30% of table size per index.
- **Random inserts fragment pages**, wasting space and causing random IO.
- **Only the leftmost prefix is usable**, so column order constrains which queries benefit.
- **Low-selectivity predicates make the index useless**, and the planner will ignore it.
- **Buffer-pool competition**: index pages evict data pages and vice versa.

**Trade-offs**

**Index types compared**

| Type | Serves | Does not serve |
|---|---|---|
| B-tree | Equality, ranges, sort order, prefix matching | Full-text, high-dimensional similarity |
| Hash | Exact equality only, slightly faster | Ranges, sorting |
| Bitmap | Low-cardinality columns in analytics | High-concurrency OLTP writes |
| GIN / inverted | Array containment, full text, JSON keys | Ranges on scalars |
| BRIN / zone map | Huge tables correlated with physical order | Unsorted or random data |
| Partial index | A hot subset (`WHERE status='pending'`) | Queries outside the predicate |

Partial indexes are the most under-used option: indexing only the 0.1% of rows in a `pending` state gives a tiny index that is always cached, while costing nothing for the 99.9% of rows that are not pending.

**How it fails**

**Index-related failures**

| Symptom | Cause | Fix |
|---|---|---|
| Index exists but is not used | Low selectivity, or a function wraps the column | Check the plan; index the expression; use a partial index |
| Writes slow down over time | Too many indexes; page splits from random keys | Drop unused indexes; use time-ordered ids |
| Query fast in test, slow in production | Stale statistics or different data distribution | Update statistics; check the actual plan against the estimate |
| Index much larger than expected | Bloat from updates and deletes | Rebuild or reindex concurrently; tune autovacuum |
| `ORDER BY ... LIMIT` scans everything | Sort column not the trailing column of the index | Add it to the index in matching direction |
| Unique index creation fails | Existing duplicates | Find and resolve duplicates first; build concurrently |
| Deletes on a parent table are slow | No index on the child's foreign key column | Index every foreign key |

**Limits**

> **Numbers worth knowing**
>
> - **Fanout**: ~100–300 keys per 8 KB page; tree depth 3–4 for hundreds of millions of rows.
> - **Selectivity threshold**: an index is typically only worthwhile below roughly 5–10% of rows matched.
> - **Write cost**: roughly one extra random write per index per row modification.
> - **Space**: each index commonly adds 10–30% of table size.
> - **Key size matters**: wide keys reduce fanout and deepen the tree — prefer narrow keys, or hash long values.

**Alternatives**

| Structure | Strength | Weakness |
|---|---|---|
| B-tree | Ranges, sorts, general purpose | Write amplification; random-insert fragmentation |
| LSM tree | Very high write throughput, sequential IO | Read amplification; compaction cost |
| Hash index | Fastest exact lookup | No ranges or ordering |
| Full scan | Simplest; fine for small tables or high selectivity | Linear cost |
| Materialised view / projection | Precomputed answer | Staleness; maintenance cost |

**In real systems**

- **PostgreSQL, MySQL InnoDB and SQL Server** all use B+-tree variants as the default index type, with InnoDB clustering the table itself by primary key.
- **InnoDB's clustered primary key** means secondary index entries store the primary key, so a wide primary key inflates every secondary index — a strong argument for narrow keys.
- **UUIDv7 and ULID** were designed specifically to fix the random-insert fragmentation that UUIDv4 causes in B-tree indexes.
- **Partial indexes** are widely used for queue-like tables, indexing only rows in a pending state.
- **`CREATE INDEX CONCURRENTLY`** exists because building an index normally takes a lock that is unacceptable on a production table.

**Common mistakes**

- **Creating a single-column index per column** instead of one well-ordered composite.
- **Putting the range column before equality columns** in a composite index.
- **Indexing a column that is always wrapped in a function.**
- **Random UUID primary keys** on write-heavy tables.
- **Never dropping indexes**, so write cost grows silently.
- **Forgetting indexes on foreign key columns**, making parent deletes scan.
- **Building indexes with a blocking lock** on a production table.

**The staff-level view**

Index design is where a small amount of expertise produces disproportionate results, and where accumulated neglect produces slow, expensive systems.

- **Review indexes as part of schema review**, with the query that justifies each one named in the migration.
- **Audit and drop unused indexes on a schedule.** Every database accumulates them, and each is a permanent write tax that nobody is measuring.
- **Standardise time-ordered identifiers** for high-insert tables; the write-throughput difference against random UUIDs is large and entirely avoidable.
- **Teach the leftmost-prefix rule explicitly.** Most “the index isn't being used” tickets are a column-order misunderstanding.
- **Require concurrent index builds** in production, and treat any migration taking a long lock on a large table as a defect.

**Go deeper**

A B-tree keeps entries sorted while supporting logarithmic insertion. High fanout — hundreds of keys per page — keeps the tree three or four levels deep even at hundreds of millions of rows, and the upper levels stay cached, so a lookup costs about one disk read. Leaf pages are linked in order, so a range scan descends once and then reads sequentially, which is why B-trees beat hash indexes for anything but exact equality.

Composite index column order determines which queries benefit: an index on `(a, b, c)` serves predicates on leftmost prefixes only, so equality columns go first and a single range or sort column goes last. Done well, one index can eliminate the filter, the sort and the heap lookup simultaneously — the sort disappears because the index is already in order, and an included column makes it covering.

The cost lands on writes: roughly one extra random write per index per row modification, plus 10–30% of table size each, plus page splits. Random UUID primary keys make this markedly worse by scattering inserts across the tree; time-ordered identifiers such as UUIDv7 append at the right edge instead. The most common real-world problem is not a missing index but a table carrying indexes nobody queries.

The B-tree is the default index structure in virtually every relational database because it simultaneously provides fast point lookups, efficient range scans, and maintained sort order — while remaining balanced under arbitrary insertion and deletion.

**Why it stays shallow.** Each page holds on the order of a hundred to a few hundred keys, so fanout is high: three levels address millions of entries, four address billions. Since the root and internal levels are touched by every query they remain in the buffer pool, meaning the practical cost of a lookup is a single random read of a leaf page regardless of table size. Leaf pages are linked to their siblings, so a range query descends once and then reads sequentially — this is the property hash indexes lack and the reason B-trees dominate general-purpose use.

**Composite order is the design.** Entries in an index on `(a, b, c)` are sorted by `a`, then `b`, then `c`, so only leftmost prefixes can narrow the scan. Equality predicates belong first and a single range or sort column last; once a range condition is applied, subsequent columns can no longer restrict the search, only filter. The payoff for getting this right is that one index can do three jobs at once — narrow to matching rows, deliver them already sorted so the `ORDER BY` vanishes, and, with included columns, cover the query so the table is never touched. Recognising that is the core skill in index design.

**Selectivity governs use.** If a predicate matches a large fraction of rows, descending the index and then performing a random heap fetch per row is slower than a sequential scan, so the planner correctly ignores the index. This surprises people, and the diagnosis is always to read the actual plan and compare estimated with actual row counts — which also distinguishes a genuine design problem from stale statistics. Partial indexes are the underused remedy: indexing only the small subset of rows in a hot state gives a tiny, always-cached index with almost no write cost for the rest of the table.

**The write tax is the hidden cost.** Every insert, delete, and update of an indexed column must maintain every relevant index, adding roughly one random write each, plus page splits when pages fill. A table with twelve indexes writes thirteen structures per row. Random UUID primary keys compound this badly by scattering insertions across the entire tree, causing constant splits, poor locality and bloat; time-ordered identifiers like UUIDv7 or ULID append at the right edge while preserving global uniqueness, and on write-heavy tables the difference is frequently two to three times in throughput.

**Maintenance and organisation.** Indexes accumulate and are never removed, because nobody remembers which query justified each one. The durable fixes are procedural: require the justifying query to be named in the migration that creates an index; run a scheduled report of unused and redundant indexes routed to owning teams as tickets; put time-ordered identifiers in the service template; and make migration tooling refuse non-concurrent index builds on large tables, since a blocking build on a production table is a defect rather than an inconvenience. Most “the index isn't being used” reports are a leftmost-prefix misunderstanding, which is why slow-query tooling should surface plans and not merely durations.

**Prove it — interview questions**

1. **[Basic] Why is a B-tree lookup fast even on a huge table?**

   <details><summary>Model answer</summary>

   Because each page holds hundreds of keys, so the fanout is high and the tree stays shallow — three or four levels covers hundreds of millions of rows. Every lookup descends that fixed number of levels, and the top levels are almost always cached in memory, so the practical cost is about one disk read regardless of table size. Range scans are cheap too, because leaf pages are linked in sorted order and can be read sequentially after a single descent.

   </details>

2. **[Basic] What is the leftmost prefix rule?**

   <details><summary>Model answer</summary>

   A composite index on `(a, b, c)` can serve predicates on `a`, on `a` and `b`, or on all three — but not on `b` alone or `c` alone, because the entries are sorted by `a` first. The practical guidance is to put equality predicates leftmost and a single range or sort column last, since once you hit a range condition, columns after it can no longer narrow the scan.

   </details>

3. **[Senior] Why might the planner ignore an index you created?**

   <details><summary>Model answer</summary>

   Most often selectivity: if the predicate matches a large fraction of rows, descending the index and then fetching each row randomly from the heap is slower than a sequential scan, so ignoring the index is the correct decision. Other common causes are a function wrapping the indexed column so the expression no longer matches, stale statistics causing a bad cardinality estimate, a type mismatch preventing the index from being used, and column order not matching the predicate. I would read the actual plan rather than guess, and compare estimated to actual row counts to distinguish a statistics problem from a design one.

   </details>

4. **[Senior] What is a covering index and when is it worth it?**

   <details><summary>Model answer</summary>

   An index that contains every column a query needs, so the engine answers entirely from the index and never touches the table. It is worth it when a frequent query returns a small number of extra columns and the heap fetch is the dominant cost — turning two IOs per row into one, which across millions of rows is substantial. The trade-off is index size and write cost, so I would add included columns for a specific hot query rather than speculatively, and I would check whether the query is frequent enough to justify taxing every write to that table.

   </details>

5. **[Staff] A write-heavy table has twelve indexes and insert throughput has degraded. How do you approach it?**

   <details><summary>Model answer</summary>

   First I would get index usage statistics, because most databases track scans per index and there are almost always several with zero reads — those are pure write tax and can be dropped immediately. Then I would look for redundancy: an index on `(a)` is subsumed by one on `(a, b)`, so the narrower one is often removable. Next I would check the primary key: if it is a random UUID, every insert splits pages across the whole tree, and moving to a time-ordered identifier typically recovers a large fraction of throughput on its own. Finally I would examine whether some indexes could be partial — indexing only the small subset of rows actually queried — which keeps the access path while removing most of the write cost. I would make each change separately with measurement, because index changes are easy to reverse and easy to misattribute.

   </details>

6. **[Principal] How do you keep index design healthy across many teams and databases?**

   <details><summary>Model answer</summary>

   By making it visible and reviewed rather than emergent. Every migration that adds an index should name the query it serves, so the index has a stated purpose that can later be re-evaluated — indexes without a documented reason are the ones nobody dares drop. A scheduled report of unused and redundant indexes per database, routed to the owning team as a ticket rather than a dashboard nobody reads, catches the accumulation that otherwise continues indefinitely. Platform defaults matter too: time-ordered identifiers in the service template, and migration tooling that refuses non-concurrent index builds on large tables, remove two of the most common failures without anyone needing to remember. And I would ensure slow-query reporting routinely surfaces plans rather than just durations, because the leftmost-prefix misunderstanding is extremely common and is only diagnosable from a plan.

   </details>

---

### LSM trees and compaction

*Buffer writes in memory, flush them as sorted immutable files, and merge those files in the background — trading read and space amplification for sequential writes.*

**Flow:** `WAL` → `Memtable` → `Sorted file` → `Compaction` → `Read merge`

> **The 30-second version**  
> Buffer writes in memory, flush sorted immutable files, merge them in the background. Sequential writes and great compression, paid for with read amplification, compaction load and expensive deletes.

**The problem**

B-trees update pages in place, which means a random write per modified page. On spinning disks that was catastrophic; even on SSDs it causes write amplification inside the device and limits sustained throughput. A write-heavy workload spends its life doing random IO.

The log-structured merge tree inverts this: never modify anything in place. Buffer writes in memory, and when the buffer is full, write it out as one sorted immutable file in a single sequential pass. Writes become sequential and batched, which is the access pattern storage hardware most prefers.

> **The three amplifications, and the law that governs them**  
> **Write amplification**: bytes physically written per logical byte, driven by compaction re-writing data repeatedly. **Read amplification**: files that must be consulted per lookup, since a key may live in any of them. **Space amplification**: bytes stored per logical byte, because obsolete versions persist until compaction. You cannot minimise all three — every compaction strategy is a choice about which one to sacrifice.

**Mental model**

Think of it as an append-only log with periodic housekeeping. Writes go to a small sorted structure in memory; when it fills, it becomes an immutable sorted file on disk. Over time these files accumulate, so background compaction merges them, discarding overwritten and deleted entries.

1. **Write-ahead log** — Every write is appended to a WAL first for durability, so a crash before flush loses nothing.
2. **Memtable** — An in-memory sorted structure (skip list or balanced tree) absorbing writes. Reads check it first.
3. **SSTable** — A flushed, immutable, sorted file with a sparse index and usually a Bloom filter.
4. **Levels** — Files organised in levels of increasing size; compaction merges files from one level into the next.
5. **Tombstone** — A delete is a write — a marker that must survive until every older copy of the key has been compacted away.

> **Deletes make things bigger, not smaller**  
> In an LSM tree a delete appends a tombstone. The data is not removed until compaction merges past every file containing older versions of that key, and the tombstone itself must be retained until then. A delete-heavy workload therefore *increases* storage and slows reads, which is the opposite of the intuition and the cause of many production surprises.

**How it works**

**The write and read paths**

```text
WRITE
  append to WAL (sequential, durable)
  insert into memtable (in memory, sorted)
  memtable full -> flush as immutable SSTable (sequential write)
  -> every write is sequential. No read required. Very fast.

READ  key = "user:42"
  1  memtable                      (in memory)
  2  immutable memtables awaiting flush
  3  L0 files  (overlapping ranges - check ALL of them)
  4  L1..Ln    (non-overlapping - binary search picks ONE per level)
  stop at the first hit; newest wins

  Bloom filter per file answers "definitely not here" in O(1),
  turning most of those checks into a memory lookup with no IO.

COMPACTION (background)
  merge files, keep only the newest version of each key,
  drop tombstones once no older version can exist,
  write the result to the next level.
```

1. **Bloom filters make reads viable** — Without them, a point lookup would touch every file. With a ~1% false-positive rate, almost all files are excluded with no IO, so a read typically costs one real file access.
2. **Leveled compaction minimises space and read amplification** — Each level is non-overlapping, so at most one file per level is consulted. The cost is high write amplification — data is re-written roughly once per level, often 10–30× overall.
3. **Tiered compaction minimises write amplification** — Files accumulate within a level and merge less often. Writes are cheaper but reads must check more files and space amplification is higher.
4. **Watch for write stalls** — If flushes or compaction fall behind ingest, the engine throttles or blocks writers. Sustained write throughput is bounded by compaction throughput, not by the write path.
5. **Avoid delete-heavy access patterns** — Queue-like tables in an LSM store accumulate tombstones and degrade steadily. Prefer time-partitioned tables you can drop whole.
6. **Range scans read across levels** — A range query merges from every level, which is more expensive than a B-tree's linked leaf pages — LSM is write-optimised, not scan-optimised.

**Compaction strategies compared**

```text
LEVELED (RocksDB default, Cassandra LCS)
  L0: overlapping, small
  L1: 10x L0, non-overlapping
  L2: 10x L1, non-overlapping ...
  write amp:  HIGH (~10-30x)
  read amp:   LOW  (~1 file per level)
  space amp:  LOW  (~1.1x)
  use: read-heavy, space-constrained

TIERED / SIZE-TIERED (Cassandra STCS default)
  merge N similarly-sized files into one larger file
  write amp:  LOW  (~4-10x)
  read amp:   HIGH (many overlapping files)
  space amp:  HIGH (up to 2x during compaction)
  use: write-heavy, space is cheap

TIME-WINDOWED (Cassandra TWCS)
  compact only within a time window; expire whole windows
  use: time-series with TTL - avoids tombstone accumulation
       entirely by dropping whole files
```

> **Compaction is a capacity planning item**  
> Compaction consumes IO, CPU and disk headroom continuously, and it competes with foreground traffic. Plan for it: reserve roughly 50% disk headroom for size-tiered strategies, expect background IO to be a significant fraction of total, and monitor compaction backlog as a leading indicator — a growing backlog predicts write stalls before they happen.

**Worked example**

An event-ingestion service writing 200,000 events per second. Why LSM, and what it costs.

**Sizing the write path and its amplification**

```text
WORKLOAD
  200k writes/s x 500 B = 100 MB/s logical

B-TREE ESTIMATE
  each write dirties a random page
  200k random page writes/s -> far beyond a single node
  -> would need heavy sharding purely for write IO

LSM
  WAL append:        100 MB/s sequential
  memtable flush:    100 MB/s sequential
  compaction:        100 MB/s x write_amp

WITH LEVELED (amp ~15x)
  physical writes = 100 x 15 = 1.5 GB/s   <- device-limited
WITH SIZE-TIERED (amp ~6x)
  physical writes = 600 MB/s              <- feasible
  cost: reads check more files; ~2x space during compaction

DECISION
  write-heavy, read-by-recent, TTL-based retention
  -> time-windowed compaction:
     writes compact only within their window,
     whole windows expire and are DROPPED (no tombstones)
     write amp approaches ~1-2x
```

| Metric | Value | Note |
|---|---|---|
| Logical | 100 MB/s | 200k events |
| Leveled | 1.5 GB/s | too much |
| Size-tiered | 600 MB/s | workable |
| Time-windowed | ~200 MB/s | **best fit** |

> **Match the compaction strategy to the data's lifecycle**  
> The largest win here came not from tuning but from recognising that the data is time-ordered with TTL-based retention. Time-windowed compaction lets whole files expire and be deleted rather than compacted, which eliminates both the write amplification and the tombstone problem simultaneously. Compaction strategy should follow from the access and retention pattern, not from a default.

**When to use it**

- **Write-heavy workloads**: event ingestion, metrics, logs, messaging, IoT.
- **Time-series data with TTL**, where time-windowed compaction makes retention nearly free.
- **Key-value and wide-column stores** where the access pattern is point lookups and partition-scoped ranges.
- **Embedded storage engines** for stateful services holding local state.
- **Workloads on flash**, where sequential writes reduce device-level wear and amplification.

**When to avoid it**

- **Do not use it for read-dominated workloads with large scans** — B-trees keep data physically ordered and scan better.
- **Do not use it for delete- or update-heavy workloads** without a time-based expiry strategy; tombstones accumulate.
- **Do not ignore compaction capacity**; sustained write throughput is bounded by it, not by the write path.
- **Do not run without disk headroom** — size-tiered compaction can temporarily double space usage.
- **Do not treat an LSM table as a queue**, which is the archetypal tombstone disaster.

**Advantages**

- **Sequential writes** with no read-modify-write, giving very high ingest throughput.
- **Excellent compression**, because SSTables are sorted, immutable and written in bulk.
- **Crash recovery is simple** — replay the WAL into a fresh memtable.
- **Immutable files simplify backup, replication and caching**, since files never change after they are written.
- **Tunable amplification** by compaction strategy, so the engine can be matched to the workload.

**Disadvantages**

- **Read amplification**: a lookup may consult several files, mitigated but not eliminated by Bloom filters.
- **Write amplification from compaction**, typically 10–30× for leveled strategies.
- **Space amplification**: obsolete versions persist until compaction, up to 2× transiently.
- **Background compaction competes** with foreground traffic for IO and CPU, causing latency variance.
- **Write stalls** when compaction falls behind, which is a cliff rather than a gradual degradation.
- **Tombstones make deletes costly**, degrading reads and increasing space until compaction clears them.

**Trade-offs**

**B-tree versus LSM**

|  | B-tree | LSM tree |
|---|---|---|
| Write pattern | Random, in place | Sequential, append only |
| Write amplification | ~2–4× | ~4–30× depending on strategy |
| Read amplification | ~1 (index descent) | Several files; Bloom filters help |
| Space amplification | Fragmentation, ~1.2× | Obsolete versions, up to ~2× |
| Range scan | Sequential leaf pages | Merge across levels |
| Deletes | Remove in place | Tombstone, cleaned by compaction |
| Latency profile | Predictable | Variable — compaction interference |
| Best fit | Read-heavy, scan-heavy, mixed | Write-heavy, ingest, time series |

**How it fails**

**LSM failure modes**

| Symptom | Cause | Fix |
|---|---|---|
| Write stalls / throttling | Compaction cannot keep up with ingest | More compaction threads, faster storage, lower-amplification strategy |
| Reads slow down steadily | Tombstone accumulation from deletes or TTL | Time-windowed compaction; drop partitions instead of rows |
| Disk fills during compaction | Insufficient headroom for size-tiered merges | Keep ~50% free; use leveled compaction |
| Latency spikes at intervals | Large compactions competing for IO | Rate-limit compaction; smaller files; more, smaller merges |
| Point lookups slower than expected | Bloom filters undersized or absent | Increase bits per key; verify filter hit rates |
| Deleted data reappears | Tombstone expired before an older copy was compacted | Respect the grace period; run repair before it elapses |
| Space never decreases after deletes | Data only removed at compaction | Trigger compaction; use TTL with time-windowed strategy |

**Limits**

> **Operating numbers**
>
> - **Write amplification**: leveled ~10–30×, size-tiered ~4–10×, time-windowed near 1–2× for append-only TTL data.
> - **Bloom filter**: ~10 bits per key gives roughly a 1% false-positive rate.
> - **Space headroom**: reserve ~50% free disk for size-tiered compaction; less for leveled.
> - **Compaction backlog** is the leading indicator of write stalls — alert on it, not on write latency.
> - **Sustained write ceiling** = device write bandwidth ÷ write amplification.

**Alternatives**

| Engine | Best for | Trade |
|---|---|---|
| LSM tree | Write-heavy ingest | Read and space amplification; compaction load |
| B-tree | Read-heavy, scan-heavy, mixed | Random write cost |
| Fractal / B-epsilon tree | Middle ground with buffered writes | Less common; fewer implementations |
| Append-only log with external index | Pure event capture | Index must be maintained separately |
| Columnar files with compaction | Analytical workloads | Not for point lookups |

**In real systems**

- **RocksDB and LevelDB** are the reference implementations and are embedded in a large fraction of modern storage systems.
- **Cassandra and ScyllaDB** expose compaction strategy per table, making the amplification trade an explicit modelling decision.
- **Kafka's log segments** are conceptually similar: immutable sequential files with background cleanup, though the access pattern is sequential rather than keyed.
- **Time-windowed compaction** exists specifically because time-series with TTL is a common workload that leveled and tiered strategies both handle badly.
- **Most cloud key-value and wide-column services** use LSM engines internally, which is why their write throughput scales so smoothly and their delete semantics are so often surprising.

**Common mistakes**

- **Using an LSM table as a queue**, accumulating tombstones until reads collapse.
- **Sizing write capacity from logical throughput**, ignoring write amplification.
- **Leaving the default compaction strategy** on time-series data with TTL.
- **Running without disk headroom** for size-tiered compaction.
- **Alerting on write latency instead of compaction backlog**, so stalls arrive unannounced.
- **Expecting disk usage to drop immediately after deletes.**
- **Choosing an LSM engine for a scan-heavy analytical workload.**

**The staff-level view**

The LSM decision is made once, in the choice of datastore, but the compaction decision is made per table and is frequently left at a default that does not fit.

- **Match compaction strategy to data lifecycle explicitly.** Time-ordered data with TTL should almost never use a general-purpose strategy.
- **Treat delete-heavy access patterns as a design defect** in LSM stores, not as a tuning problem. Prefer dropping time-partitioned tables whole.
- **Monitor compaction backlog as a first-class SLI**, because write stalls are a cliff and the backlog is the only warning.
- **Budget for compaction in capacity planning**: IO, CPU and disk headroom are all consumed continuously by it.
- **Know the write-amplification factor for your configuration**, because sustained write ceiling is device bandwidth divided by that number — and teams routinely size for logical throughput instead.

**Go deeper**

An LSM tree never updates in place. Writes append to a write-ahead log and land in an in-memory sorted memtable; when it fills, it is flushed as an immutable sorted file. Background compaction merges files, keeping the newest version of each key and discarding obsolete ones. Every write is sequential with no read first, which is why ingest throughput is so much higher than a B-tree's random-page updates.

The costs are three amplifications you cannot all minimise. Write amplification comes from compaction rewriting data — 10–30× for leveled strategies, 4–10× for size-tiered. Read amplification comes from a key potentially living in any file, mitigated by per-file Bloom filters that exclude almost all of them with no IO. Space amplification comes from obsolete versions persisting until compacted, transiently up to 2×. Compaction strategy is the knob that chooses which to sacrifice.

Two practical consequences dominate. Sustained write capacity equals device bandwidth divided by write amplification, so sizing from logical throughput can be off by an order of magnitude. And deletes are writes: a tombstone must persist until compaction merges past every older copy, so delete-heavy workloads grow and slow down — which is why time-windowed compaction, where whole expired files are dropped rather than compacted, is the right choice for TTL'd time-series data.

The log-structured merge tree is the answer to a hardware-shaped question: random writes are expensive and sequential writes are cheap, so restructure storage so that all writes are sequential and pay for it later, in the background.

**The write path.** A write appends to a write-ahead log for durability and inserts into an in-memory sorted structure. When that fills it is flushed as an immutable sorted file — one sequential pass, no read-modify-write, no random IO. This is why LSM engines sustain write rates that in-place structures cannot approach, and why they compress well: files are written in bulk, sorted, and never modified.

**The read path and Bloom filters.** Because a key may exist in the memtable or in any file, a lookup must consult them in newest-first order. Without help this would be prohibitive; a per-file Bloom filter answers “definitely not here” in constant time from memory, so with roughly ten bits per key and a 1% false-positive rate, nearly every file is excluded without touching disk and a typical read costs one real file access. Undersized or missing filters are a frequent cause of inexplicably slow point lookups.

**The three amplifications.** Write amplification is physical bytes written per logical byte, produced by compaction rewriting data as it moves between levels. Read amplification is the number of files consulted per lookup. Space amplification is stored bytes per logical byte, because superseded versions survive until compacted. These trade against each other and the compaction strategy is the choice. Leveled compaction keeps each level non-overlapping so reads touch about one file per level and space stays near 1.1×, at 10–30× write amplification. Size-tiered merges similarly sized files less frequently, giving 4–10× write amplification but more files to read and transient space usage up to 2×. Time-windowed compaction restricts merging to within a time window so entire expired windows can be deleted rather than compacted, driving write amplification toward 1–2× for append-only TTL data.

**Deletes are the counter-intuitive part.** A delete appends a tombstone, and the underlying data is removed only when compaction merges past every file that might hold an older version — with the tombstone itself retained until then, so a stale replica cannot resurrect the value. Consequently a delete-heavy workload increases storage and degrades reads, and using an LSM table as a queue is the archetypal disaster. The right pattern is time-partitioned data where retention drops whole files.

**Capacity and operations.** Sustained write throughput is device write bandwidth divided by write amplification, which means sizing from logical throughput can be wrong by more than an order of magnitude. Compaction consumes IO, CPU and disk headroom continuously and competes with foreground traffic, giving LSM engines latency variance that B-trees lack. Most importantly, when compaction falls behind, the engine throttles or blocks writers — a cliff, not a slope — so the compaction backlog is the metric to alert on, since it rises well before write latency does. Choosing an LSM engine therefore commits the organisation to understanding compaction strategy, amplification and tombstone behaviour before the first incident rather than during it.

**Prove it — interview questions**

1. **[Basic] Why are LSM writes faster than B-tree writes?**

   <details><summary>Model answer</summary>

   Because they avoid random in-place updates. An LSM write appends to a write-ahead log and inserts into an in-memory sorted structure; when that fills, it is written out as one immutable sorted file in a single sequential pass. No page is read before being written and no random IO is involved. A B-tree, by contrast, must locate and modify the correct page, which is a random read followed by a random write for every modification.

   </details>

2. **[Basic] What is a tombstone and why does it matter?**

   <details><summary>Model answer</summary>

   A delete in an LSM tree is itself a write: it appends a marker saying the key is removed. The actual data stays until compaction merges past every file that could contain an older version, and the tombstone must be retained until then so the deletion is not undone by an older copy. This means deletes temporarily increase storage and slow reads, which is the opposite of the intuition, and it is why delete-heavy workloads degrade steadily in LSM stores.

   </details>

3. **[Senior] Explain the three amplification factors and how compaction strategy trades between them.**

   <details><summary>Model answer</summary>

   Write amplification is physical bytes written per logical byte, driven by compaction re-writing data as it moves through levels. Read amplification is how many files a lookup must consult, since a key may be in any of them. Space amplification is stored bytes per logical byte, because obsolete versions persist until compacted. Leveled compaction keeps levels non-overlapping, so reads check about one file per level and space stays near 1.1× — but data is rewritten roughly once per level, giving 10–30× write amplification. Size-tiered merges similarly sized files less often, cutting write amplification to 4–10× but increasing both read and space amplification, with transient usage up to 2×. You choose which to sacrifice based on whether the workload is read-heavy, write-heavy, or space-constrained.

   </details>

4. **[Senior] How do Bloom filters make LSM reads practical?**

   <details><summary>Model answer</summary>

   Without them, a point lookup would have to check every file, because a key could be in any of them. A Bloom filter per file answers “definitely not present” in constant time from memory, with a tunable false-positive rate — around 1% at ten bits per key. That means almost every file is excluded with no disk access, and a typical lookup costs one real file read. They are the reason read amplification is an inconvenience rather than a disqualifier, and undersized filters are a common cause of unexpectedly slow reads.

   </details>

5. **[Staff] An LSM-backed service is hitting write stalls. Walk through your diagnosis.**

   <details><summary>Model answer</summary>

   Write stalls mean compaction is not keeping up with ingest, so the engine is throttling writers to prevent unbounded file accumulation. First I would confirm with the compaction backlog metric, which should have been rising for some time before the stall — that is the leading indicator and stalls are a cliff. Then I would establish the real ceiling: sustained write capacity is device write bandwidth divided by write amplification, and teams routinely size for logical throughput, so the system may simply have been provisioned two orders of magnitude short. From there the levers are, in order of preference: change compaction strategy to reduce amplification, which is free if the data's lifecycle suits it — time-ordered TTL data on a leveled strategy is a common misconfiguration; increase compaction parallelism if CPU is available; move to faster storage; or reduce ingest. I would also check whether deletes or TTL are generating tombstones that compaction must repeatedly process, since that is amplification doing work that produces no value.

   </details>

6. **[Principal] How would you decide between B-tree and LSM storage engines for a new platform?**

   <details><summary>Model answer</summary>

   By the workload's write-to-read ratio and its access shape, but I would weight operational predictability heavily, because that is what teams underestimate. LSM gives much higher write throughput and better compression, but it introduces background compaction that competes with foreground traffic, so latency has variance that B-trees do not, and it introduces a cliff — write stalls — rather than gradual degradation. It also makes deletes expensive in a way that surprises every team that meets it for the first time. So for ingest-dominated workloads with time-ordered data and TTL retention, LSM with time-windowed compaction is clearly right and the operational model is simple. For mixed transactional workloads with meaningful scan traffic and in-place updates, a B-tree engine is easier to reason about and to operate, and the write ceiling is rarely the binding constraint. I would also consider who will be on call: an LSM engine requires understanding compaction strategy, amplification and tombstone behaviour to operate well, and that knowledge has to exist somewhere in the organisation before the first incident, not after.

   </details>

---

### Write-ahead logging

*Append the intent to a durable log before touching data pages, so a crash at any instant leaves a recoverable, consistent database.*

**Flow:** `Transaction` → `WAL flush` → `Commit acknowledgment` → `Page flush` → `Recovery replay`

> **The 30-second version**  
> Write the intent to a durable sequential log before touching data pages. That ordering makes arbitrary multi-page changes atomic, makes commits one sequential write, and powers replication and point-in-time recovery.

**The problem**

A transaction modifies five pages scattered across a large data file. The process crashes after three of them have reached disk. On restart, the database contains half of a transaction — some rows updated, others not — with no way to tell which.

Making the five page writes atomic is impossible: the storage device guarantees atomicity only for a single sector. So instead of trying to make the data writes atomic, you make a *record of intent* atomic, and write it first.

> **The write-ahead rule**  
> **The log record describing a change must be durable before the changed page is written.** That single ordering constraint is what makes recovery possible: if the page made it to disk, the log describing it certainly did, so recovery can always determine what state the page should be in.

**Mental model**

Think of a builder who writes every instruction in a numbered notebook before performing it. If the work is interrupted, anyone can read the notebook and either finish the job or undo the half-finished parts. The notebook is append-only and never rewritten, so writing to it is fast and sequential.

1. **Log record** — A description of a change — which page, what it was, what it becomes — stamped with a monotonically increasing sequence number (LSN).
2. **Commit record** — Marks a transaction as durable. Once this record is on disk, the transaction is committed, regardless of whether any data page has been written.
3. **Page LSN** — Each data page records the LSN of the last change applied to it, so recovery knows whether a logged change is already reflected.
4. **Checkpoint** — A marker saying “everything before this point has been applied to data pages,” which bounds how much log recovery must replay.
5. **Recovery** — Replay the log forward from the last checkpoint (redo), then undo any transaction that never committed.

> **Why this makes commits fast, not slow**  
> Without a WAL, committing would require flushing every modified page — several random writes. With a WAL, committing requires flushing one sequential append. The data pages can be written later, lazily, in whatever order is most efficient. The log turns many random synchronous writes into one sequential synchronous write, which is why WAL improves throughput as well as durability.

**How it works**

**Commit path and recovery**

```text
COMMIT PATH
  1  modify page in the buffer pool (memory only)
  2  append log record(s) describing the change
  3  append COMMIT record
  4  fsync the log            <- the ONLY synchronous disk write
  5  acknowledge the client
  ... later, at leisure ...
  6  background writer flushes dirty pages to the data file
  7  checkpoint records that pages up to LSN N are durable

CRASH RECOVERY (ARIES-style)
  ANALYSIS  scan from the last checkpoint; find dirty pages and
            transactions that were in flight
  REDO      replay every logged change whose LSN > page LSN
            (repeating history, including uncommitted work)
  UNDO      roll back transactions with no COMMIT record,
            writing compensation log records as it goes

Result: committed work is present, uncommitted work is gone,
        regardless of when the crash happened.
```

1. **Group commit amortises fsync** — Flushing the log costs a device round trip. Batching many transactions into one flush turns N fsyncs into one, which is the difference between hundreds and tens of thousands of commits per second.
2. **Redo must be idempotent** — Recovery may replay a change already present in a page, so every change is applied only if the page LSN is older than the log record's LSN. Without this, recovery could corrupt what it is repairing.
3. **Undo needs compensation records** — Rolling back writes its own log records, so a crash *during* recovery is also recoverable. This is what makes ARIES robust rather than merely plausible.
4. **Checkpoints bound recovery time** — More frequent checkpoints mean shorter recovery and more background IO. This is the direct knob on your recovery time objective.
5. **The log is also the replication stream** — Followers apply the same log records. This is why WAL, physical replication and point-in-time recovery are all the same mechanism viewed differently.
6. **`fsync` must actually reach the device** — Storage layers that lie about durability — write caches without power protection, some virtualised disks — silently break the guarantee. Verify it rather than assuming it.

**Durability settings and what they actually cost**

```text
SYNCHRONOUS COMMIT (default, safe)
  fsync the WAL before acknowledging
  durability: committed means committed
  cost: one device round trip per commit (mitigated by group commit)

ASYNCHRONOUS COMMIT
  acknowledge before the WAL is flushed
  durability: a crash may lose the last N milliseconds of commits
  gain: large throughput increase for small transactions
  use: only where losing recent commits is acceptable

NO WAL / UNLOGGED TABLES
  durability: table is truncated after a crash
  use: derived data, caches, staging tables you can rebuild

SYNCHRONOUS REPLICATION
  fsync locally AND wait for a replica to acknowledge
  durability: survives losing the whole primary node
  cost: adds network round trip to every commit
```

> **“Committed” is a claim about a specific failure domain**  
> A local fsync protects against process and OS crashes. It does not protect against that machine's disk failing, or that availability zone disappearing. If a commit must survive node loss, it must be acknowledged by a replica before the client is told it succeeded — and that is a latency cost you accept explicitly, not a setting you discover afterwards.

**Worked example**

Sizing the commit path for a payments service where durability is non-negotiable.

**From durability requirement to configuration**

```text
REQUIREMENT
  a captured payment must never be lost, even if the primary
  node is destroyed        -> RPO = 0

CONFIGURATION
  synchronous_commit = remote_apply (or equivalent)
  one synchronous replica in another availability zone
  additional async replicas for read scaling / DR

LATENCY BUDGET PER COMMIT
  local WAL fsync              ~0.1-1 ms  (NVMe with power-loss
                                           protection)
  network to AZ replica        ~0.5-1 ms  round trip
  replica WAL fsync            ~0.1-1 ms
  total added latency          ~1-3 ms

THROUGHPUT
  without group commit: 1 commit per round trip  -> ~500/s
  with group commit:    batch 50 commits per flush -> ~25,000/s
  -> group commit is what makes synchronous durability viable

WHAT YOU STILL MUST DECIDE
  what happens if the sync replica is unavailable:
    block writes (safe, unavailable) or
    degrade to async (available, RPO > 0)
  -> this is a business decision; write it down in advance.
```

| Metric | Value | Note |
|---|---|---|
| Local fsync | ~0.1–1 ms | process crash safe |
| + sync replica | ~1–3 ms | **node loss safe** |
| Group commit | ~25k/s | amortises fsync |
| Replica down | policy needed | decide in advance |

> **The question nobody answers until the incident**  
> Synchronous replication is easy to configure and hard to operate, because it forces a choice you must make *before* the replica fails: block writes and be unavailable, or fall back to asynchronous and accept a non-zero recovery point. Systems that leave this undecided make the choice by accident, under pressure, in the worst possible way.

**When to use it**

- **Every durable database** — this is not optional infrastructure, it is how durability is implemented.
- **Crash recovery**, giving bounded recovery time from the last checkpoint.
- **Physical replication**, where followers apply the same log records.
- **Point-in-time recovery**, replaying the log to any moment between backups.
- **Change data capture**, where the log is read to produce a downstream event stream.

**When to avoid it**

- **Do not disable synchronous commit** for data whose loss matters, however tempting the throughput gain.
- **Do not assume local durability survives node loss** — that requires replica acknowledgement.
- **Do not let the log fill the disk.** A full WAL volume stops the database entirely, and this is a common self-inflicted outage.
- **Do not retain WAL indefinitely for a disconnected replica**; bound it and let the replica rebuild instead.
- **Do not trust `fsync` without verifying it**, especially on virtualised or consumer storage.

**Advantages**

- **Atomicity and durability for arbitrary multi-page changes**, which the hardware cannot provide directly.
- **Commits become one sequential write** instead of many random ones, improving throughput.
- **Bounded recovery time** through checkpointing.
- **One mechanism serves durability, replication, PITR and CDC**, which is why it is so central.
- **Data page writes can be deferred and reordered** for efficiency without compromising correctness.

**Disadvantages**

- **Every commit pays an fsync**, which is a device round trip unless amortised by group commit.
- **Write amplification**: data is written twice, once to the log and once to the data file.
- **Log space must be managed** — retention for replicas and PITR competes with disk capacity.
- **Recovery time grows with checkpoint interval**, trading steady-state IO against RTO.
- **Synchronous replication adds a network round trip** to the critical path of every commit.

**Trade-offs**

**Durability levels**

| Setting | Survives | Cost | RPO |
|---|---|---|---|
| No WAL / unlogged | Nothing; table truncated on crash | None | All data |
| Async commit | Nothing recent | None | Milliseconds of commits |
| Sync commit (local fsync) | Process and OS crash | 1 device round trip | 0 for that node |
| Sync commit + sync replica | Node and disk loss | +1 network round trip | 0 |
| Sync replica in another region | Region loss | +cross-region latency | 0 |

Each row down this table adds latency to every commit and removes a class of data loss. The right choice is per dataset, not per system — session data and payment records rarely deserve the same guarantee.

**How it fails**

**WAL-related failures**

| Failure | Cause | Fix |
|---|---|---|
| Database refuses writes | WAL volume full | Separate volume with headroom; alert on WAL size; bound replication slot retention |
| Slow recovery after a crash | Long checkpoint interval; large redo backlog | More frequent checkpoints; measure recovery time deliberately |
| Commit latency spikes | fsync contention; shared volume with data files | Dedicated WAL device; group commit; check storage queue depth |
| Data loss despite “committed” | fsync lied, or async commit enabled | Verify durability end to end; use power-loss-protected storage |
| Replica falls permanently behind | Replay slower than generation | Faster replica IO; reduce write rate; parallel apply if available |
| Writes blocked when a replica dies | Synchronous replication with no fallback policy | Decide and configure the policy in advance; quorum of two replicas |
| Disk fills from retained WAL | Inactive replication slot holding log segments | Monitor slot lag; drop abandoned slots; set retention limits |

> **The replication slot disk-fill outage**  
> A replica goes offline and its replication slot keeps the primary retaining WAL segments so it can catch up later. If nobody notices, the WAL volume fills and the primary stops accepting writes — a complete outage caused by a *replica* failing. Always bound slot retention and alert on WAL volume growth, not just on replica lag.

**Limits**

> **Numbers to plan with**
>
> - **fsync latency**: ~0.1–1 ms on NVMe with power-loss protection; several ms on network-attached storage.
> - **Group commit** can raise commit throughput by one to two orders of magnitude for small transactions.
> - **Write amplification**: roughly 2×, since every change is written to both log and data file.
> - **Checkpoint interval** directly sets the redo backlog and therefore recovery time — measure it, do not assume.
> - **Cross-AZ round trip** ~0.5–1 ms; cross-region 50–150 ms, which is why synchronous cross-region commit is rarely acceptable.

**Alternatives**

| Technique | Provides | Trade |
|---|---|---|
| Write-ahead log | Atomicity, durability, replication, PITR | fsync per commit; 2× write amplification |
| Shadow paging / copy-on-write | Atomic switch of a page tree | Fragmentation; poor locality |
| Log-structured storage | Log *is* the data | Compaction cost; read amplification |
| Battery-backed write cache | Makes fsync nearly free | Hardware dependency; fails silently if the battery dies |
| Replicated consensus log | Durability across nodes | Consensus latency on every write |

**In real systems**

- **ARIES** is the algorithm nearly every relational database implements — repeating history during redo, then undoing with compensation records.
- **PostgreSQL's WAL** serves durability, streaming replication, point-in-time recovery and logical decoding for CDC from one mechanism.
- **MySQL InnoDB's redo log plus doublewrite buffer** additionally guards against torn pages, where a partial page write would otherwise be unrecoverable.
- **Kafka** applies the same principle at the system level: an append-only log is the primary artifact, with consumers as deferred appliers.
- **Debezium and similar CDC tools** read the database's replication log directly, which is why they capture every change without polling.

**Common mistakes**

- **Enabling asynchronous commit for throughput** on data whose loss matters.
- **Assuming a local fsync protects against losing the node.**
- **Unbounded replication slot retention**, filling the WAL volume and stopping writes.
- **Never testing recovery**, so checkpoint configuration is an untested guess about RTO.
- **Sharing a volume between WAL and data files**, so fsync contends with page flushes.
- **Trusting storage that does not honour fsync**, especially in virtualised environments.
- **Leaving the synchronous-replica failure policy undefined** until the replica actually fails.

**The staff-level view**

WAL configuration is where durability promises become concrete, and where the gap between what a team believes and what the system guarantees is usually widest.

- **State the recovery point objective per dataset, then configure to match.** Payments and session data should not share a durability setting.
- **Verify durability rather than assuming it.** Power-cut testing, or at minimum confirming that the storage layer honours fsync, catches a class of silent data-loss bugs.
- **Decide the synchronous-replica failure policy in advance** — block writes or degrade to async — and write it into the runbook. Deciding during an incident goes badly.
- **Monitor WAL volume and replication slot retention as first-class alerts.** A replica failing should never be able to take down the primary.
- **Measure recovery time in practice**, by actually killing a node under load. Checkpoint settings are a recovery-time decision and are rarely validated.

**Go deeper**

The write-ahead rule is that a log record describing a change must be durable before the changed page is written. That single ordering constraint makes recovery deterministic: any page that reached disk has its description in the log, so recovery can always reconstruct the correct state. A commit therefore needs only one sequential fsync of the log rather than several random page flushes — which is why WAL improves throughput as well as guaranteeing durability.

Recovery follows the ARIES pattern: analyse from the last checkpoint to find dirty pages and in-flight transactions, redo every logged change newer than the page's recorded sequence number (repeating history idempotently), then undo uncommitted transactions while writing compensation records so that a crash during recovery is itself recoverable. Checkpoint frequency is the direct knob on recovery time.

The critical operational point is that “committed” is a claim about a specific failure domain. A local fsync survives a process or OS crash; surviving node or zone loss requires a replica to acknowledge before the client does. That adds a network round trip per commit, made viable by group commit — and it forces a decision teams usually defer: when the synchronous replica is unavailable, do you block writes or degrade to asynchronous? Decide that before the incident, not during it.

Write-ahead logging exists because storage hardware guarantees atomicity only for a single sector, while transactions routinely modify many pages. Rather than attempting atomic multi-page writes, the database makes a *record of intent* durable first, and derives everything else from that.

**The rule and its consequences.** The log record describing a change must reach durable storage before the changed page does. Because of this ordering, any page found on disk after a crash necessarily has its describing record in the log, so recovery can always determine the correct state. A second consequence is performance: committing requires flushing only the log — one sequential append — while dirty pages are written later, lazily, in whatever order is most efficient. The WAL converts many random synchronous writes into one sequential one, and group commit amortises even that across many transactions, which is the difference between hundreds and tens of thousands of commits per second.

**Recovery.** ARIES structures it as analysis, redo, undo. Analysis scans from the last checkpoint to identify dirty pages and in-flight transactions. Redo repeats history, replaying every change whose log sequence number exceeds the target page's recorded sequence number — applying even uncommitted work, which keeps the logic uniform and makes replay idempotent. Undo then rolls back transactions with no commit record, writing compensation log records as it proceeds so that a crash during recovery is itself recoverable. Checkpoint frequency bounds the redo backlog and therefore recovery time, trading steady-state background IO against RTO.

**One mechanism, four products.** The same log provides crash recovery, physical replication (followers apply the same records), point-in-time recovery (replay to any moment between backups), and change data capture (downstream consumers decode it into events). This is why the WAL is so central and why its configuration touches so many concerns that look unrelated.

**Durability is a claim about a failure domain.** Asynchronous commit acknowledges before flushing and loses the last few milliseconds on a crash. Synchronous local commit survives process and OS crashes on that machine and nothing more. Surviving node or disk loss requires a replica to acknowledge before the client is told the commit succeeded, adding a network round trip — about one to three milliseconds within a region, and 50 to 150 across regions, which is why synchronous cross-region commit is rarely acceptable. These are per-dataset choices: session data and payment records should not share a setting.

**Where it goes wrong operationally.** Two failures recur. First, storage that does not honour fsync — some virtualised or consumer devices — silently breaks the guarantee, so durability must be verified by power-cut or hard-kill testing rather than assumed. Second, an offline replica whose replication slot causes the primary to retain log segments indefinitely fills the log volume and stops the primary accepting writes: a redundancy mechanism converted into a total outage by a *replica* failing. Bound slot retention and alert on log volume growth, not merely on replica lag. And decide the synchronous-replica failure policy — block writes or degrade to asynchronous — in advance, because it is a business trade about availability versus data loss, and making it under pressure during an incident consistently goes badly.

**Prove it — interview questions**

1. **[Basic] What is the write-ahead rule?**

   <details><summary>Model answer</summary>

   The log record describing a change must be durable on disk before the changed data page is written. That ordering is what makes recovery possible: if a page reached disk, the log entry describing it certainly did, so recovery can always determine what the page should contain. It also means a commit requires flushing only the log, not the modified pages, which is why the WAL improves throughput as well as durability.

   </details>

2. **[Basic] Why is committing faster with a WAL than without one?**

   <details><summary>Model answer</summary>

   Without a log, committing would require flushing every modified page — several random writes scattered across the data file. With a log, committing requires one sequential append and a single fsync, and the data pages can be written later in whatever order is most efficient. Group commit amortises even that fsync across many transactions, so the log converts many random synchronous writes into a fraction of one.

   </details>

3. **[Senior] Walk through crash recovery.**

   <details><summary>Model answer</summary>

   Three phases, in the ARIES model. Analysis scans forward from the last checkpoint to determine which pages were dirty and which transactions were in flight. Redo then replays every logged change whose sequence number is newer than the page's recorded sequence number — repeating history, including work by transactions that never committed, which keeps the logic simple and idempotent. Undo finally rolls back those uncommitted transactions, writing compensation log records as it does so, which is what makes a crash during recovery itself recoverable. The result is that committed work is present and uncommitted work is gone, no matter when the crash occurred.

   </details>

4. **[Senior] What does “committed” actually guarantee?**

   <details><summary>Model answer</summary>

   It depends entirely on the configuration, and the gap between assumption and reality is where data loss happens. A synchronous local commit guarantees survival of a process or operating system crash on that machine. It guarantees nothing about that machine's disk failing or its availability zone disappearing — for that, the commit must be acknowledged by a replica before the client is told it succeeded. And asynchronous commit guarantees nothing about the last few milliseconds at all. So “committed” is a claim about a specific failure domain, and the right question is always: which failures is this promise meant to survive?

   </details>

5. **[Staff] How do you configure durability for a payments system, and what must you decide in advance?**

   <details><summary>Model answer</summary>

   I would require zero recovery point for captured payments, which means synchronous commit locally plus acknowledgement from a replica in another availability zone before the client is told the payment succeeded — that adds roughly one to three milliseconds per commit, made viable at volume by group commit. Cross-region synchronous would add 50 to 150 milliseconds and is usually unacceptable, so region loss is handled by asynchronous replication with a documented, non-zero recovery point. The decision that must be made in advance, and that teams consistently defer, is what happens when the synchronous replica is unavailable: block writes and be unavailable, or degrade to asynchronous and accept data-loss risk. That is a business trade, it belongs in the runbook before the incident, and leaving it undecided means it gets made under pressure by whoever is on call.

   </details>

6. **[Principal] How do you make durability guarantees trustworthy across an organisation?**

   <details><summary>Model answer</summary>

   By treating them as claims that must be tested rather than settings that are configured. Three practices do most of the work. First, per-dataset recovery point and recovery time objectives, written down and mapped to concrete configuration, so nobody assumes session data and payment records carry the same guarantee. Second, actual verification: power-cut or hard-kill testing under load, both to confirm the storage stack honours fsync — virtualised and consumer storage sometimes does not, and the failure is silent — and to measure real recovery time, since checkpoint settings are an untested guess about RTO in most systems. Third, operational guardrails so the durability machinery cannot itself cause an outage: bounded replication slot retention and alerting on log volume growth, because a failed replica silently filling the primary's disk is a remarkably common way to convert a redundancy mechanism into a total outage.

   </details>

---

### MVCC snapshots

*Keep multiple versions of each row so readers see a consistent snapshot without blocking writers — at the cost of garbage that must be collected.*

**Flow:** `Writer` → `New version` → `Snapshot visibility` → `Reader` → `Version cleanup`

> **The 30-second version**  
> Writers create new row versions instead of overwriting, and each reader sees the versions committed as of its snapshot — so reads and writes never block each other, but old versions must be collected.

**The problem**

A report scans ten million rows for thirty seconds. Meanwhile, thousands of transactions are updating those rows. With traditional locking, either the report blocks every writer for thirty seconds, or the writers block the report, or the report sees a mixture of old and new data and produces a total that never existed at any moment in time.

Multi-version concurrency control removes the conflict entirely: writers create new versions instead of overwriting, and each reader sees the set of versions that existed when it started. Readers never block writers and writers never block readers.

> **What a snapshot actually is**  
> Not a copy of the data — a **point in the transaction timeline**. A snapshot records which transactions had committed at the moment it was taken. A row version is visible if the transaction that created it is in that committed set, and the transaction that deleted it is not. Visibility is a predicate evaluated per row, not a physical copy.

**Mental model**

Every row version carries the transaction id that created it and, once superseded or deleted, the transaction id that ended it. A reader holds a list of what was committed when it began, and filters versions through that list.

1. **Version** — A physical row copy tagged with creating and deleting transaction ids.
2. **Snapshot** — The set of transactions considered committed at a point in time — usually a minimum id, a maximum id, and a list of in-flight exceptions.
3. **Visibility rule** — A version is visible if its creator committed before the snapshot and its deleter did not commit before the snapshot.
4. **Garbage** — Versions no longer visible to any active snapshot. These must be removed or the table grows forever.
5. **Long transaction** — The enemy of MVCC. While it runs, nothing it might still need can be cleaned up — anywhere in the database.

> **Long-running transactions cause database-wide bloat**  
> A snapshot held for hours prevents garbage collection of *any* version created after it, across every table — not just the ones that transaction touched. An idle-in-transaction connection, a forgotten analytics query, or a replica with a long query can silently cause the primary's tables and indexes to bloat until disk fills. This is the single most common MVCC operational failure.

**How it works**

**Visibility in practice**

```text
ROW VERSIONS for id=1
  version A: value=100, xmin=10, xmax=25
  version B: value=200, xmin=25, xmax=null

READER with snapshot taken when committed = {..., 20}
  version A: created by 10 (committed)  -> candidate
             deleted by 25 (NOT yet committed at snapshot)
             -> VISIBLE, value = 100
  version B: created by 25 (not committed at snapshot)
             -> invisible

READER with snapshot committed = {..., 30}
  version A: deleted by 25 (committed) -> invisible
  version B: created by 25 (committed), not deleted -> VISIBLE

Two concurrent readers see different, individually consistent
states. Neither blocks the writer; the writer blocked neither.
```

1. **An update is an insert plus a logical delete** — The old version is marked ended, a new version is written. This is why update-heavy tables grow and why indexes must often point at both versions.
2. **Read committed takes a new snapshot per statement** — So two statements in the same transaction can see different data. This is the default in many databases and surprises people writing multi-statement logic.
3. **Repeatable read takes one snapshot per transaction** — Everything in the transaction sees one consistent moment, which is what makes consistent backups and reports possible.
4. **Writes still conflict** — MVCC removes read-write blocking, not write-write blocking. Two transactions updating the same row still serialise — one waits, or one aborts, depending on isolation level.
5. **Garbage collection is mandatory background work** — Vacuum, purge, or undo-segment cleanup must keep pace with the update rate, or space and scan cost grow without bound.
6. **Index entries are versioned too** — A dead row's index entry persists until cleanup, so index bloat follows table bloat and degrades lookups.

**Where the versions live: two designs**

```text
IN-PLACE VERSIONS (PostgreSQL)
  new version written into the table heap
  old version stays until VACUUM removes it
  + fast rollback (just abandon the new version)
  + no undo log contention
  - table and index BLOAT; vacuum must keep up
  - every update touches every index on the table

UNDO LOG (MySQL InnoDB, Oracle)
  row updated IN PLACE; the old value written to an undo log
  readers reconstruct old versions from undo
  + table stays compact
  - rollback is expensive (must apply undo)
  - long transactions grow the undo log instead
  - reconstructing old versions costs CPU on reads

SAME TRADE, DIFFERENT LOCATION:
the garbage has to live somewhere, and long transactions
prevent its removal either way.
```

> **Idle in transaction is a production hazard**  
> A connection that has begun a transaction and then waits on application logic, a network call, or user input holds a snapshot open. Monitor for it, set an `idle_in_transaction_session_timeout`, and never wrap an external API call inside a database transaction — the remote service's latency becomes your bloat.

**Worked example**

An analytics query causing a production incident, traced properly.

**How a read-only query fills the disk**

```text
TIMELINE
  09:00  analyst opens a session, runs a 4-hour report
         -> snapshot pinned at 09:00
  09:00+ OLTP traffic continues: 5,000 updates/s
         each update creates a new version; old versions
         cannot be removed because the 09:00 snapshot
         might still need them
  11:00  table size doubled; index scans slower
  12:30  autovacuum running constantly, removing nothing
  13:00  disk 95% full, write latency spiking
  13:05  incident

WHY THE REPORT'S OWN TABLES DID NOT MATTER
  the snapshot is global to the transaction. Versions in
  EVERY table are retained, including tables the report
  never reads.

FIXES
  run long reports on a replica with
    hot_standby_feedback = off, accepting query cancellation
  or snapshot-export / dedicated read window
  or set a statement timeout and chunk the report
  plus: alert on oldest transaction age, not just on disk
```

| Metric | Value | Note |
|---|---|---|
| Snapshot age | 4 hours | the actual cause |
| Bloat | 2× table size | unreclaimable |
| Vacuum | running, useless | **cannot remove** |
| Alert on | oldest xact age | leading indicator |

> **Monitor transaction age, not just disk usage**  
> Disk filling is the symptom; the oldest open transaction is the cause, and it is visible hours earlier. Every MVCC database exposes the age of its oldest snapshot, and alerting on it converts a 4 a.m. disk-full incident into a ticket about a forgotten session.

**When to use it**

- **Mixed read-write workloads**, which is nearly every transactional database — this is the default concurrency model for good reason.
- **Long analytical reads against live data**, where a consistent point-in-time view is needed without blocking writers.
- **Consistent backups**, which are snapshots taken at a transaction boundary.
- **Read replicas**, which serve consistent snapshots without coordinating with the primary.

**When to avoid it**

- **Do not hold transactions open across external calls or user think-time.**
- **Do not run multi-hour transactions on a primary** that is also taking heavy writes.
- **Do not assume MVCC prevents write conflicts** — it does not; two writers to one row still serialise.
- **Do not rely on read committed for multi-statement invariants**; each statement gets a fresh snapshot.
- **Do not let garbage collection fall behind** — tune it as deliberately as you would tune indexes.

**Advantages**

- **Readers never block writers and writers never block readers**, which removes the dominant source of contention in mixed workloads.
- **Consistent point-in-time reads** without locking, making reports and backups safe against live traffic.
- **Read-only transactions need no locks at all**, so read scaling is nearly free.
- **Rollback is cheap** in in-place-version designs, since the new version is simply abandoned.
- **Natural fit for replicas**, which apply changes and serve snapshots independently.

**Disadvantages**

- **Storage overhead from multiple versions**, sometimes several times the logical data size.
- **Garbage collection is mandatory ongoing work** competing with foreground traffic.
- **Long transactions cause database-wide bloat**, not just local impact.
- **Index bloat follows table bloat**, degrading lookups until cleanup runs.
- **Write-write conflicts remain**, and at serialisable isolation they surface as retryable aborts the application must handle.
- **Undo-based designs pay CPU on reads** to reconstruct older versions.

**Trade-offs**

**Isolation levels under MVCC**

| Level | Snapshot | Prevents | Still possible |
|---|---|---|---|
| Read committed | New per statement | Dirty reads | Non-repeatable reads, phantoms |
| Repeatable read / snapshot | One per transaction | Dirty and non-repeatable reads | Write skew (in snapshot isolation) |
| Serializable (SSI) | One per transaction + conflict tracking | All anomalies | Retryable serialisation failures |

Snapshot isolation is not serialisable: two transactions can each read a consistent state, each write something the other did not see, and together violate an invariant neither broke alone. That is write skew, and it is why serialisable isolation exists — and why applications using it must handle retries as normal control flow rather than as errors.

**How it fails**

**MVCC failure modes**

| Symptom | Cause | Fix |
|---|---|---|
| Table and index bloat | Long-running transaction pinning old versions | Alert on oldest transaction age; timeouts; run reports on replicas |
| Vacuum runs constantly, reclaims nothing | Oldest snapshot older than the garbage | Find and end the long transaction |
| Disk full on a read-heavy day | Analytics query holding a snapshot | Statement timeouts; dedicated replica |
| Sequential scans slower over time | Dead tuples still scanned until cleaned | Tune autovacuum aggressiveness per table |
| Transaction id wraparound warnings | Vacuum unable to freeze old rows | Treat as an emergency; resolve blocking transactions |
| Unexpected serialisation failures | Serialisable isolation detecting conflicts | Implement retry logic; keep transactions short |
| Replica causes primary bloat | Feedback keeping the primary's snapshot alive for replica queries | Disable feedback and accept query cancellation, or accept the bloat deliberately |

**Limits**

> **Operating guidance**
>
> - **Transaction duration**: aim for seconds. Anything over minutes on a busy primary deserves scrutiny.
> - **Bloat tolerance**: 20–30% is normal; consistently above that means cleanup is losing.
> - **Alert on oldest transaction age**, typically at 5–15 minutes, well before disk pressure.
> - **Update-heavy tables** may need per-table autovacuum tuning; global defaults are usually too passive.
> - **Serialisable isolation** requires the application to retry — budget for a small but non-zero abort rate.

**Alternatives**

| Concurrency control | Readers | Writers | Trade |
|---|---|---|---|
| Two-phase locking | Block writers | Block readers | Simple; poor concurrency |
| MVCC | Never blocked | Blocked only by writers | Version storage and garbage collection |
| Optimistic (OCC) | Never blocked | Validate at commit | Aborts under contention |
| Serializable snapshot isolation | Never blocked | Conflict-tracked | Retryable aborts |
| Single-threaded execution | Serial | Serial | No contention at all; throughput ceiling |

Single-threaded execution deserves a mention: some in-memory systems avoid concurrency control entirely by executing transactions one at a time on a partition, which works because a memory-resident transaction takes microseconds. It is the ownership-partitioning idea applied to concurrency control.

**In real systems**

- **PostgreSQL** keeps versions in the heap, which makes rollback cheap and makes autovacuum tuning a core operational skill.
- **MySQL InnoDB** uses undo logs, keeping tables compact but paying to reconstruct old versions and growing undo under long transactions.
- **Oracle's `ORA-01555` snapshot-too-old error** is the undo-log equivalent of bloat: the old version a long query needs has already been recycled.
- **PostgreSQL's serializable snapshot isolation** detects dangerous read-write dependency structures and aborts, providing true serialisability without locking reads.
- **Read replicas everywhere** rely on MVCC to serve consistent snapshots while continuously applying changes.

**Common mistakes**

- **Holding a transaction open across an external API call or user think-time.**
- **Running long reports on the primary** and causing database-wide bloat.
- **Alerting on disk usage instead of transaction age**, so the cause is invisible until the symptom is critical.
- **Assuming snapshot isolation is serialisable**, then hitting write skew on an invariant.
- **Leaving autovacuum at defaults** on heavily updated tables.
- **Treating serialisation failures as bugs** rather than as expected, retryable outcomes.
- **Ignoring index bloat**, which follows table bloat and degrades lookups.

**The staff-level view**

MVCC is invisible when it works and produces a bewildering class of incidents when it does not — because the cause is usually a harmless-looking read.

- **Alert on the oldest open transaction, everywhere.** It is the leading indicator for an entire family of incidents and it is trivial to monitor.
- **Set idle-in-transaction and statement timeouts as platform defaults**, so a forgotten session cannot become a disk-full outage.
- **Route long analytical reads to replicas** with an explicit policy about whether the primary's cleanup is allowed to be held back for them.
- **Ban external calls inside transactions** in code review; a third party's latency should never determine your bloat.
- **Treat serialisation failures as normal control flow** where serialisable isolation is used, with retry built into the data-access layer rather than left to each caller.

**Go deeper**

Under MVCC, an update writes a new row version rather than overwriting the old one, and each version carries the transaction ids that created and ended it. A snapshot is not a copy of data but a point in the transaction timeline — the set of transactions committed when it was taken — and visibility is a per-row predicate evaluated against it. The result is that readers never block writers and writers never block readers, which removes the dominant source of contention in mixed workloads.

The cost is garbage. Superseded versions remain until no active snapshot could need them, so background cleanup must keep pace with the update rate, and index entries for dead rows bloat alongside the table. The operational hazard that follows is severe and non-obvious: a single long-running transaction pins a snapshot and prevents cleanup of versions in *every* table, so a four-hour report on a busy primary can bloat the whole database and fill the disk. Monitor the age of the oldest open transaction, not just disk usage.

MVCC removes read-write blocking, not write-write conflicts — two transactions updating the same row still serialise. And snapshot isolation is not serialisable: two transactions reading consistent states and writing disjoint rows can jointly violate an invariant neither broke alone, which is write skew. Preventing it needs serialisable isolation, whose retryable aborts must be treated as normal control flow in the data-access layer.

Multi-version concurrency control is the default concurrency model in modern databases because it eliminates the worst contention pattern in mixed workloads: long reads and frequent writes fighting over the same rows.

**The mechanism.** An update is logically an insert plus a delete — a new row version is written, tagged with the creating transaction id, and the old version is marked as ended by that transaction. A snapshot records which transactions had committed at a moment in time, and a version is visible to a reader if its creator is in that committed set and its deleter is not. Taking a snapshot is therefore nearly free regardless of database size, because it is a point in a timeline rather than a copy of anything.

**Two implementations, one trade.** Heap-versioned designs write new versions into the table, making rollback trivially cheap but letting dead tuples and their index entries accumulate until vacuum removes them — which is why vacuum tuning is a core operational skill and why update-heavy tables bloat. Undo-log designs update in place and write the previous value to a separate log, keeping tables compact and scans fast, but paying CPU to reconstruct older versions, making rollback expensive, and growing the undo log under long transactions until a reader's needed version has been recycled. The garbage has to live somewhere; the choice determines which symptom you diagnose.

**Long transactions are the defining hazard.** While any snapshot remains open, no version created after it can be reclaimed — across the entire database, not merely the tables that transaction touches, because cleanup must assume it might read anything. A four-hour analytical query on a primary taking thousands of updates per second therefore causes hours of dead versions to accumulate everywhere, with background cleanup running continuously and reclaiming nothing. The same mechanism produces vacuum futility, index bloat, degrading sequential scans, and eventually transaction id wraparound pressure. Crucially the cause — a harmless-looking read-only query, or a connection idle inside a transaction while waiting on an external API — is visible hours before the symptom, via the age of the oldest open transaction.

**What MVCC does not give you.** It removes read-write blocking, not write-write conflicts: two transactions updating the same row still serialise, one waiting or aborting. And snapshot isolation is weaker than serialisability. Two transactions can each read a consistent state, each verify an invariant, and each write a *different* row such that together the invariant is violated — write skew. Because the writes are disjoint, no write conflict is detected. Preventing it requires serialisable isolation, which tracks read-write dependency structures and aborts one participant, or explicitly materialising the conflict by locking or writing a shared row. Applications on serialisable isolation must treat retries as normal control flow, implemented once in the data-access layer rather than by each caller.

**Platform defaults do most of the prevention.** An idle-in-transaction timeout stops an application holding a snapshot across a network call, which is the single most common cause. A finite statement timeout bounds runaway queries. Monitoring on oldest-transaction age gives hours of warning for a whole family of incidents. And a convention routing long analytical reads to replicas — with an explicit decision about whether the replica may hold back the primary's cleanup, since the default often quietly does — removes the remaining case. These four settings convert a confusing category of 4 a.m. incidents into ordinary tickets.

**Prove it — interview questions**

1. **[Basic] How does MVCC let readers avoid blocking writers?**

   <details><summary>Model answer</summary>

   Writers do not overwrite rows; they create new versions tagged with the transaction that created them. A reader holds a snapshot — effectively a record of which transactions had committed when it started — and filters versions through it, seeing only those created by transactions committed before the snapshot and not yet deleted as of it. Because the reader never needs the writer's new version and the writer never needs to modify the reader's old one, neither has to wait.

   </details>

2. **[Basic] What is a snapshot?**

   <details><summary>Model answer</summary>

   Not a copy of the data, but a point in the transaction timeline: the set of transactions considered committed at the moment it was taken, usually represented as a minimum id, a maximum id, and the list of transactions still in flight. Visibility is then a predicate evaluated per row version rather than a physical snapshot, which is why taking one is nearly free regardless of database size.

   </details>

3. **[Senior] Why does a long-running read-only query cause bloat?**

   <details><summary>Model answer</summary>

   Because while its snapshot is open, no version created after that snapshot can be removed — anywhere in the database, not just in the tables it reads. Garbage collection has to assume the query might still need any of them. So on a primary taking thousands of updates per second, a four-hour report causes hours of dead versions to accumulate across every table, with vacuum running continuously and reclaiming nothing. The fix is to move such queries to a replica, enforce statement and idle-in-transaction timeouts, and alert on the age of the oldest open transaction rather than waiting for disk pressure.

   </details>

4. **[Senior] What is write skew and why does snapshot isolation permit it?**

   <details><summary>Model answer</summary>

   Two transactions each read a consistent snapshot, each check an invariant that holds in their view, and each write a different row — so neither conflicts on a write, but together they violate the invariant. The classic case is two doctors both going off-call because each sees the other still on call. Snapshot isolation permits it because it only prevents concurrent writes to the *same* row, and here the writes are disjoint. Preventing it requires serialisable isolation, which tracks read-write dependencies and aborts one transaction, or materialising the conflict explicitly by writing to a shared row or taking an explicit lock.

   </details>

5. **[Staff] Compare heap-versioned and undo-log MVCC implementations operationally.**

   <details><summary>Model answer</summary>

   Heap versioning, as in PostgreSQL, writes new versions into the table itself, so rollback is trivially cheap — just abandon the new version — but the table and every index on it accumulate dead entries until vacuum removes them, which makes vacuum tuning a core operational skill and makes update-heavy tables prone to bloat. Undo-log designs, as in InnoDB and Oracle, update the row in place and write the previous value to an undo log, so tables stay compact and scans stay fast, but rollback is expensive, reads of old versions cost CPU to reconstruct, and a long transaction grows the undo log instead of the table — surfacing as a snapshot-too-old error rather than as bloat. The underlying trade is identical: the garbage has to live somewhere, and long transactions prevent reclaiming it either way. What differs is which symptom you will be diagnosing at 3 a.m.

   </details>

6. **[Principal] What platform defaults would you set to prevent MVCC incidents?**

   <details><summary>Model answer</summary>

   Four, all cheap and all preventing a recurring class of outage. An idle-in-transaction timeout, so an application holding a transaction open across a network call or user think-time cannot pin a snapshot indefinitely — this single setting prevents the most common cause. A statement timeout sized generously but finitely, so a runaway query cannot run for hours. Standard monitoring on the age of the oldest open transaction, alerting well before disk pressure, because that is the leading indicator for bloat, vacuum futility and transaction id wraparound alike. And a routing convention that long analytical reads go to a replica, with an explicit, documented decision about whether the replica is permitted to hold back the primary's cleanup — because the default behaviour there quietly transfers the problem back to the primary and teams rarely notice they chose it.

   </details>

---

### Transaction isolation levels

*A menu of how much concurrency anomaly you will tolerate in exchange for throughput — and the defaults are weaker than almost everyone assumes.*

**Flow:** `Concurrent transactions` → `Snapshot or locks` → `Conflict detection` → `Commit or retry`

> **The 30-second version**  
> A menu of permitted anomalies. Defaults are Read Committed or Repeatable Read — neither prevents write skew — so enforce invariants with constraints and targeted locks before reaching for serialisable.

**The problem**

Two transactions run at the same time. Whether the result is correct depends entirely on a setting most applications never change and most engineers cannot name. The default in PostgreSQL and Oracle is Read Committed; in MySQL it is Repeatable Read. Neither is serialisable, and the difference shows up as data corruption under load, not as an error.

The dangerous property of weak isolation is that it is invisible in testing. Anomalies require specific interleavings that appear under production concurrency and disappear when you try to reproduce them.

> **The anomalies, in order of subtlety**
>
> - **Dirty read**: seeing uncommitted data. Prevented by every level above Read Uncommitted.
> - **Non-repeatable read**: reading a row twice in one transaction and getting different values.
> - **Phantom read**: re-running a query and finding new rows matching the predicate.
> - **Lost update**: two transactions read, modify and write the same row; one change vanishes.
> - **Write skew**: two transactions each read a consistent state, each write a *different* row, and together break an invariant neither broke alone. This one survives Repeatable Read and is why serialisable exists.

**Mental model**

Isolation levels are a contract about which interleavings the database is allowed to produce. Serialisable means the outcome is equivalent to running the transactions one after another in some order. Everything weaker permits specific deviations in exchange for concurrency.

1. **Read Uncommitted** — May see uncommitted changes. Essentially unused in practice; most MVCC databases do not implement it distinctly.
2. **Read Committed** — A fresh snapshot per *statement*. Never sees uncommitted data, but two statements in one transaction can disagree.
3. **Repeatable Read / Snapshot** — One snapshot for the whole transaction. Consistent reads, but write skew remains possible.
4. **Serialisable** — Equivalent to some serial order. Implemented either by locking reads or by tracking read-write dependencies and aborting.
5. **The retry contract** — At serialisable, the database may abort a transaction that would violate serialisability. The application must retry — that is not an error condition, it is the protocol.

> **The question that picks the level**  
> Ask: *does correctness depend on something I read staying true until I write?* If yes — checking a balance before withdrawing, counting on-call doctors before going off-call, verifying uniqueness before inserting — then Read Committed and Repeatable Read are both insufficient, and you need serialisable isolation or an explicit lock. If no, weaker isolation is fine and cheaper.

**How it works**

**What each level prevents**

```text
                    dirty  non-repeat  phantom  lost    write
                    read   read        read     update  skew
Read Uncommitted      no       no        no       no      no
Read Committed       YES       no        no       no*     no
Repeatable Read      YES      YES       YES**     YES     no
Serialisable         YES      YES       YES       YES     YES

*  lost update is prevented in MVCC read-committed only for
   a single UPDATE statement; a read-then-write across two
   statements is still vulnerable.
** SQL standard permits phantoms at Repeatable Read; MVCC
   implementations usually prevent them via snapshot.

NOTE: level NAMES are standard, BEHAVIOUR is not. Verify
what your specific database does rather than trusting the name.
```

1. **Read Committed re-snapshots per statement** — Inside one transaction, `SELECT count(*)` twice can return different numbers. Any multi-statement logic that assumes stability is already broken.
2. **Snapshot isolation stops concurrent writes to the same row only** — Two transactions writing different rows never conflict, which is exactly the gap write skew exploits.
3. **Serialisable comes in two flavours** — Lock-based (S2PL) takes read locks and blocks; SSI tracks read-write dependency cycles and aborts one participant. SSI preserves MVCC's non-blocking reads and pays in aborts instead of waits.
4. **Explicit locks are the targeted alternative** — `SELECT ... FOR UPDATE` takes a write lock on read rows, converting a read-then-write into a serialised operation without raising the level for the whole transaction.
5. **Materialise the conflict when the rows differ** — Write skew happens because the writes touch different rows. Introducing a shared row to lock — a counter, a summary, a scheduling slot — gives the database something to serialise on.
6. **Keep transactions short** — Every level performs better with short transactions, and serialisable degrades sharply with long ones because conflict windows widen.

**Write skew, concretely**

```text
INVARIANT: at least one doctor must remain on call.
Currently on call: Alice, Bob.

T1 (Alice)                     T2 (Bob)
SELECT count(*) WHERE oncall   SELECT count(*) WHERE oncall
  -> 2, fine to leave            -> 2, fine to leave
UPDATE alice SET oncall=false  UPDATE bob SET oncall=false
COMMIT                         COMMIT

Result: zero doctors on call. No write conflict occurred —
they updated DIFFERENT rows. Snapshot isolation allows it.

FIXES
  a) SERIALIZABLE: SSI detects the read-write dependency
     cycle and aborts one transaction -> retry sees count=1
  b) SELECT ... FOR UPDATE on all oncall rows
     -> serialises the read, second transaction waits
  c) materialise: a shift_coverage row with a count;
     both transactions update THAT row -> write conflict
```

> **Read Committed plus read-then-write is the most common real bug**  
> `SELECT balance` then `UPDATE balance = balance - 100` in two statements is not atomic at Read Committed. Two concurrent withdrawals can both read 100 and both succeed. The fix is either a single conditional statement (`UPDATE ... WHERE balance >= 100`), an explicit `FOR UPDATE` lock, or serialisable isolation — and the first is usually the cheapest and best.

**Worked example**

A booking system, with three implementations of the same feature at increasing correctness.

**Three ways to not double-book**

```text
REQUIREMENT: a room may have at most one booking per time slot.

ATTEMPT 1 (Read Committed, read-then-write) - BROKEN
  SELECT count(*) FROM bookings
    WHERE room=5 AND slot='10:00';        -- both see 0
  INSERT INTO bookings ...;               -- both succeed
  -> double booked

ATTEMPT 2 (unique constraint) - CORRECT, and cheapest
  CREATE UNIQUE INDEX ON bookings(room, slot);
  INSERT ... ON CONFLICT DO NOTHING;
  -> second insert affects 0 rows -> "slot taken"
  works at ANY isolation level, because the constraint is
  enforced by the index, not by the transaction protocol.

ATTEMPT 3 (overlapping ranges - constraint not expressible)
  bookings can be 10:00-11:30, so "same slot" is a RANGE
  overlap, which a unique index cannot express.
  options:
    - exclusion constraint (if the database supports ranges)
    - SELECT ... FOR UPDATE on the room row (serialise per room)
    - SERIALIZABLE isolation with retry
  -> per-room locking is usually best: it serialises only
     what must be serialised, and contention is naturally
     partitioned by room.
```

| Metric | Value | Note |
|---|---|---|
| Read committed | broken | read-then-write race |
| Unique index | correct | **any isolation level** |
| Row lock per room | correct | partitioned contention |
| Serialisable | correct | needs retry logic |

> **Prefer constraints, then targeted locks, then isolation**  
> A unique index solves the problem for all writers at all isolation levels, including migration scripts and future services. An explicit lock serialises exactly the operation that must be serial. Raising the isolation level is the broadest and most expensive instrument, and it makes every transaction pay for one operation's requirement. Reach for them in that order.

**When to use it**

- **Read Committed** for the large majority of operations — reads, reporting, independent writes — where a per-statement snapshot is sufficient.
- **Repeatable Read** for multi-statement reads that must agree with each other: reports, exports, consistency checks.
- **Serialisable** for operations whose correctness depends on a read staying true until a write — balances, allocation, scheduling, quota.
- **Explicit `FOR UPDATE`** when only one specific operation needs serialisation and you want to keep the rest cheap.
- **Constraints** whenever the invariant is expressible as one — always the first choice.

**When to avoid it**

- **Do not run everything at serialisable by default** — you pay aborts and contention for operations that do not need it.
- **Do not assume the default level is safe**; it is Read Committed or Repeatable Read, and neither prevents write skew.
- **Do not implement read-then-write across two statements** without a lock or a conditional predicate.
- **Do not use serialisable without retry logic**; aborts are part of the contract, not an exceptional failure.
- **Do not hold a serialisable transaction open for a long time** — conflict windows widen and abort rates climb sharply.

**Advantages**

- **Weaker levels give higher concurrency** and fewer aborts, which matters at high write rates.
- **Snapshot isolation gives consistent reads for free**, since readers never block.
- **Serialisable removes an entire class of reasoning**: if it commits, the result is equivalent to a serial execution.
- **Per-transaction choice** means you can pay for strictness only where correctness demands it.
- **SSI keeps reads non-blocking**, unlike lock-based serialisability, trading waits for retries.

**Disadvantages**

- **Level names are standard but behaviour is not**, so portable reasoning is unreliable.
- **Weak levels permit anomalies that testing rarely catches**, appearing only under production concurrency.
- **Serialisable costs aborts**, and abort rate grows with transaction length and contention.
- **Retry logic must be implemented correctly**, including idempotency of any side effects performed before the abort.
- **Lock-based serialisability blocks readers**, reintroducing the contention MVCC removed.

**Trade-offs**

**Choosing a mechanism for a read-then-write invariant**

| Mechanism | Scope of cost | Failure mode | Best when |
|---|---|---|---|
| Unique / check constraint | None — enforced by the index | Insert fails; handle it | Invariant is expressible as a constraint |
| Conditional update (`WHERE`) | None | 0 rows affected | Single-row compare-and-set |
| `SELECT ... FOR UPDATE` | Blocks writers of those rows | Waiting, possible deadlock | One operation needs serialising |
| Serialisable isolation | Whole transaction | Retryable abort | Multi-row invariant not expressible otherwise |
| Application-level lock | Whole operation | Lock service dependency | Invariant spans systems |

> **The answer that shows depth**  
> “Read Committed won't help here because the invariant spans two rows — this is write skew, not a lost update. I'd first try to express it as a constraint; if the overlap check makes that impossible, I'd take a `FOR UPDATE` lock on the room row so contention is partitioned by room, and keep serialisable in reserve for cases where even that isn't expressible.”

**How it fails**

**Isolation-related failures**

| Symptom | Cause | Fix |
|---|---|---|
| Double bookings / overselling under load | Read-then-write at Read Committed | Conditional update, constraint, or explicit lock |
| Report totals do not balance | Read Committed re-snapshotting per statement | Repeatable Read for the reporting transaction |
| Invariant broken with no write conflict | Write skew under snapshot isolation | Serialisable, `FOR UPDATE`, or materialise the conflict |
| High abort rate at serialisable | Long transactions or a hot contended row | Shorten transactions; partition contention; use targeted locks instead |
| Deadlocks after adding `FOR UPDATE` | Inconsistent lock acquisition order | Acquire locks in a canonical order everywhere |
| Works in staging, corrupts in production | Anomaly requires real concurrency | Concurrency tests; enforce invariants with constraints |
| Retry loop causes duplicate side effects | Non-idempotent work before the abort | Do external effects after commit; use idempotency keys |

**Limits**

> **Practical guidance**
>
> - **Default levels**: PostgreSQL and Oracle use Read Committed; MySQL InnoDB uses Repeatable Read. Verify rather than assume.
> - **Serialisable abort rate** should stay low single-digit percent; consistently higher means contention needs partitioning.
> - **Transaction duration** dominates abort rate at serialisable — aim for milliseconds.
> - **Retry policy**: bounded attempts with jittered backoff, typically 3–5.
> - **Lock ordering**: a single canonical order for multi-row locks prevents most deadlocks.

**Alternatives**

| Approach | Guarantees | Cost |
|---|---|---|
| Database isolation level | Defined anomaly prevention | Blocking or aborts |
| Database constraints | Enforced for all writers, always | Only expressible invariants |
| Conditional writes / CAS | Single-row atomicity | One row only |
| Partition ownership | Serialisation by routing | Requires an ownership model |
| Application-level locks | Cross-system invariants | Lock service availability; needs fencing |
| Event sourcing with a single writer | Total order per stream | Architectural commitment |

Partition ownership is the scalable answer where it fits: if all operations on a resource route to one owner, they are serialised by routing and no isolation level is needed at all.

**In real systems**

- **PostgreSQL's SSI** provides true serialisability while keeping reads non-blocking, detecting dangerous dependency cycles and aborting one transaction.
- **MySQL InnoDB's Repeatable Read** additionally uses gap locks on ranges, which prevents some phantoms but also causes deadlocks that surprise people migrating from PostgreSQL.
- **Financial systems** typically use explicit row locks or single-statement conditional updates rather than serialisable isolation, because the contention profile is well understood and locks are more predictable.
- **Distributed SQL databases** often default to serialisable or snapshot isolation because their coordination cost makes the weaker levels less of a saving.
- **The `SELECT ... FOR UPDATE SKIP LOCKED` idiom** turns a table into a work queue, serialising claims without blocking unrelated workers.

**Common mistakes**

- **Read-then-write across two statements** at Read Committed.
- **Assuming Repeatable Read prevents write skew** — it does not.
- **Using serialisable without retry logic**, treating aborts as errors.
- **Performing external side effects inside a transaction that may be retried.**
- **Raising isolation globally** to fix one operation.
- **Acquiring multi-row locks in inconsistent order**, creating deadlocks.
- **Trusting level names across databases**, whose behaviours differ materially.

**The staff-level view**

Isolation level is a system-wide default that almost nobody revisits, and the anomalies it permits are exactly the ones testing cannot find.

- **Enforce invariants with constraints wherever possible.** They bind every writer at every isolation level, including tools and future services.
- **Teach the diagnostic question**: does correctness depend on something you read staying true until you write? That single question routes engineers to the right mechanism.
- **Prefer targeted locks to global isolation changes**, so one operation's requirement does not tax every transaction.
- **Where serialisable is used, put retry in the data-access layer** with bounded attempts and jitter, and make sure external side effects happen after commit.
- **Add concurrency tests for invariant-critical paths.** Sequential tests cannot detect these bugs, so the absence of failures in CI means nothing.

**Go deeper**

Isolation levels define which concurrency anomalies the database may produce. Read Committed takes a fresh snapshot per statement, so it prevents dirty reads only. Repeatable Read takes one snapshot per transaction, giving consistent reads but still permitting write skew. Serialisable guarantees the outcome matches some serial order, either by locking reads or by tracking read-write dependencies and aborting one participant.

The practical diagnostic is: does correctness depend on something you read staying true until you write? A balance check before a withdrawal, a count of on-call doctors before going off-call, a slot availability check before booking — all of these break at the default levels. Lost updates on a single row are fixed cheaply with a conditional write; write skew across different rows needs serialisable isolation, an explicit `FOR UPDATE` lock, or materialising the conflict on a shared row.

Prefer mechanisms in this order: a database constraint, which binds every writer at every isolation level including scripts and future services; a conditional single-statement write; a targeted row lock whose contention is partitioned by the natural resource; and serialisable isolation last, since it taxes every transaction and requires retry logic in the data-access layer. And test concurrently — these anomalies never appear in sequential tests.

Isolation levels are a negotiation between correctness and concurrency, and the defaults sit further toward concurrency than most engineers realise. Both major defaults — Read Committed and Repeatable Read — permit anomalies that corrupt data silently under production load.

**The anomaly ladder.** Dirty reads see uncommitted data and are prevented everywhere above Read Uncommitted. Non-repeatable reads mean the same row yields different values within one transaction; phantoms mean the same predicate yields different rows. Lost updates occur when two transactions read, modify and write the same value and one change vanishes. Write skew is the subtle one: two transactions each read a consistent state, each check an invariant, and each write a *different* row, so no write conflict is detected while the invariant is jointly violated. Write skew survives Repeatable Read and is the reason serialisable isolation exists.

**The routing question.** Does correctness depend on something you read remaining true until you write? If not, weak isolation is fine and cheaper. If so, the mechanism depends on shape. A single-row dependency — check a balance, then decrement it — is a lost update, and the best fix is to eliminate the read: a conditional statement whose `WHERE` clause encodes the precondition, with the affected row count as the result. A multi-row dependency is write skew, and needs serialisable isolation, an explicit lock on the rows read, or a materialised shared row that gives the database something to conflict on.

**Order of preference.** Constraints first, always: a unique index or check constraint is enforced by the storage engine for every writer at every isolation level, which means it also binds migration scripts, admin tools and services written years later by people unaware of the original reasoning. Then conditional writes, which are free. Then targeted locks — `SELECT ... FOR UPDATE` serialises exactly the contended resource, and when that resource is naturally partitioned by room, account or tenant, contention scales with the partition rather than globally. Serialisable isolation last, because it applies to the entire transaction and taxes operations that never needed it.

**Serialisable has a protocol, not just a setting.** Under SSI the database aborts transactions that would violate serialisability, which means retries are part of the contract rather than an error condition. That retry belongs in the shared data-access layer with bounded attempts and jittered backoff, and it imposes a design rule: any external side effect — charging a card, sending an email, calling a partner API — must happen after commit, because retrying a transaction that already performed one produces a worse outcome than the anomaly being prevented. Abort rate also scales with transaction duration and contention, so long serialisable transactions on a hot row degrade badly.

**Testing is the organisational gap.** These anomalies require specific interleavings that only occur under real concurrency, so a green sequential test suite is not evidence of correctness. Invariant-critical paths need tests that run the operation in parallel and assert the invariant afterwards. And because level names are standardised while behaviours are not — MySQL's Repeatable Read uses gap locks that PostgreSQL's does not, producing different deadlock profiles — portable reasoning about isolation is unreliable, and the specific database's documented behaviour has to be the reference.

**Prove it — interview questions**

1. **[Basic] What does Read Committed guarantee?**

   <details><summary>Model answer</summary>

   Only that you never see uncommitted data. It takes a fresh snapshot for each statement, so two identical queries within one transaction can return different results if another transaction commits in between. That makes it fine for independent reads and single-statement writes, and insufficient for anything where a value read in one statement is relied upon by a later one.

   </details>

2. **[Basic] What is a lost update and how do you prevent it?**

   <details><summary>Model answer</summary>

   Two transactions read the same value, each modifies it, and each writes back — so one modification is silently overwritten. The classic case is reading a balance of 100, subtracting 100 in the application, and writing 0, twice. The cheapest prevention is to avoid the read entirely: a single conditional statement like `UPDATE accounts SET balance = balance - 100 WHERE id = ? AND balance >= 100`, where the affected row count tells you whether it succeeded. Alternatives are an explicit `FOR UPDATE` lock or serialisable isolation, but the conditional write is simpler and faster.

   </details>

3. **[Senior] Explain write skew and why Repeatable Read does not prevent it.**

   <details><summary>Model answer</summary>

   Two transactions each read a consistent snapshot, each verify an invariant that holds in their view, and each write a *different* row — so no write-write conflict occurs and snapshot isolation sees nothing wrong, yet together they violate the invariant. The canonical example is two on-call doctors each observing that two people are on call and each marking themselves off. Repeatable Read only prevents concurrent writes to the same row, and here the writes are disjoint. Preventing it requires serialisable isolation, which tracks read-write dependencies and aborts one transaction, or an explicit lock on the rows that were read, or materialising the conflict by having both transactions write a shared row.

   </details>

4. **[Senior] When would you use `SELECT ... FOR UPDATE` instead of raising the isolation level?**

   <details><summary>Model answer</summary>

   When exactly one operation needs serialising and I do not want every other transaction to pay for it. A row lock serialises precisely the contended resource and, if that resource is naturally partitioned — per room, per account, per tenant — contention scales with the partition rather than globally. Raising the isolation level applies to the whole transaction and increases abort rates system-wide. The trade-off is that explicit locks block rather than abort, and inconsistent lock ordering across code paths produces deadlocks, so the acquisition order has to be canonical and enforced.

   </details>

5. **[Staff] A team reports rare double bookings that they cannot reproduce. How do you approach it?**

   <details><summary>Model answer</summary>

   Rare and unreproducible is the signature of an isolation anomaly, because it requires a specific interleaving that only appears under production concurrency. First I would find the write path and check whether it reads and then writes across two statements — at Read Committed that is unsafe, and it is by far the most common cause. Then I would determine whether the invariant is single-row, which makes it a lost update fixable with a conditional write or a unique constraint, or multi-row, which makes it write skew requiring serialisable isolation or an explicit lock. My preferred fix order is constraint first, because it binds every writer including migration scripts and future services regardless of isolation level; then conditional write; then a targeted row lock partitioned by the natural resource; and serialisable isolation last, since it is the broadest instrument. I would also add a concurrency test that runs the operation in parallel, since the existing test suite demonstrably cannot catch this class of bug.

   </details>

6. **[Principal] How do you prevent concurrency anomalies across an organisation rather than fixing them one at a time?**

   <details><summary>Model answer</summary>

   By moving enforcement from transaction protocol into schema, because the schema binds everyone. For every invariant, the first question in design review should be whether it is expressible as a unique index, check constraint or exclusion constraint — those hold at any isolation level and cannot be bypassed by a migration script, an admin tool, or a service written next year by someone who has never heard of this discussion. Where an invariant genuinely is not expressible, the pattern should be a documented targeted lock with a canonical acquisition order, not an isolation-level change, so one operation's requirement does not tax the whole system. Alongside that, retry belongs in the shared data-access layer with bounded attempts and jitter, and with a hard rule that external side effects occur after commit, since retrying a transaction that already charged a card is worse than the anomaly it was preventing. Finally, invariant-critical paths need concurrency tests, because a green sequential test suite is not evidence of anything here.

   </details>

---

### Optimistic concurrency control

*Assume conflicts are rare: read a version, compute, then write only if nothing changed — and retry when it did.*

**Flow:** `Read version` → `Compute change` → `Compare-and-set` → `Commit or conflict`

> **The 30-second version**  
> Read a version, compute without holding anything, then write only if the version is unchanged. Free when conflicts are rare, wasteful when they are not.

**The problem**

Pessimistic locking assumes conflict and pays for it up front: acquire a lock, hold it while you work, release it. That cost is real even when no one else wanted the row — and it becomes severe when “while you work” includes a user thinking, a network call, or a long computation.

In most systems, conflicts are rare. Two people editing the same document paragraph at the same second is unusual; two requests updating the same user profile simultaneously is unusual. Optimistic concurrency control exploits that: do the work without holding anything, and validate at the end.

> **The trade, stated precisely**  
> Pessimistic control pays a **guaranteed cost** (lock acquisition, held resources, blocking) to avoid a **rare cost** (wasted work). Optimistic control pays **nothing in the common case** and pays **full re-execution** when conflict occurs. The break-even is the conflict rate: below roughly 10–20% contention, optimistic wins decisively; above it, wasted work dominates and pessimistic is better.

**Mental model**

Read the current state along with a version marker. Do your work. Then write conditionally: “apply this change only if the version is still what I read.” If the version moved, someone else won, and you start over with fresh data.

1. **Version token** — A monotonically increasing number, a timestamp, an ETag, or a hash of the row. It must change on every write.
2. **Read phase** — Fetch state and version. No locks held, no resources reserved.
3. **Compute phase** — Arbitrary work, potentially long, potentially involving user interaction.
4. **Validate-and-write** — A single atomic operation conditioned on the version. Either it applies or it reports conflict.
5. **Retry** — Re-read, re-compute, re-attempt. Bounded attempts with backoff.

> **It is the same primitive everywhere**  
> `UPDATE ... WHERE version = n` in SQL, a conditional put in a key-value store, `If-Match` with an ETag in HTTP, compare-and-swap in a CPU instruction, and a CAS loop in lock-free data structures are all the same idea at different scales. Recognising that means the reasoning transfers: version, validate, retry.

**How it works**

**The pattern at four layers**

```text
SQL
  SELECT data, version FROM docs WHERE id = 1;    -- version 7
  ... user edits for 10 minutes ...
  UPDATE docs SET data = ?, version = 8
   WHERE id = 1 AND version = 7;
  -- 0 rows -> someone else saved; show a conflict UI

KEY-VALUE
  put_if(key, new_value, expect_version = 7)
  -> ok | conflict

HTTP
  GET /docs/1        -> ETag: "v7"
  PUT /docs/1        If-Match: "v7"
  -> 200 OK | 412 Precondition Failed

LOCK-FREE CODE
  do { old = load(ptr); new = f(old); }
  while (!compare_and_swap(ptr, old, new));
```

1. **The version must change on every write** — A timestamp with second granularity is not sufficient under load; two writes in the same second become indistinguishable. Use a counter, or a high-resolution monotonic value.
2. **Retry must re-read, not just re-attempt** — Retrying with the stale version will fail forever. The loop is: read, compute, attempt; on conflict, go back to read.
3. **Bound the retries** — Unbounded retry under high contention is a livelock that consumes CPU and produces nothing. Three to five attempts with jittered backoff, then surface the conflict.
4. **Decide what conflict means to the user** — Sometimes retry silently (incrementing a counter). Sometimes merge (collaborative text). Sometimes the user must choose (“this document changed since you opened it”). That decision is product design, not plumbing.
5. **Side effects must be idempotent or deferred** — If the compute phase sent an email or charged a card, retrying repeats it. Perform external effects after a successful commit, or protect them with idempotency keys.
6. **Version at the right granularity** — Versioning a whole document makes two people editing different sections conflict. Versioning per field or per section reduces false conflicts but complicates the merge.

**Choosing optimistic or pessimistic by contention**

```text
conflict rate   optimistic                  pessimistic
-------------   -------------------------   ------------------------
< 1%            excellent - no lock cost    wasteful lock overhead
1-10%           good - occasional retry     lock contention starts
10-30%          degrading - wasted work     often better
> 30%           livelock risk               clearly better
> 50%           unusable                    the only option

ALSO CHOOSE PESSIMISTIC WHEN:
  the compute phase is very expensive (retry cost is high)
  the compute phase has side effects
  fairness matters (optimistic can starve a slow writer)

ALSO CHOOSE OPTIMISTIC WHEN:
  the "transaction" spans user think-time (never hold a
    database lock across a human)
  the participants are distributed and a shared lock would
    need its own availability story
```

> **Optimistic control can starve slow participants**  
> A transaction that takes 500 ms to compute will keep losing to transactions that take 5 ms, potentially forever. There is no queue and no fairness. If the workload mixes fast and slow writers on the same rows, either partition them or fall back to pessimistic locking for the slow path.

**Worked example**

A collaborative document editor: why optimistic control is the only viable choice, and what the version granularity costs.

**Version granularity changes the conflict rate**

```text
DOCUMENT-LEVEL VERSION
  two users edit different paragraphs
  -> both read version 12, both write version 13
  -> one gets a conflict for no real reason
  false conflict rate: HIGH on any shared document

SECTION-LEVEL VERSION
  docs(id, version)
  sections(doc_id, section_id, content, version)
  -> conflicts only when the SAME section is edited
  false conflict rate: LOW
  cost: more rows, and cross-section invariants are now
        not atomic

OPERATION-LEVEL (CRDT / OT)
  transmit operations, not states
  -> most edits merge automatically; conflicts are rare
  cost: significant complexity; restricted operation set

WHY NOT PESSIMISTIC AT ALL
  holding a lock across human editing time means one user
  blocks another for minutes. Unacceptable at any scale.
```

| Metric | Value | Note |
|---|---|---|
| Doc-level | high false conflicts | simple |
| Section-level | low conflicts | **good default** |
| Operation-level | near zero | complex |
| Pessimistic | unusable | human think-time |

> **Conflict rate is a design parameter, not a fact**  
> Teams treat their conflict rate as something to measure and live with. It is actually something they chose when they picked the version granularity. Halving the scope of a version token often halves the conflict rate, which is a far cheaper intervention than switching concurrency strategies.

**When to use it**

- **Low-contention updates**: user profiles, settings, most CRUD, configuration.
- **Transactions spanning user think-time**, where holding a lock is unacceptable.
- **Distributed systems without a shared lock service**, where a version token travels with the data.
- **HTTP APIs**, where `ETag` and `If-Match` provide the mechanism natively and make clients well-behaved.
- **Read-heavy workloads with occasional writes**, where lock overhead on every access would dominate.

**When to avoid it**

- **Do not use it above roughly 20–30% contention** — wasted work dominates and livelock becomes a real risk.
- **Do not use it when the compute phase is very expensive**; retrying a long computation is costly.
- **Do not use it when the compute phase has irreversible side effects**, unless they are idempotent or deferred to after commit.
- **Do not retry unboundedly**; cap attempts and surface the conflict.
- **Do not use coarse-grained versions** on frequently edited aggregates — you will manufacture conflicts.

**Advantages**

- **No locking cost in the common case**, so the uncontended path is as fast as an unsafe one.
- **No deadlocks**, because nothing is held while waiting for anything else.
- **Works across process, network and user boundaries**, since the version token is just data.
- **Readers are never blocked**, and writers never block each other.
- **Natural fit for HTTP and distributed systems**, where holding a lock across a network is undesirable.

**Disadvantages**

- **Wasted work on conflict**, which is the entire cost and scales with the compute phase.
- **Degrades badly under contention**, with livelock at the extreme.
- **No fairness** — slow writers can be starved indefinitely by fast ones.
- **Retry logic must handle side effects**, which is easy to get wrong.
- **Conflict resolution is a product problem**, not just a technical one, and someone must decide what the user sees.

**Trade-offs**

**Optimistic versus pessimistic**

|  | Optimistic | Pessimistic |
|---|---|---|
| Uncontended cost | Zero | Lock acquire and release |
| Contended cost | Wasted work, retry | Waiting |
| Deadlock | Impossible | Possible; needs ordering discipline |
| Fairness | None — fast writers win | FIFO-ish via lock queue |
| Across user think-time | Natural | Unacceptable |
| Across network / services | Natural | Needs a lock service plus fencing |
| Break-even | Below ~10–20% conflict | Above it |

**How it fails**

**OCC failure modes**

| Symptom | Cause | Fix |
|---|---|---|
| Update silently does nothing | Conflict returned zero rows and was not checked | Always inspect the affected row count and act on it |
| Retry loop never succeeds | Retrying without re-reading the version | Re-read at the top of every retry |
| CPU burn under load | Unbounded retries at high contention | Cap attempts; back off with jitter; fall back to a lock |
| Duplicate side effects | Emails or charges issued during the compute phase | Defer effects until after commit; idempotency keys |
| Frequent conflicts on unrelated edits | Version scope too coarse | Version per field or per section |
| A slow client can never save | Starvation by faster writers | Partition contention, or escalate to a lock after N failures |
| Version never changes | Timestamp granularity too coarse | Use a counter or a high-resolution monotonic value |

**Limits**

> **Working numbers**
>
> - **Break-even conflict rate**: roughly 10–20%; above that pessimistic locking usually wins.
> - **Retry budget**: 3–5 attempts with exponential backoff plus jitter.
> - **Version granularity** directly scales conflict rate — halving the scope roughly halves conflicts.
> - **Compute-phase cost** is the retry cost; long computations make optimistic control expensive.
> - **Escalation**: after N failed attempts, taking a real lock is a reasonable fallback and bounds the worst case.

**Alternatives**

| Mechanism | Best for | Trade |
|---|---|---|
| Optimistic (version check) | Low contention, long compute, distributed | Wasted work; starvation |
| Pessimistic row lock | High contention, short critical section | Blocking; deadlock risk |
| Atomic single-statement update | Simple arithmetic changes | Only expressible operations |
| Partition ownership | Serialising by routing | Requires an ownership model |
| CRDTs | Concurrent edits that must always merge | Restricted operations; metadata growth |
| Queue the writes | Serialising a hot resource | Latency; a queue to operate |

The atomic single-statement update deserves first consideration whenever it fits: `UPDATE counters SET n = n + 1 WHERE id = ?` has no read, no version, no retry and no conflict. Optimistic control is for changes that genuinely require reading before deciding.

**In real systems**

- **HTTP `ETag` and `If-Match`** make optimistic concurrency a standard part of REST semantics, with `412 Precondition Failed` as the conflict signal.
- **DynamoDB conditional writes** and **etcd's compare-and-swap** expose the primitive directly and are the basis of most coordination patterns built on them.
- **ORM version columns** — JPA's `@Version`, ActiveRecord's lock_version — implement exactly this pattern automatically for entity updates.
- **Lock-free data structures** use CPU-level compare-and-swap in the same read-compute-validate loop, at nanosecond scale.
- **Collaborative editors** combine fine-grained versions with operational transformation or CRDTs, because both coarse versions and locks are unusable across human editing time.

**Common mistakes**

- **Not checking the affected row count**, so a conflicting update silently does nothing.
- **Retrying without re-reading**, guaranteeing repeated failure.
- **Unbounded retries** that livelock under contention.
- **Side effects inside the compute phase**, duplicated on retry.
- **Coarse version scope** producing conflicts between genuinely unrelated edits.
- **Using a coarse timestamp as the version**, so concurrent writes look identical.
- **Using it on hot, highly contended rows** where wasted work dominates.

**The staff-level view**

Optimistic concurrency is usually the right default, and the interesting decisions are about granularity and about what a conflict means to the user.

- **Make version checking automatic in the data-access layer**, so an unchecked update cannot silently do nothing — that failure is silent and therefore especially dangerous.
- **Treat version granularity as a design parameter** and tune it deliberately; most conflict problems are granularity problems.
- **Define conflict resolution as a product decision** per entity: silent retry, automatic merge, or a user-facing choice. Leaving it to each developer produces inconsistent behaviour.
- **Enforce the rule that external side effects happen after commit**, because retry semantics make anything else unsafe.
- **Provide an escalation path**: after repeated failures, take a real lock. It bounds the worst case and prevents starvation without abandoning the optimistic fast path.

**Go deeper**

Optimistic concurrency control assumes conflict is rare. You read state with a version token, perform the work holding no locks, then write conditionally on that version. If it changed, your write affects zero rows and you re-read and retry. The uncontended path costs nothing, there are no deadlocks, and the compute phase can be arbitrarily long — including user think-time, where holding a database lock would be unacceptable.

The cost is wasted work on conflict, so the break-even is the conflict rate: below roughly 10–20% optimistic wins decisively, above 30% wasted work dominates and pessimistic locking is better. It also offers no fairness — a writer with a long compute phase can be starved indefinitely by faster ones — so mixed fast and slow writers on the same row need partitioning or escalation to a lock after repeated failures.

Three implementation rules matter. Retry must re-read, not just re-attempt, or it fails forever. Retries must be bounded with jittered backoff, or high contention becomes livelock. And external side effects must happen after commit, because anything performed during the compute phase is duplicated on retry. Finally, treat version granularity as a design parameter: versioning per section rather than per document often halves the conflict rate for free.

Optimistic concurrency control is the same primitive at every scale — compare-and-swap in a CPU, a conditional write in a key-value store, `If-Match` in HTTP, a version column in an ORM — and understanding it once transfers everywhere.

**The economics.** Pessimistic locking pays a guaranteed cost to avoid a rare one: acquire, hold, release, block others, risk deadlock. Optimistic control pays nothing in the uncontended case and pays full re-execution when a conflict occurs. The crossover is the conflict rate, empirically around 10–20%: below it the lock overhead on every operation exceeds the occasional retry, above it wasted work dominates and beyond roughly 30% livelock becomes a genuine risk. This also means the compute phase's cost matters — retrying a 5 ms computation is cheap, retrying a 5 s one is not.

**Where it is the only option.** Any “transaction” spanning user think-time cannot hold a database lock, because one person would block another for minutes. Any coordination across service or network boundaries would otherwise require a distributed lock service with its own availability, fencing and operational story. In both cases the version token is just data that travels with the request, which is why optimistic control, not distributed locking, is the default coordination primitive in well-designed distributed systems.

**Granularity is the lever nobody pulls.** Teams treat their conflict rate as a fact to be measured and endured, when it is actually a consequence of how broadly they scoped the version token. Versioning a whole document makes two people editing unrelated paragraphs conflict; versioning per section or per field eliminates those false conflicts entirely. The constraint is that you cannot version below the granularity at which your invariants must hold — but within that limit, halving the scope typically halves the conflict rate, which is far cheaper than changing strategy.

**Three implementation rules that are easy to get wrong.** The retry loop must re-read: retrying with the stale version fails forever, and this bug looks like a hang. Retries must be bounded with jittered backoff and an escalation path — after N failures, taking a real lock bounds the worst case and prevents the starvation that optimistic control otherwise permits, since there is no queue and no fairness. And external side effects must be deferred until after a successful commit or protected by idempotency keys, because anything done during the compute phase is repeated on every retry; a retried transaction that already charged a card is worse than the anomaly it was avoiding.

**Conflict resolution is a product decision.** What the user sees when a conflict occurs — a silent retry for a counter, an automatic merge for independent fields, or an explicit choice showing what changed and who changed it — is a design question per entity, not a technical default. Leaving it to individual developers produces inconsistent behaviour across a product. The engineering side is to make the version check mandatory in the shared data-access layer, because the failure mode of an unchecked conditional update is that it silently does nothing, which is precisely the kind of bug that survives to production.

**Prove it — interview questions**

1. **[Basic] How does optimistic concurrency control work?**

   <details><summary>Model answer</summary>

   You read the data along with a version marker, do your work without holding any lock, and then write conditionally: apply the change only if the version is still what you read. If the version changed, someone else committed first, so the write affects zero rows and you re-read and retry. The essential property is that nothing is held during the compute phase, so it can be arbitrarily long — including user think-time.

   </details>

2. **[Basic] What does `If-Match` do in HTTP?**

   <details><summary>Model answer</summary>

   It is optimistic concurrency at the protocol level. A `GET` returns an `ETag` representing the current version; the client sends it back in `If-Match` on a subsequent `PUT`, and the server applies the change only if the resource still has that version, otherwise returning `412 Precondition Failed`. It gives the same guarantee as a database version column across a stateless HTTP boundary, without the server holding anything between requests.

   </details>

3. **[Senior] When does optimistic concurrency stop being the right choice?**

   <details><summary>Model answer</summary>

   When the conflict rate gets high enough that wasted work dominates — roughly above 10 to 20 per cent, with livelock becoming a real risk past 30. It is also wrong when the compute phase is expensive, because the retry cost scales with it, and when the compute phase has irreversible side effects that cannot be deferred. A subtler case is fairness: there is no queue, so a writer that takes 500 milliseconds to compute will keep losing to writers taking 5 milliseconds and may starve indefinitely. Where fast and slow writers share a hot row, I would either partition the contention or escalate to a pessimistic lock after repeated failures.

   </details>

4. **[Senior] How do you choose version granularity?**

   <details><summary>Model answer</summary>

   By trading false conflicts against atomicity. A version on the whole document means two users editing different sections conflict for no real reason; a version per section eliminates those false conflicts but means cross-section invariants are no longer enforced by a single check. I treat the conflict rate as a design parameter rather than a measurement — halving the scope of the version token typically halves conflicts, which is far cheaper than changing concurrency strategy. The limit is the smallest unit over which an invariant must hold: you cannot version below the granularity your correctness requires.

   </details>

5. **[Staff] Design conflict handling for a system where users edit shared records.**

   <details><summary>Model answer</summary>

   Three decisions. First, granularity: version at the smallest unit that preserves the invariants, typically per section or per field rather than per record, since that is the cheapest lever on conflict rate. Second, resolution semantics per entity, decided as a product question rather than left to each developer — a counter should retry silently, independent fields should merge automatically, and genuinely conflicting edits should surface a choice showing what changed and who changed it. Third, enforcement: the data-access layer should make the version check mandatory so an unchecked update cannot silently no-op, which is a dangerously quiet failure, and it should own the retry loop with bounded attempts and jitter. I would add the rule that external side effects only occur after a successful commit, since retry makes anything else unsafe, and an escalation to a real lock after repeated failures so a slow client is never starved indefinitely.

   </details>

6. **[Principal] Where does this pattern appear beyond databases, and why does that matter?**

   <details><summary>Model answer</summary>

   It is the same primitive at every scale: compare-and-swap in CPU instructions underpinning lock-free data structures, conditional writes in key-value stores, `If-Match` in HTTP, version columns in ORMs, and compare-and-set in coordination services like etcd. That matters practically because the reasoning transfers completely — read a version, compute without holding anything, validate atomically, retry on conflict — so an engineer who understands it in one place can apply it in all of them, and an architecture can use it consistently from the CPU cache line to the API boundary. It also matters strategically: because the version token is just data, optimistic control works across process, network and organisational boundaries where a lock would require its own availability story, a fencing mechanism and an operational owner. That is why it, rather than distributed locking, is the default coordination primitive in well-designed distributed systems.

   </details>

---

### Deadlocks and lock ordering

*Two transactions each hold what the other needs. The cure is not cleverness — it is acquiring locks in a single canonical order everywhere.*

**Flow:** `Transaction A` → `Lock X` → `Wait for Y` → `Transaction B` → `Wait for X`

> **The 30-second version**  
> Two transactions each hold what the other wants. Eliminate the circular wait by locking rows in one canonical order everywhere, keep transactions short, and always retry the victim.

**The problem**

Transfer money from account 1 to account 2 and, at the same moment, from account 2 to account 1. Each transaction locks its source account, then reaches for its destination — which the other already holds. Neither can proceed and neither will release. The database detects the cycle and kills one.

Deadlocks are not exotic. They appear in any system where a transaction touches more than one row, and they scale with concurrency — meaning they are absent in development and routine in production.

> **Four conditions, one practical remedy**  
> A deadlock needs mutual exclusion, hold-and-wait, no preemption, and a **circular wait**. The first three are inherent to locking. The fourth is the only one you control, and you eliminate it by ensuring every transaction acquires locks in the same total order. That single discipline removes almost every deadlock in practice.

**Mental model**

Picture a wait-for graph: an edge from A to B means A is waiting for a lock B holds. A deadlock is a cycle in that graph. Databases detect cycles and break them by aborting a victim; your job is to make cycles impossible to form.

1. **Lock acquisition order** — If every transaction locks rows in ascending primary key order, no cycle can form — a transaction can only ever wait on a higher key than it holds.
2. **Lock escalation and range locks** — Some engines lock gaps or escalate row locks to page or table locks, creating conflicts between statements that touch no common row.
3. **Index order matters** — Locks are acquired in the order the engine visits rows, which is determined by the access path — so changing an index can change deadlock behaviour.
4. **Victim selection** — The engine chooses which transaction to abort, typically the one with least work done. Your application must retry it.
5. **Deadlock versus lock wait timeout** — A deadlock is detected and broken quickly; a long lock wait is a different symptom with a different cause and a different fix.

> **A deadlock is a correctness-preserving failure, not corruption**  
> The database detected a cycle and aborted a transaction to break it. Nothing is inconsistent. The bug is that your application treated the abort as an error instead of retrying it, and the design flaw is that the cycle was possible at all.

**How it works**

**The canonical deadlock, and the canonical fix**

```text
DEADLOCK
  T1: UPDATE accounts SET .. WHERE id = 1   -- holds lock(1)
  T2: UPDATE accounts SET .. WHERE id = 2   -- holds lock(2)
  T1: UPDATE accounts SET .. WHERE id = 2   -- waits for T2
  T2: UPDATE accounts SET .. WHERE id = 1   -- waits for T1
  -> cycle -> engine aborts one

FIX: ORDER LOCK ACQUISITION
  both transactions sort ids before locking

  ids = sorted([from_id, to_id])            -- [1, 2] always
  for id in ids: SELECT .. FOR UPDATE WHERE id = id
  ... perform the transfer ...

  now T2 also takes lock(1) first, so it simply WAITS
  for T1 to finish. No cycle is possible.

This is the whole technique. It is not sophisticated and it
eliminates the overwhelming majority of real deadlocks.
```

1. **Sort by a stable key before acquiring** — Primary key ascending is the usual choice. Any total order works, as long as every code path uses the same one.
2. **Keep transactions short and narrow** — Deadlock probability grows with how long locks are held and how many rows are touched. Shortening transactions reduces the window more cheaply than any other change.
3. **Beware locks acquired implicitly** — Foreign keys take locks on parent rows, triggers take locks you did not write, and an unindexed foreign key causes a scan that locks far more than intended.
4. **Watch for range and gap locks** — In engines using gap locks at Repeatable Read, two inserts into the same range can deadlock even with no shared row. Reducing isolation or restructuring the predicate often fixes it.
5. **Always implement retry** — The database will abort a victim. Retry with jittered backoff — usually 2–3 attempts is enough, since the conflicting transaction has by then completed.
6. **Read the deadlock log** — Every engine emits the participating statements and the locks held. That output names the two code paths whose ordering disagrees, which is usually the entire diagnosis.

**Less obvious deadlock sources**

```text
UNINDEXED FOREIGN KEY
  DELETE FROM parent WHERE id = 5
  -> scans child table to check references
  -> locks MANY child rows, not just the relevant ones
  -> deadlocks against unrelated child updates
  FIX: index every foreign key column

GAP LOCKS (MySQL Repeatable Read)
  T1: INSERT INTO t (k) VALUES (10)   -- gap lock 5..15
  T2: INSERT INTO t (k) VALUES (12)   -- gap lock 5..15
  -> both wait; deadlock with no shared row
  FIX: Read Committed, or insert with explicit values that
       do not share a gap

LOCK UPGRADE
  SELECT (shared lock) then UPDATE (needs exclusive)
  two transactions both hold shared, both want exclusive
  -> deadlock
  FIX: SELECT ... FOR UPDATE from the start

TRIGGERS AND CASCADES
  locks acquired by code you did not write, in an order
  you did not choose
  FIX: know what your triggers touch; prefer explicit logic
```

> **Deadlock frequency is a design signal**  
> Occasional deadlocks under high concurrency are normal and retry handles them. A *rising* deadlock rate means something changed — a new code path with a different lock order, a dropped index causing scans, or transactions growing longer. Track the rate as a metric rather than treating each occurrence as an isolated incident.

**Worked example**

An order system with rising deadlocks after a feature launch. Diagnosing from the log.

**From deadlock log to root cause**

```text
DEADLOCK LOG (simplified)
  T1 holds: inventory(sku=A) X-lock
     waits:  inventory(sku=B) X-lock
     stmt:   UPDATE inventory SET qty=qty-1 WHERE sku='B'
  T2 holds: inventory(sku=B) X-lock
     waits:  inventory(sku=A) X-lock
     stmt:   UPDATE inventory SET qty=qty-1 WHERE sku='A'

DIAGNOSIS
  both transactions decrement inventory for a multi-item
  order, iterating the cart in the order the USER added
  items -> no canonical order -> cycle possible whenever
  two carts share two SKUs in opposite order

FIX
  ORDER BY sku before the update loop, in every code path
  (checkout, admin adjustment, returns, import job)

BETTER FIX (removes the lock entirely for the common case)
  single statement per item with a conditional predicate:
    UPDATE inventory SET qty = qty - 1
     WHERE sku = ? AND qty >= 1
  still needs ordering if a transaction touches several SKUs,
  but each statement is short, so the window shrinks sharply.

VERIFY
  deadlock rate metric returns to baseline after deploy
```

| Metric | Value | Note |
|---|---|---|
| Cause | unordered iteration | cart order |
| Fix | sort by SKU | **every code path** |
| Window | shorter statements | fewer conflicts |
| Verify | deadlock rate | metric, not anecdote |

> **The fix must cover every code path, not just the guilty one**  
> Ordering only helps if it is universal. A checkout that sorts and an admin tool that does not will still deadlock against each other. This is why lock ordering belongs in a shared helper — a function that takes a set of ids, sorts them, and locks them — rather than as a convention each developer must remember.

**When to use it**

- **Any transaction touching multiple rows** — ordering is cheap insurance and should be the default pattern.
- **Transfers, batch updates and multi-item operations**, which are the classic sources.
- **Systems with high write concurrency**, where deadlock probability is meaningful.
- **When adding explicit `FOR UPDATE` locks**, which introduce ordering requirements that did not exist before.

**When to avoid it**

- **Do not treat deadlocks as errors to log and ignore** — they abort real user operations.
- **Do not retry without backoff and a bound**, which can produce repeated collisions.
- **Do not fix a deadlock by lengthening a lock timeout**; that converts a fast abort into a slow stall.
- **Do not rely on “it rarely happens”**; frequency scales with concurrency and will grow.
- **Do not lock rows inside a transaction that also calls an external service** — the remote latency becomes your lock hold time.

**Advantages**

- **Lock ordering is simple, universal and cheap** — no new infrastructure and no performance cost.
- **Deadlock detection is automatic** in every mainstream engine, so the failure is fast and safe.
- **Aborts preserve correctness**, so retry is a legitimate and complete remedy.
- **Deadlock logs are unusually informative**, naming the exact statements and locks involved.

**Disadvantages**

- **Ordering discipline must be universal** — one non-conforming code path reintroduces the risk.
- **Implicit locks are hard to see**: foreign keys, triggers, cascades and index maintenance all acquire locks you did not write.
- **Gap and range locks create conflicts without shared rows**, which is deeply counter-intuitive.
- **Aborted work is wasted**, and at high rates the waste is significant.
- **Retry requires idempotency** for anything the transaction did before aborting.

**Trade-offs**

**Approaches to deadlock**

| Approach | Effect | Cost |
|---|---|---|
| Canonical lock ordering | Prevents cycles entirely | Discipline across all code paths |
| Shorter transactions | Shrinks the conflict window | May require restructuring logic |
| Optimistic concurrency | No locks held, so no deadlock | Wasted work under contention |
| Single-statement atomic updates | No multi-row lock held | Only for expressible operations |
| Coarser locking (table lock) | Trivially cycle-free | Destroys concurrency |
| Serialising via partition ownership | Routing replaces locking | Requires an ownership model |
| Longer lock timeouts | Hides the symptom | Converts fast aborts into stalls — avoid |

**How it fails**

**Deadlock-adjacent failures**

| Symptom | Likely cause | Fix |
|---|---|---|
| Deadlock rate rising after a release | New code path with a different lock order | Find it in the deadlock log; route through the shared ordering helper |
| Deadlocks on inserts with no shared row | Gap locks at Repeatable Read | Read Committed, or restructure the insert pattern |
| Parent delete deadlocks against child updates | Unindexed foreign key causing a scan | Index the foreign key column |
| Deadlock after adding `SELECT ... FOR UPDATE` | New lock introduced without ordering | Sort ids before locking |
| Lock wait timeouts, not deadlocks | A long transaction holding locks | Shorten transactions; find the slow statement |
| Retry storm after a deadlock | Immediate retry colliding again | Jittered backoff, bounded attempts |
| Deadlocks involving an external call | Remote latency inside a transaction | Move the external call outside the transaction |

**Limits**

> **Operating guidance**
>
> - **Retry attempts**: 2–3 with jittered backoff is usually sufficient, since the conflicting transaction has completed by then.
> - **Deadlock rate** should be near zero at low concurrency and stay flat as traffic grows; a rising trend indicates a new unordered path.
> - **Transaction duration** is the dominant factor — halving it roughly halves deadlock probability.
> - **Detection latency** is typically sub-second, so deadlocks fail fast compared with lock wait timeouts.
> - **Lock timeout** should be short enough that a stall is noticed, but it is not a deadlock remedy.

**Alternatives**

| Strategy | Removes deadlock by | Trade |
|---|---|---|
| Canonical ordering | Eliminating circular wait | Discipline |
| Optimistic concurrency | Holding no locks | Retry cost under contention |
| Atomic single statements | Not holding across statements | Limited expressiveness |
| Partition ownership | Serialising by routing | Ownership model required |
| Queueing writes to a resource | One writer at a time | Latency; a queue to run |
| Wait-die / wound-wait schemes | Timestamp-based preemption | Rarely exposed in mainstream engines |

**In real systems**

- **Bank transfer implementations** canonically sort account ids before locking, which is the textbook example precisely because it is the textbook failure.
- **MySQL InnoDB's gap locks at Repeatable Read** are a frequent source of deadlocks that surprise teams migrating from PostgreSQL, where the default is Read Committed.
- **PostgreSQL's deadlock log** prints the full statement text and lock graph, making diagnosis largely mechanical.
- **ORMs** often acquire locks in map or set iteration order, which is unstable — a common hidden source of unordered acquisition.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** avoids the problem entirely for queue-like workloads by declining to wait at all.

**Common mistakes**

- **Iterating rows in user- or map-determined order** when acquiring locks.
- **Treating a deadlock abort as an unrecoverable error** instead of retrying.
- **Retrying immediately without backoff**, colliding again.
- **Increasing lock timeouts** to “fix” deadlocks, converting them into stalls.
- **Unindexed foreign keys**, silently widening the lock footprint.
- **Calling an external service inside a transaction holding locks.**
- **Fixing ordering in one code path** while others remain unordered.

**The staff-level view**

Deadlocks are a discipline problem with a mechanical solution, so the leverage is in making the correct pattern the easy one.

- **Provide a shared helper that locks a set of ids in sorted order**, and require its use. Conventions decay; a function call does not.
- **Track deadlock rate as a service metric**, so a new unordered code path is visible at deploy time rather than discovered by users.
- **Ban external calls inside transactions** in code review — remote latency inside a lock hold is both a deadlock and a bloat problem.
- **Make retry with jittered backoff part of the data-access layer**, so no caller has to remember that a deadlock is retryable.
- **Index every foreign key.** Unindexed foreign keys cause scans that lock far more than intended, and the resulting deadlocks look unrelated to their cause.

**Go deeper**

A deadlock is a cycle in the wait-for graph: transaction A holds a lock B needs while B holds one A needs. Databases detect the cycle and abort a victim, which preserves correctness — the bug is that the application treated the abort as an error rather than retrying, and that the cycle was possible at all.

Of the four conditions required, only circular wait is practically controllable, and it is eliminated by acquiring locks in a single canonical order — normally ascending primary key. Sorting identifiers before locking makes cycles impossible, but only if every code path does it, which is why the ordering belongs in a shared helper rather than a convention. Shortening transactions is the second lever, since deadlock probability scales with how long locks are held.

The non-obvious sources matter most in practice: an unindexed foreign key makes a parent delete scan and lock many child rows, gap locks at Repeatable Read let two inserts deadlock with no shared row, and locks taken by triggers or cascades are acquired in an order you did not choose. Track the deadlock rate as a metric — a rising trend after a release almost always means a new code path with a different lock order.

Deadlocks are the most mechanical of concurrency problems: a well-defined cause, a well-defined detection mechanism, and a fix that requires discipline rather than sophistication.

**The mechanism.** Four conditions are necessary — mutual exclusion, hold-and-wait, no preemption, and circular wait. The first three are inherent to lock-based concurrency control. The fourth is the one an application controls, and eliminating it is a matter of acquiring locks in a total order: if every transaction locks lower keys before higher ones, a transaction can only ever wait on something ordered after what it already holds, and no cycle can form. Sorting identifiers before a multi-row update is therefore the entire technique, and it removes the overwhelming majority of real deadlocks.

**Universality is the requirement.** Ordering only works if every path obeys it. A checkout flow that sorts and an admin tool that does not will still deadlock against each other, and the admin tool is exactly the path least likely to be reviewed with concurrency in mind. This is why the correct implementation is a shared function that takes a set of identifiers, sorts them, and acquires the locks — so that out-of-order acquisition requires deliberately bypassing the standard path rather than merely forgetting a convention. A common hidden violation is iterating a hash map or a user-ordered collection such as a shopping cart, where the acquisition order is unstable.

**Implicit locks are the hard part.** Locks you did not write cause deadlocks you cannot see in the code. An unindexed foreign key forces a parent delete to scan the child table, locking far more rows than the operation logically concerns and colliding with unrelated updates. Gap locks in engines using Repeatable Read allow two inserts into the same key range to deadlock with no shared row at all, which is deeply counter-intuitive and a frequent surprise when migrating between databases. Triggers and cascades acquire locks in an order chosen by someone else. And a lock upgrade — a shared lock taken by `SELECT` followed by an exclusive lock for `UPDATE` — deadlocks when two transactions both hold shared and both want exclusive, which is fixed by taking `FOR UPDATE` from the start.

**Detection, retry and measurement.** Every mainstream engine detects cycles within milliseconds and aborts a victim, so a deadlock is a fast, correctness-preserving failure rather than corruption. The application's obligation is to retry with jittered backoff — two or three attempts suffice, because the conflicting transaction has completed by then — and that retry belongs in the shared data-access layer so no caller needs to know a deadlock is retryable. Crucially, lengthening lock timeouts is not a remedy: it converts a fast abort into a long stall. The right instrument is a deadlock-rate metric with alerting on trend, because a rising rate after a release almost always identifies a new unordered code path, and because these failures are probabilistic enough that absence of complaints proves nothing.

**Adjacent discipline.** Transaction duration is the multiplier on all of this — halving it roughly halves deadlock probability, and it simultaneously reduces version bloat and lock wait times. The strongest single rule is to prohibit external service calls inside transactions that hold locks, since a remote system's latency then determines both your lock hold time and your snapshot age. Where deadlocks remain frequent despite ordering, the underlying signal is usually that a resource is too contended for lock-based access at all, and the answer is a different model: optimistic concurrency, single-statement atomic updates, or routing all operations on that resource to a single owner so serialisation comes from the routing rather than from locks.

**Prove it — interview questions**

1. **[Basic] What causes a deadlock?**

   <details><summary>Model answer</summary>

   Two or more transactions each holding a lock the other needs, forming a cycle in the wait-for graph. The classic case is a transfer where one transaction locks account 1 then wants account 2, while another locks account 2 then wants account 1. Four conditions are required — mutual exclusion, hold-and-wait, no preemption and circular wait — and circular wait is the only one an application can practically eliminate.

   </details>

2. **[Basic] How do you prevent deadlocks?**

   <details><summary>Model answer</summary>

   By acquiring locks in a single canonical order everywhere, usually ascending primary key. If every transaction locks lower ids before higher ones, a transaction can only ever wait on something ordered after what it holds, so no cycle can form. The discipline has to be universal — one code path that iterates in a different order reintroduces the risk — which is why it belongs in a shared helper rather than in a convention.

   </details>

3. **[Senior] How do you diagnose a deadlock in production?**

   <details><summary>Model answer</summary>

   Read the engine's deadlock log, which prints the participating statements and the locks each transaction held and wanted. That output almost always names the two code paths whose acquisition order disagrees, which is the entire diagnosis. If the statements touch no common row, the cause is usually implicit locking — gap locks on a range at Repeatable Read, or an unindexed foreign key causing a delete to scan and lock many child rows. I would also check whether the deadlock rate rose after a specific release, since a new unordered code path is the most common trigger.

   </details>

4. **[Senior] Why can an unindexed foreign key cause deadlocks?**

   <details><summary>Model answer</summary>

   Because deleting or updating a parent row requires the engine to check for referencing child rows, and without an index on the child's foreign key column that check is a full scan — which locks far more child rows than the operation logically concerns. Those incidental locks then conflict with unrelated updates to other child rows, producing deadlocks between statements that appear to have nothing in common. Indexing every foreign key column narrows the lock footprint to the rows that actually reference the parent, and it is one of the highest-value routine schema fixes.

   </details>

5. **[Staff] Deadlock rate jumped after a release. Walk through your response.**

   <details><summary>Model answer</summary>

   First, confirm it is a deadlock rather than lock wait timeouts, because they have different causes and fixes. Then pull the deadlock log and read the statements — that identifies the two paths involved directly. The most likely cause is a new code path acquiring the same rows in a different order, often because it iterates a collection whose order is user-determined or non-deterministic, such as a cart or a hash map. The fix is to route both paths through a shared function that sorts ids before locking. Beyond the immediate fix, I would check whether transaction duration also increased, since the conflict window scales with hold time, and whether any new foreign key was added without an index. Finally I would verify with the deadlock rate metric rather than by absence of complaints, because these are probabilistic and low traffic can mask a persistent problem.

   </details>

6. **[Principal] How do you keep deadlocks from recurring across many teams and services?**

   <details><summary>Model answer</summary>

   By removing the opportunity rather than teaching the discipline. The data-access layer should expose a single way to lock a set of rows, which sorts the identifiers internally, so that acquiring locks out of order requires deliberately bypassing the standard path. Retry with jittered backoff belongs in the same layer, so no caller has to know that a deadlock is a retryable outcome rather than an error. Deadlock rate should be a standard exported metric with a default alert on trend, so a new unordered path is caught at deploy time by the team that introduced it rather than by users. Two schema-level guardrails complete it: linting that flags foreign keys without indexes, since those silently widen lock footprints, and a review rule against external calls inside transactions, because remote latency inside a lock hold converts a fast deadlock into a long stall and simultaneously causes version bloat. Each of those turns a thing people must remember into a thing that is hard to get wrong.

   </details>

---

### Query planning and statistics

*The planner estimates row counts from statistics and picks an execution strategy — so most “slow query” problems are really bad-estimate problems.*

**Flow:** `SQL` → `Statistics` → `Candidate plans` → `Chosen operators` → `Actual execution`

> **The 30-second version**  
> The planner picks a strategy from estimated row counts. When a query is slow, find the first plan node where estimated and actual diverge — that estimate, not the plan, is the bug.

**The problem**

You write what data you want; the database decides how to get it. For a join of four tables there are dozens of valid orderings and several algorithms for each join, differing in cost by orders of magnitude. The planner explores that space and picks one, guided entirely by estimates of how many rows each step will produce.

When a query is unexpectedly slow, the plan is almost never “wrong” given what the planner believed. It is optimal for a row count that was badly estimated. That reframes debugging completely: the question is not “why did it choose a sequential scan” but “why did it think that would return 50 rows when it returns 500,000.”

> **Read plans by comparing estimated with actual**  
> Every plan node reports an estimated row count and, when you ask for actual execution, a real one. A node where estimate and actual differ by more than an order of magnitude is the root cause, and every choice above it in the tree was made on that false premise. Find the first such node and you have found the bug.

**Mental model**

The planner is a cost model applied to a search space. It enumerates ways to execute the query, estimates the cost of each from statistics about the data, and picks the cheapest. Statistics are a compressed summary of reality, and every failure mode traces back to that compression.

1. **Statistics** — Row counts, distinct values, most common values and their frequencies, histograms of value distribution, and correlation between column order and physical order.
2. **Selectivity estimate** — What fraction of rows a predicate will match, derived from those statistics.
3. **Cardinality estimate** — Rows produced by each node, propagated upward — errors compound multiplicatively through joins.
4. **Cost model** — A weighted combination of estimated IO and CPU, tuned by parameters such as random versus sequential page cost.
5. **Plan choice** — Access method (scan versus index), join algorithm (nested loop, hash, merge) and join order.

> **Cardinality errors compound through joins**  
> If each of three join predicates is underestimated by 10×, the final estimate can be off by 1,000×. That is how a plan that looks sensible at the top of the tree produces a nested loop executing half a million times. The error is almost always introduced at a leaf and amplified upward.

**How it works**

**Join algorithms and when each wins**

```text
NESTED LOOP
  for each row in A: probe index on B
  cost ~ |A| x index_lookup
  WINS when A is small and B has a selective index
  DISASTER when |A| was underestimated

HASH JOIN
  build a hash table on the smaller side, probe with the larger
  cost ~ |A| + |B|, needs memory for the build side
  WINS for large unsorted inputs
  DEGRADES when the build side exceeds memory (spills to disk)

MERGE JOIN
  sort both sides, then walk them together
  cost ~ sort(A) + sort(B) + |A| + |B|
  WINS when inputs are already sorted (e.g. by index order)

The planner's choice hinges almost entirely on estimated
input sizes. Get the estimate wrong and it picks a nested
loop over a million rows.
```

1. **Always read the actual plan, not the estimated one** — `EXPLAIN` shows what the planner intends; `EXPLAIN ANALYZE` (or equivalent) shows what actually happened, including real row counts and timings. Only the second one diagnoses anything.
2. **Keep statistics fresh** — Bulk loads, large deletes and rapidly growing tables invalidate statistics. A plan chosen from month-old statistics on a table that has grown tenfold will be wrong.
3. **Raise the statistics target on skewed columns** — Default histograms are coarse. For columns with heavy skew or high cardinality, more buckets materially improve estimates.
4. **Beware correlated predicates** — Planners assume independence: `WHERE city='Paris' AND country='France'` is estimated as the product of two selectivities, which is wildly wrong. Extended or multi-column statistics fix this where supported.
5. **Parameter sniffing cuts both ways** — A cached plan built for one parameter value can be terrible for another — a plan optimal for `status='archived'` (millions of rows) is wrong for `status='pending'` (twelve rows).
6. **Prefer fixing estimates to forcing plans** — Hints and forced plans freeze a decision that was correct at one data size. They are a last resort, and they need an owner and a review date.

**The estimate-versus-actual diagnostic**

```text
EXPLAIN ANALYZE output, simplified:

  Nested Loop  (est rows=12  actual rows=284,391)  time=48,201ms
    -> Index Scan on orders  (est rows=12  actual rows=284,391)
         Filter: created_at > now() - interval '1 day'
    -> Index Scan on customers  (est rows=1  actual rows=1)

READING IT
  the leaf estimate is off by ~24,000x
  because of that, the planner chose a nested loop
  -> 284,391 index probes instead of one hash join

WHY THE ESTIMATE WAS WRONG
  statistics were last collected before the table grew,
  so "rows created in the last day" was computed against
  an old row count and an old date range.

FIX
  refresh statistics -> planner now estimates ~280k rows
  -> chooses a hash join -> 48s becomes 0.9s

Note: nothing about the query or the indexes changed.
```

> **The first big estimate error is the one to fix**  
> Plans are trees, and an error at a leaf propagates upward. Scan the plan from the bottom for the first node where estimated and actual diverge sharply; everything above it inherited that mistake. Fixing the leaf usually fixes the whole plan, whereas fixing a symptom higher up produces a different bad plan.

**Worked example**

A report that runs in 200 ms for most tenants and 90 seconds for one. Same query, same indexes.

**Parameter sniffing and data skew**

```text
QUERY
  SELECT ... FROM events WHERE tenant_id = $1 AND ts > $2

DATA
  10,000 tenants. 9,999 have < 50k events each.
  1 tenant has 400 million events.

WHAT HAPPENS
  first execution uses tenant_id = 'small-tenant'
  planner estimates ~50k rows -> index scan + nested loop
  plan is CACHED for the prepared statement
  later execution with the huge tenant reuses that plan
  -> nested loop over 400M rows -> 90 seconds

FIXES, in order of preference
  1  make the estimate honest: multi-column statistics,
     higher statistics target on tenant_id
  2  disable plan caching for this statement, so each
     execution plans against its actual parameter
  3  separate code paths for large tenants (they are
     genuinely a different workload)
  4  partition by tenant so the planner sees per-partition
     statistics

WHAT NOT TO DO
  add a hint forcing a hash join -> now the 9,999 small
  tenants pay a hash build they do not need
```

| Metric | Value | Note |
|---|---|---|
| Small tenant | 200 ms | index + nested loop |
| Large tenant | 90 s | **same cached plan** |
| Root cause | plan reuse | across skewed data |
| Best fix | honest estimates | not hints |

> **Skew defeats a single plan, not a single index**  
> When one parameter value covers a million times more data than another, no single cached plan is right for both. The answer is either to stop caching the plan, to make the statistics good enough that re-planning is cheap and correct, or to recognise that the large tenant is a different workload deserving a different path. Forcing a plan optimises for whichever case you had in mind and penalises the other.

**When to use it**

- **Whenever a query is unexpectedly slow** — read the actual plan before changing anything.
- **After bulk loads, large deletes or rapid growth**, when statistics may no longer describe the data.
- **When a query is fast sometimes and slow other times**, which points at parameter sniffing or skew.
- **Before adding an index**, to confirm the planner would actually use it.
- **During schema design**, since correlated columns and skewed distributions are estimate hazards you can anticipate.

**When to avoid it**

- **Do not add hints or force plans as a first response**; they freeze a decision that was correct only at one data size.
- **Do not tune based on `EXPLAIN` alone** — without actual row counts you are guessing.
- **Do not assume the planner is wrong.** It is usually optimal for the estimate it was given.
- **Do not rely on default statistics** for heavily skewed or high-cardinality columns.
- **Do not ignore correlated predicates**, where independence assumptions produce order-of-magnitude errors.

**Advantages**

- **Declarative queries stay portable** across data sizes, indexes and hardware, because the plan adapts.
- **The planner explores orderings no human would enumerate** for multi-table joins.
- **Plans adapt automatically** as data grows, provided statistics are maintained.
- **Plan output is an excellent diagnostic**, exposing exactly where the model and reality diverged.

**Disadvantages**

- **Estimates can be badly wrong**, and errors compound multiplicatively through joins.
- **Independence assumptions fail** on correlated columns, which are extremely common in real schemas.
- **Plan caching plus skewed parameters** produces intermittent, hard-to-reproduce slowness.
- **Statistics maintenance is background work** that competes with foreground traffic.
- **Plan stability is not guaranteed** — the same query can change plan after a statistics refresh, which is both the feature and the risk.

**Trade-offs**

**Responses to a bad plan**

| Response | Effect | Risk |
|---|---|---|
| Refresh statistics | Fixes the root cause | None; should be routine |
| Increase statistics target | Better estimates on skewed columns | Slightly slower analysis, more memory |
| Multi-column / extended statistics | Fixes correlated predicates | Must be declared explicitly |
| Rewrite the query | Removes an estimation hazard | Less readable; may not be portable |
| Add or change an index | Gives the planner a better option | Write cost; may still be ignored |
| Disable plan caching for a statement | Re-plans per parameter value | Planning cost on every execution |
| Hint / force a plan | Deterministic | Freezes a decision; goes stale silently |

Work down this table in order. The first three address the cause; the last one addresses the symptom and creates a maintenance obligation nobody remembers a year later.

**How it fails**

**Planning failures**

| Symptom | Cause | Fix |
|---|---|---|
| Sudden slowdown with no code change | Statistics refreshed and plan flipped, or data crossed a threshold | Compare plans; fix estimates rather than reverting |
| Fast for some parameters, slow for others | Cached plan plus skewed data distribution | Re-plan per parameter, or separate the outlier path |
| Nested loop over millions of rows | Underestimated input cardinality | Refresh and improve statistics on the leaf predicate |
| Hash join spilling to disk | Build side larger than working memory | Increase work memory; reduce build side; better estimate |
| Index exists but is not used | Low estimated selectivity, or a function wrapping the column | Check the plan; index the expression; partial index |
| Estimates wrong on `a AND b` | Independence assumption on correlated columns | Multi-column or extended statistics |
| Plan regressed after a bulk load | Stale statistics on a changed table | Analyse after bulk operations as part of the job |

**Limits**

> **Practical guidance**
>
> - **Estimate versus actual divergence** beyond 10× at any node is the signal to investigate.
> - **Statistics targets**: raise for columns with heavy skew or very high cardinality; defaults are tuned for the average case.
> - **Correlated predicates** can produce errors of two or three orders of magnitude without extended statistics.
> - **Planning time** itself matters for very short queries — thousands of simple queries per second can spend meaningful CPU planning.
> - **Auto-analyse thresholds** are proportional to table size, so very large tables can go a long time between refreshes; tune per table.

**Alternatives**

| Approach | When | Trade |
|---|---|---|
| Cost-based planning (default) | Almost always | Depends on statistics quality |
| Prepared statements with generic plans | High-volume simple queries | Bad under parameter skew |
| Re-plan per execution | Skewed parameters | Planning cost on every call |
| Plan hints / forced plans | A known-good plan that keeps regressing | Goes stale; needs an owner |
| Materialised view | Expensive query run repeatedly | Staleness; maintenance cost |
| Denormalised projection | Query shape is fixed and hot | Write amplification; sync obligation |

**In real systems**

- **PostgreSQL's `EXPLAIN (ANALYZE, BUFFERS)`** shows estimated versus actual rows plus IO, which is the single most useful diagnostic in the ecosystem.
- **Extended statistics** in PostgreSQL exist specifically to fix the correlated-column independence assumption, which is otherwise a leading source of bad plans.
- **SQL Server's parameter sniffing** is the most documented instance of cached-plan-plus-skew, with `OPTIMIZE FOR UNKNOWN` and recompile hints as standard workarounds.
- **Adaptive query execution** in modern engines re-plans mid-flight when actual row counts diverge from estimates, attacking the problem at its root.
- **Query store / plan history features** let teams detect plan regressions, which is how sudden slowdowns with no code change are usually diagnosed.

**Common mistakes**

- **Adding hints before understanding why the estimate was wrong.**
- **Reading `EXPLAIN` without actual row counts.**
- **Assuming stale statistics are rare** after bulk loads or rapid growth.
- **Ignoring correlated predicates** and the independence assumption.
- **Blaming plan instability** rather than fixing the estimate that made both plans plausible.
- **Caching a plan across wildly skewed parameter values.**
- **Tuning cost parameters globally** to fix one query.

**The staff-level view**

Query performance work is diagnostic work, and the most valuable thing a Staff engineer can do is redirect teams from guessing to reading plans.

- **Teach the estimate-versus-actual method explicitly.** It converts performance debugging from folklore into a mechanical procedure.
- **Make statistics maintenance part of bulk-load jobs**, not a background hope. Most plan regressions after a data change are stale statistics.
- **Treat hints as technical debt with an owner and a review date.** They are occasionally correct and always freeze a decision.
- **Capture plans in slow-query logging**, not just durations, since a duration alone cannot distinguish a bad plan from a genuinely large query.
- **Watch for tenant or entity skew in design review**, because a single dominant value in a filtered column is what turns plan caching into an intermittent production incident.

**Go deeper**

A cost-based planner enumerates ways to execute a query — access methods, join algorithms, join orders — estimates each from statistics, and picks the cheapest. Nested loops win when one input is small and the other has a selective index; hash joins win on large unsorted inputs; merge joins win when inputs are already ordered. The choice hinges almost entirely on estimated input sizes.

Therefore most slow queries are estimate problems, not plan problems. Run the query with actual execution statistics and scan from the leaves for the first node where estimated and actual rows diverge by an order of magnitude or more; everything above it inherited that error, and cardinality mistakes compound multiplicatively through joins. A leaf underestimated by 24,000× is why the planner chose a nested loop that probes 284,000 times.

The recurring causes are stale statistics after bulk loads or growth, the independence assumption failing on correlated predicates like city and country, and plan caching combined with data skew — where a plan built for a small tenant is reused for one with a million times more data. Fix estimates first: refresh statistics, raise targets on skewed columns, declare extended statistics. Hints and forced plans freeze a decision that was correct at one data size and go stale silently.

The query planner is a cost model applied to a search space, and understanding it converts performance debugging from folklore into a procedure.

**What it decides.** For each table, an access method — sequential scan, index scan, index-only scan, bitmap scan. For each join, an algorithm: nested loop (cost proportional to the outer input times an index probe, excellent when the outer side is small), hash join (linear in both inputs but requiring memory for the build side), or merge join (requiring sorted inputs, free when an index provides the order). And an ordering for multi-table joins, whose search space grows factorially. Every one of these choices is driven by estimated row counts.

**Why estimates fail.** Statistics are a lossy summary: row counts, distinct values, most-common values and their frequencies, and histograms. They go stale after bulk loads, large deletes and rapid growth, and auto-analyse thresholds proportional to table size mean very large tables can go long periods without refresh. Histograms are coarse by default, so heavily skewed columns need a higher statistics target. And the independence assumption — that `a AND b` selectivity is the product of each — is catastrophically wrong for correlated columns like city and country, or product and category, which are ubiquitous in real schemas. Errors introduced at a leaf then compound multiplicatively as they propagate through joins.

**The diagnostic method.** Run the query with actual execution statistics, not estimates alone, and read the plan from the leaves upward looking for the first node where estimated and actual rows diverge by an order of magnitude or more. That node is where the model departed from reality; every decision above it was made on a false premise, which is why fixing a symptom higher in the tree merely produces a different bad plan. This single procedure resolves the large majority of “why is this query slow” questions and replaces the guesswork of adding indexes and hoping.

**Plan caching plus skew.** A prepared statement's plan built for a parameter matching fifty thousand rows may be reused for one matching four hundred million, producing intermittent slowness that is fast in testing and catastrophic for one tenant. No single cached plan is correct across a million-fold difference in matched rows. The options are to make estimates good enough that re-planning per execution is correct, to disable caching for that statement and pay the planning cost, or to acknowledge that the outlier is a different workload deserving its own path or its own partition. Forcing a plan optimises for whichever case was in mind and penalises the other.

**Order of remedies.** Refresh statistics; raise statistics targets on skewed columns; declare extended or multi-column statistics for correlated predicates; rewrite the query to remove an estimation hazard; add or change an index to give the planner a better option; disable plan caching for a skewed statement. Hints and forced plans are last, because they are occasionally correct and always freeze a decision that was optimal only at one data size — they need a named owner and a review date, or they become invisible technical debt that silently degrades as data grows.

**Organisationally**, capture plans in slow-query logging rather than durations alone, because a duration cannot distinguish a bad plan from a legitimately large query and teams lose enormous time to that ambiguity. Retain plan history so a sudden regression with no deployment can be diagnosed by comparison. Put statistics refresh inside bulk-load and migration jobs rather than trusting background thresholds. And anticipate skew at design review: a filtered column with one dominant value is the precondition for the entire class of intermittent planning incidents.

**Prove it — interview questions**

1. **[Basic] What does a query planner do?**

   <details><summary>Model answer</summary>

   It turns a declarative query into an execution strategy: which access method to use for each table, which join algorithm, and in what order to join. It enumerates candidate plans, estimates the cost of each using statistics about the data, and picks the cheapest. Because the choice is driven entirely by estimated row counts, the quality of those statistics determines the quality of the plan.

   </details>

2. **[Basic] How do you read a query plan?**

   <details><summary>Model answer</summary>

   Run it with actual execution statistics rather than estimates alone, then compare estimated and actual rows at every node. Scan from the leaves upward for the first node where they diverge by more than an order of magnitude — that is where the planner was misled, and every decision above it inherited the error. The operator types and timings then tell you what it chose as a consequence, such as a nested loop that made hundreds of thousands of probes because it expected a dozen.

   </details>

3. **[Senior] Why would the same query be fast for one parameter and slow for another?**

   <details><summary>Model answer</summary>

   Almost always data skew combined with plan caching. If a prepared statement's plan was built for a parameter matching fifty thousand rows and is then reused for one matching four hundred million, the chosen strategy — typically an index scan feeding a nested loop — becomes catastrophic. The fixes in order are: improve statistics so estimates are honest, disable plan caching for that statement so it re-plans per parameter, or recognise that the outlier is genuinely a different workload and give it a separate code path or partition. Forcing a plan is the wrong answer because it optimises for one case at the other's expense.

   </details>

4. **[Senior] Why are correlated predicates a problem?**

   <details><summary>Model answer</summary>

   Because planners assume predicates are independent and multiply their selectivities. `WHERE city = 'Paris' AND country = 'France'` is estimated as the product of two fractions, when in reality the second condition adds almost no additional selectivity — every Paris row is already French. Errors of two or three orders of magnitude are routine, and they propagate upward through joins. The remedy is multi-column or extended statistics where the engine supports them, declared explicitly for the column groups that are genuinely correlated — which is a schema-design-time observation, not something discovered by accident.

   </details>

5. **[Staff] A query that has been fine for a year suddenly got slow, with no deployment. What happened?**

   <details><summary>Model answer</summary>

   Most likely the plan changed, either because statistics were refreshed and the new estimates favoured a different strategy, or because the data crossed a threshold where the cost model's preference flipped. I would compare the current plan with the historical one if plan history is captured, which usually makes it obvious. The important discipline is to fix the cause rather than force the old plan back: if the new plan is worse, the estimates that justified it are probably wrong, and that same estimate error will cause other problems. Common underlying causes are a table that grew past the point where the previous index strategy paid off, a skewed value becoming dominant, or statistics that were stale for a long time and only now got refreshed — meaning the old plan was accidentally right for reasons nobody understood.

   </details>

6. **[Principal] How do you make query performance debuggable across an organisation?**

   <details><summary>Model answer</summary>

   By capturing the evidence automatically and teaching one method. Slow-query logging should record the plan, not merely the duration, because a duration alone cannot distinguish a bad plan from a legitimately large query and teams waste enormous time on that ambiguity. Plan history should be retained so a sudden regression can be diagnosed by comparison rather than by speculation. Statistics refresh belongs inside bulk-load and migration jobs rather than left to background thresholds, since those thresholds are proportional to table size and very large tables can go a long time without an update. Beyond tooling, the single highest-leverage thing is teaching the estimate-versus-actual method, because it converts performance work from folklore — adding indexes and hints and hoping — into a mechanical procedure that reliably identifies the first node where the model diverged from reality. And I would treat hints as debt with a named owner and a review date, since they are occasionally the right answer and always a frozen decision that will silently go stale.

   </details>

---
