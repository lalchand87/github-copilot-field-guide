# Curriculum · Data Storage

[← System Design index](../README.md)

> 8 lessons in **Data Storage**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Data Storage** (8): [Relational data modeling](#relational-data-modeling) · [Document storage](#document-storage) · [Key-value storage](#key-value-storage) · [Wide-column storage](#wide-column-storage) · [Object storage](#object-storage) · [Time-series data modeling](#time-series-data-modeling) · [Graph data modeling](#graph-data-modeling) · [Columnar analytical storage](#columnar-analytical-storage)

## Data Storage

### Relational data modeling

*Model entities and the constraints between them, let the database enforce correctness, and denormalise only where a measured access pattern demands it.*

**Flow:** `Entities` → `Keys` → `Constraints` → `Transactions` → `Queries`

> **The 30-second version**  
> Model entities and relationships, and let constraints enforce the invariants so no writer can bypass them. Normalise for correctness; denormalise only for a profiled query, preferably via a materialised view.

**The problem**

Application code is the wrong place to enforce data correctness. It is duplicated across services, bypassed by migration scripts and admin tools, subject to race conditions between check and write, and it changes every sprint. Data outlives all of it.

A relational model puts the invariants where they cannot be bypassed: unique constraints, foreign keys, check constraints, and transactions. The schema becomes an executable specification of what states are possible.

> **What “we'll enforce it in the service” actually produces**
>
> - **Orphaned rows** after a delete path nobody updated, discovered years later in a report that does not reconcile.
> - **Duplicate entities** because `SELECT`-then-`INSERT` is not atomic under concurrency.
> - **Impossible states** — an order with no customer, a payment with no order — that every downstream consumer must now defensively handle forever.
> - **Silent divergence** between two services that both write the same table with different rules.

**Mental model**

Think of the schema as the set of states the world is allowed to be in. Every constraint removes a class of impossible states permanently, for every writer, including ones written after you leave.

1. **Entity** — A thing with independent identity and lifecycle: customer, order, product. Gets a table and a primary key.
2. **Relationship** — How entities connect: one-to-many becomes a foreign key on the many side; many-to-many becomes a join table with its own meaning.
3. **Key** — Primary key is identity. Natural keys carry meaning and can change; surrogate keys are stable and meaningless. Use surrogate keys for identity, natural keys as unique constraints.
4. **Constraint** — `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`. Each one is an invariant the database enforces on every writer, forever.
5. **Transaction** — The unit within which constraints may be temporarily violated and must be restored. This is why multi-row invariants are cheap inside one database and expensive across services.

> **Normalise for correctness, denormalise for a measured query**  
> Third normal form means every fact is stored exactly once, so it cannot become inconsistent. That is a correctness property, not an aesthetic one. Denormalisation duplicates a fact to make a specific read faster — which means accepting the responsibility to keep copies in sync. Do it when a profiled query demands it, never preemptively, and always name which query justified it.

**How it works**

**A schema that enforces its own rules**

```text
CREATE TABLE customers (
  id          bigserial PRIMARY KEY,          -- surrogate identity
  email       citext NOT NULL UNIQUE,         -- natural key as a constraint
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id          bigserial PRIMARY KEY,
  customer_id bigint NOT NULL
              REFERENCES customers(id),       -- no orphan orders, ever
  status      text NOT NULL
              CHECK (status IN ('pending','paid','shipped','cancelled')),
  total_cents bigint NOT NULL CHECK (total_cents >= 0),
  placed_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  order_id    bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id  bigint NOT NULL REFERENCES products(id),
  quantity    int NOT NULL CHECK (quantity > 0),
  unit_cents  bigint NOT NULL,                -- price AT TIME OF ORDER
  PRIMARY KEY (order_id, product_id)          -- no duplicate lines
);
```

1. **Model time explicitly** — `unit_cents` on the order line is not denormalisation — it is a different fact. The product's price today and the price paid then are distinct, and confusing them corrupts history the first time a price changes.
2. **Use surrogate primary keys, enforce natural keys as constraints** — Email is how humans identify a customer, but emails change. A surrogate key gives stable identity; a unique index on email keeps the business rule.
3. **Prefer `CHECK` over application validation for closed sets** — A status column with a check constraint cannot hold garbage, no matter which service or script writes it. An enum type is stronger still but harder to change.
4. **Choose delete semantics deliberately** — `ON DELETE CASCADE` for genuinely owned children, `RESTRICT` for referenced entities. Soft deletes (`deleted_at`) preserve history but make every query and every unique constraint more complicated.
5. **Index for access patterns, not for tables** — Write the queries first. Composite index column order matters: the leftmost columns must match your equality predicates.
6. **Version the schema as migrations** — Every change is a forward migration checked into the repo, reviewed like code, and applied by the same pipeline in every environment.

**Normalisation, and where to stop**

```text
1NF  atomic columns; no arrays-of-values masquerading as strings
2NF  no partial dependency on part of a composite key
3NF  no transitive dependency: a non-key column must depend
     only on the key
-----
In practice: aim for 3NF, then denormalise deliberately.

DENORMALISE WHEN:
  a profiled query joins 5+ tables on a hot path
  an aggregate is read 1000x more often than written
  a historical value must be frozen (price at purchase)

THE COST YOU TAKE ON:
  every duplicated fact needs an update path
  and a reconciliation job to prove it has not drifted
```

> **The EAV trap**  
> Entity-attribute-value tables (`entity_id, attribute_name, value`) look like flexible schema. They defeat constraints, types, indexes and the query planner simultaneously, and every query becomes a self-join. If you need flexible attributes, use a JSON column with a check constraint or a proper document store — not an EAV table pretending to be relational.

**Worked example**

Modelling inventory reservation — the classic case where the schema either enforces correctness or the application fails to.

**Two designs for the same feature**

```text
NAIVE (application-enforced)
  SELECT available FROM products WHERE id = 42;   -- reads 1
  -- ... application checks available > 0 ...
  UPDATE products SET available = available - 1 WHERE id = 42;
  -- two concurrent requests both read 1, both decrement
  -- -> available = -1. Oversold.

SCHEMA-ENFORCED
  ALTER TABLE products
    ADD CONSTRAINT available_non_negative CHECK (available >= 0);

  UPDATE products
     SET available = available - 1
   WHERE id = 42 AND available > 0;
  -- atomic read-modify-write; 0 rows affected means "sold out"
  -- the CHECK is a backstop against any other writer

RESERVATION MODEL (better still)
  CREATE TABLE reservations (
    id          uuid PRIMARY KEY,
    product_id  bigint NOT NULL REFERENCES products(id),
    quantity    int NOT NULL CHECK (quantity > 0),
    expires_at  timestamptz NOT NULL,
    order_id    uuid UNIQUE          -- at most one order per reservation
  );
  -- availability = stock - SUM(active reservations)
  -- now expiry, audit and partial release are all expressible
```

| Metric | Value | Note |
|---|---|---|
| Naive | oversells | race between read and write |
| Conditional update | correct | one statement, atomic |
| Check constraint | backstop | **binds every writer** |
| Reservation table | auditable | expiry and history |

> **The general move**  
> Whenever correctness depends on “nothing changed between my read and my write,” replace the read-then-write with a **conditional write** whose predicate encodes the condition. The row count tells you whether it succeeded. Then add a constraint as a backstop, because the conditional write protects this code path and the constraint protects every path.

**When to use it**

- **Whenever entities have relationships and invariants that span rows** — which is most transactional business data.
- **When query shapes will evolve.** A normalised model answers questions you have not thought of yet; a denormalised one answers only the questions it was built for.
- **When multiple services or tools write the same data**, so constraints are the only enforcement that cannot be bypassed.
- **For financial, inventory, identity and entitlement data**, where an impossible state is a business incident.
- **As the default starting point**, moving to other models only when a measured access pattern does not fit.

**When to avoid it**

- **Do not normalise past the point of usefulness.** Six joins on a hot read path for purity's sake is a real cost with no correctness benefit.
- **Do not use EAV tables** to simulate flexible schema.
- **Do not denormalise preemptively.** Every duplicated fact is a future inconsistency plus a reconciliation job.
- **Do not model naturally hierarchical documents as many tables** if they are always read and written as a unit — that is a document store's job.
- **Do not put high-volume append-only event streams in a relational table** without partitioning and a retention plan.

**Advantages**

- **Invariants are enforced for every writer**, including future services and one-off scripts.
- **Each fact is stored once**, so it cannot become internally inconsistent.
- **Ad-hoc queries are possible** without redesigning storage, which is worth more over time than any single optimisation.
- **Transactions make multi-row invariants cheap** inside one database.
- **Mature tooling**: query planners, indexes, constraints, migrations, and decades of operational knowledge.

**Disadvantages**

- **Joins cost** on very hot paths, and deep join chains become the bottleneck.
- **Schema changes require migrations**, which need care at large table sizes.
- **Horizontal scaling is hard** — sharding breaks joins and cross-shard transactions.
- **Fixed schema resists genuinely heterogeneous data**, where documents fit better.
- **Object-relational impedance mismatch** costs mapping effort in application code.

**Trade-offs**

**Normalised versus denormalised**

|  | Normalised | Denormalised |
|---|---|---|
| Write cost | One row per fact | Every copy must be updated |
| Read cost | Joins | Single-row read |
| Consistency | Guaranteed by structure | Must be maintained and verified |
| Query flexibility | Any question | Only pre-anticipated questions |
| Storage | Minimal | Multiplied |
| Schema evolution | Localised change | Change every copy |

The honest framing: normalisation buys correctness and flexibility; denormalisation buys read latency and pays in write complexity and drift risk. A materialised view or read projection is often the better middle ground, because the duplication is maintained by the database rather than by application code.

> **A strong trade-off answer**  
> “I'd keep the transactional model in 3NF so constraints enforce correctness, then add a materialised view for the dashboard query that joins six tables. That way the duplication is derived and refreshable rather than hand-maintained, and if it drifts I can rebuild it from the source of truth.”

**How it fails**

**Data modelling failures**

| Failure | Root cause | Fix |
|---|---|---|
| Orphaned rows | No foreign key; a delete path missed a cleanup | Foreign keys with explicit delete semantics |
| Duplicate entities | `SELECT`-then-`INSERT` race | Unique constraint plus `INSERT ... ON CONFLICT` |
| Oversold inventory | Read-then-write without atomicity | Conditional update plus a check constraint |
| Prices change retroactively | Order lines reference current product price | Store the value at the time of the transaction |
| Slow query after growth | Missing or wrongly ordered composite index | Index for the actual predicate; leftmost columns must match |
| Migration locks a large table | `ALTER` requiring a full rewrite | Expand-contract; add nullable, backfill in batches, then constrain |
| Two services disagree about a record | Both write the same table with different rules | One owner per table; others go through its API |

**Limits**

> **Practical limits**
>
> - **Single-node OLTP** comfortably handles tens of thousands of writes per second and terabytes of data on modern hardware.
> - **Join depth**: beyond roughly five to seven tables on a hot path, plan a projection or a materialised view.
> - **Index cost**: each index slows writes and consumes space; audit for unused indexes periodically.
> - **Row size**: very wide rows and large blobs belong in object storage with a reference in the row.
> - **Migration safety**: adding a nullable column is cheap; adding `NOT NULL` with a default, changing types, or adding constraints may rewrite the table — use expand-contract.

**Alternatives**

| Model | Better when | Cost |
|---|---|---|
| Relational | Relationships and invariants matter; queries evolve | Joins; sharding is hard |
| Document | Data is read and written as a whole aggregate | Cross-document consistency is manual |
| Key-value | Access is purely by known key | No queries beyond the key |
| Wide-column | Massive write volume with known partition access | Access patterns must be fixed up front |
| Graph | Traversals of arbitrary depth are the main query | Operational maturity; scaling |
| Columnar | Analytical aggregation over many rows | Poor for point writes |

**In real systems**

- **Order, billing and ledger systems** are overwhelmingly relational, because multi-row invariants and auditability are the product.
- **Identity and entitlement stores** use foreign keys and unique constraints to make impossible permission states unrepresentable.
- **PostgreSQL's JSONB columns** let teams keep a relational spine with flexible attributes, avoiding both EAV and a second datastore.
- **Materialised views and read replicas** are how mature relational systems serve denormalised reads without hand-maintained duplication.
- **Distributed SQL databases** (Spanner-style) exist because organisations wanted relational semantics without giving them up to shard.

**Common mistakes**

- **Enforcing invariants only in application code**, where the next writer bypasses them.
- **`SELECT`-then-`INSERT`/`UPDATE`** instead of a conditional write.
- **Referencing current product price from historical order lines.**
- **EAV tables** in place of a document column or a document store.
- **Denormalising before profiling**, then maintaining copies by hand.
- **Composite indexes with the wrong column order** for the actual predicates.
- **Blocking migrations on large tables** instead of expand-contract.

**The staff-level view**

The schema outlives the services that use it, the framework it was written against, and usually the team. That asymmetry should drive how much care it gets.

- **Push every invariant you can into the database.** Application enforcement is bypassed by the next migration script, the next admin tool, or the next service.
- **Assign one owner per table.** Shared write access across services is how data models diverge silently; other services should go through the owner's API.
- **Make denormalisation justify itself with a named query and a profile**, and prefer database-maintained materialised views over hand-maintained copies.
- **Standardise expand-contract migrations** so schema change is routine rather than an event requiring downtime.
- **Model time deliberately.** Values that were true at a moment — prices, addresses, tax rates — must be captured, not referenced.

**Go deeper**

A relational schema is an executable specification of which states are possible. Foreign keys prevent orphans, unique constraints prevent duplicates, check constraints prevent garbage values, and transactions let multi-row invariants be restored atomically. Crucially, the database is the one place every writer passes through — application-level enforcement is bypassed by the next migration script, admin tool or service.

Normalise to third normal form so each fact is stored once and cannot become internally inconsistent, then denormalise only where a profiled query on a hot path demands it, preferring materialised views so the duplication is derived rather than hand-maintained. Distinguish denormalisation from time modelling: an order line storing the price paid is not a duplicate of the product's current price, it is a different fact, and confusing them corrupts history the first time a price changes.

The recurring correctness pattern is to replace read-then-write with a conditional write whose predicate encodes the condition, using the affected row count as the outcome, and to add a constraint as a backstop. Use surrogate primary keys for stable identity with natural keys as unique constraints, give each table exactly one owning service, and make every schema change an expand-contract migration so it is routine rather than an event.

The schema outlives the application code, the framework, and usually the team. That asymmetry is the argument for putting as much correctness as possible into it.

**Constraints as invariants.** `NOT NULL`, `UNIQUE`, `FOREIGN KEY` and `CHECK` each eliminate a class of impossible states for every writer, permanently. This matters because application-level validation is duplicated across services, skipped by migration scripts and admin tooling, and subject to races between the check and the write. The practical pattern that follows: wherever correctness depends on nothing changing between a read and a write, replace the pair with a conditional write whose `WHERE` clause encodes the condition, and treat the affected row count as the outcome — then add a constraint as the backstop that binds paths you did not write.

**Normalisation is a correctness property.** Third normal form means every fact exists in exactly one place, so it cannot become internally inconsistent. Denormalisation deliberately duplicates a fact to make a specific read cheaper, which means taking on an update path for every copy plus a reconciliation job to prove the copies have not drifted. Do it in response to a profiled query, name the query in the migration, and prefer a materialised view or read projection so the database maintains the duplication and it can be rebuilt from the source of truth.

**Model time explicitly.** An order line storing `unit_cents` is not a denormalised copy of the product's price; it is the price paid, which is a different and permanent fact. The same applies to addresses, tax rates, terms of service versions and exchange rates. Referencing a mutable current value from a historical record silently rewrites the past the first time that value changes, and the corruption is usually discovered much later by a report that does not reconcile.

**Keys and ownership.** Use surrogate primary keys for identity because natural keys carry business meaning and therefore change, and a changing primary key propagates through every referencing foreign key; keep the business rule as a unique constraint. Assign exactly one owning service per table, with others going through its API and their write permission revoked at the database level — three services writing one table with three sets of validation rules means the model has already diverged in ways nobody has measured. Where the data genuinely belongs to different bounded contexts, split the table and propagate events rather than sharing it.

**Evolution.** Schema change should be routine, which means expand-contract as the only pattern: add the new shape, dual-write, backfill in throttled batches, switch reads behind a flag, verify with a reconciliation query, then drop the old shape. Migration tooling should refuse operations that take long locks on large tables, so the unsafe path is unavailable rather than merely discouraged. Combined with scheduled assertion queries for each table's invariants, this converts data correctness from something verified during incidents into something monitored continuously.

**Knowing when to leave.** Relational modelling is the right default, and the signals to move are specific: aggregates always read and written as a unit suggest a document store; access purely by known key suggests key-value; enormous write volume with fixed partition-scoped access suggests wide-column; arbitrary-depth traversal suggests a graph; and aggregation over many rows suggests columnar. What should *not* trigger a move is discomfort with joins, or a desire for schemaless flexibility that a JSON column with a check constraint would satisfy while keeping the relational spine intact.

**Prove it — interview questions**

1. **[Basic] Why put constraints in the database rather than the application?**

   <details><summary>Model answer</summary>

   Because the database is the one place every writer must pass through. Application checks are duplicated across services, skipped by migration scripts and admin tools, and vulnerable to races between the check and the write. A unique constraint or foreign key removes a class of impossible states permanently, for all writers, including ones written after everyone on the current team has left.

   </details>

2. **[Basic] When would you denormalise?**

   <details><summary>Model answer</summary>

   When a specific, profiled query on a hot path is too slow because of joins, or when a value must be frozen at a point in time — an order line storing the price paid is not really denormalisation, it is a different fact. I would not denormalise preemptively, because every duplicated fact needs an update path and a reconciliation job to prove it has not drifted. Where possible I would use a materialised view so the duplication is derived and rebuildable rather than hand-maintained.

   </details>

3. **[Senior] How do you prevent overselling inventory?**

   <details><summary>Model answer</summary>

   Never read then write. Use a conditional update — `UPDATE products SET available = available - 1 WHERE id = ? AND available > 0` — where the predicate encodes the condition and the affected row count tells you whether it succeeded. Add a check constraint that available is non-negative as a backstop, because the conditional update protects this code path while the constraint protects every path, including scripts. For anything with expiry or partial release, I would model reservations as rows instead, so availability becomes stock minus active reservations and the history is auditable.

   </details>

4. **[Senior] Surrogate or natural primary keys?**

   <details><summary>Model answer</summary>

   Surrogate keys for identity, natural keys as unique constraints. Natural keys carry business meaning and therefore change — emails change, SKUs get reissued, country codes get reassigned — and a changing primary key propagates through every foreign key that references it. A surrogate key is stable and meaningless, which is exactly what identity should be. The business rule that emails must be unique is still enforced, just as a unique index rather than as the primary key.

   </details>

5. **[Staff] A single table is now written by three services. How do you fix it?**

   <details><summary>Model answer</summary>

   Assign one owner and move the others behind its API, because three writers with three sets of validation rules means the model has already diverged in ways nobody has noticed yet. Before migrating I would write reconciliation queries that assert the invariants each service believes hold, and run them against production — that usually reveals existing corruption and makes the case concrete. Then I would introduce the owner's API, migrate the other services' writes one at a time behind a flag, and finally revoke their write permission at the database level so the boundary is enforced rather than agreed. If the data genuinely belongs to different bounded contexts, the better answer may be to split the table instead, with events propagating between them.

   </details>

6. **[Principal] How do you make schema change safe at organisational scale?**

   <details><summary>Model answer</summary>

   By making expand-contract the only pattern anyone uses, and by supporting it in tooling rather than in documentation. Every change is decomposed into additive steps: add the new nullable column or table, dual-write, backfill in bounded batches with throttling, switch reads behind a flag, verify with a reconciliation query, and only then drop the old shape. Migration tooling should refuse operations known to take long locks on large tables, so the unsafe path is unavailable rather than discouraged. Alongside that, each table needs a named owner and its invariants expressed as scheduled assertion queries, so drift is detected continuously rather than discovered during an incident. The goal is that schema change stops being an event requiring a maintenance window and becomes an ordinary, reviewable code change.

   </details>

---

### Document storage

*Store an aggregate as one self-contained document so it can be read and written in a single operation — and accept that consistency stops at the document boundary.*

**Flow:** `Aggregate key` → `Document` → `Nested fields` → `Field index` → `Query`

> **The 30-second version**  
> Store each aggregate as one self-contained document for cheap whole-object reads and atomic single-document writes — and accept that consistency and joins stop at the document boundary.

**The problem**

Some data is naturally a whole. A product listing with its variants, images, attributes and localised descriptions is always fetched together, always displayed together, and is meaningless in pieces. Splitting it across eight relational tables means eight joins on every read, and a schema migration every time a new attribute type appears.

Document storage keeps the aggregate intact: one key, one document, one read. The cost is that atomicity and consistency stop at the document boundary.

> **The defining trade**  
> A document store gives you **cheap, complete reads of one aggregate** and **flexible per-document structure**, in exchange for **no cross-document transactions** (or expensive ones) and **no joins**. If your correctness invariants fit inside one document, this is a very good trade. If they span documents, it is a very bad one.

**Mental model**

Think of each document as an object you can serialise and deserialise whole. The database stores the object graph rather than decomposing it into rows.

1. **Aggregate boundary** — The set of data that changes together and must be consistent together. This is the document. Getting this boundary right is the entire design.
2. **Embedding** — Child data stored inside the parent document. Fast to read, but grows the document and must be bounded.
3. **Referencing** — Storing an id and fetching separately. Necessary when the child is large, unbounded, or shared between parents.
4. **Schema-on-read** — The store does not enforce structure; the application interprets it. Flexibility now, migration debt later.
5. **Secondary indexes** — Indexes on fields inside documents make non-key queries possible, at the usual write cost.

> **“Schemaless” means the schema lives in your code**  
> There is always a schema — the set of shapes your application can handle. Making it implicit means five years of accumulated variants, each written by a different version of the code, all of which must still be readable. Use schema validation at the database level, version your documents explicitly, and write migrations, or you will be maintaining a decade of shapes simultaneously.

**How it works**

**Embed or reference: the decision**

```text
EMBED when the child is:
  - owned exclusively by the parent
  - bounded in size and count
  - always read with the parent
  - changed with the parent

  product {
    _id, name, price,
    variants: [ {sku, size, colour, stock}, ... ],   -- bounded, owned
    images:   [ {url, alt, order}, ... ]
  }

REFERENCE when the child is:
  - shared between parents
  - unbounded (comments, events, audit log)
  - large
  - updated independently and frequently

  product { _id, name, category_id }          -- shared
  review  { _id, product_id, body, rating }   -- unbounded

THE ANTI-PATTERN
  product { reviews: [ ...50,000 entries... ] }
  -> document grows without bound, every write rewrites it all,
     and you hit the document size limit at the worst moment
```

1. **Design the document around the write transaction** — Whatever must change atomically must live in one document, because that is the only atomicity you get cheaply. This inverts relational thinking: you model for the write boundary, then make reads fit.
2. **Bound every array** — An unbounded embedded array is the most common document-store failure. Cap it, or reference instead. Some designs keep the most recent N embedded and the rest in a separate collection.
3. **Use versioned documents** — Include a `schemaVersion` field and handle each version on read. Then migrate lazily on write, or in a background job — never with a big-bang migration across billions of documents.
4. **Index deliberately** — Secondary indexes on nested fields work but cost write throughput and memory. Index the queries you actually run; audit unused indexes.
5. **Use optimistic concurrency for updates** — A version field plus a conditional update (`WHERE version = n`) prevents lost updates without locking, which fits the single-document atomicity model.
6. **Do not emulate joins in application code** — Fetching N documents in a loop to assemble a view is a scatter with no query planner. If you need that shape regularly, either restructure the documents or maintain a projection.

**Optimistic update on a document**

```text
read:    { _id: "p42", version: 7, stock: 10, ... }

write:   UPDATE where _id = "p42" AND version = 7
         SET stock = 9, version = 8

matched 0 documents  -> someone else wrote; re-read and retry
matched 1 document   -> success, and no lost update

This is the document-store equivalent of a compare-and-set,
and it is the primary concurrency primitive available to you.
```

> **Document size is a design constraint, not a limit to avoid**  
> Most stores cap documents at a few megabytes. Treat that cap as a signal: if you are approaching it, your aggregate boundary is wrong. Large or unbounded child collections belong in their own documents with a reference, not embedded.

**Worked example**

A product catalogue. Show how the aggregate boundary decides the whole design.

**Aggregate boundary in practice**

```text
ACCESS PATTERNS (write these FIRST)
  A1  render a product page                  99% of reads
  A2  update stock for one variant           frequent write
  A3  list products in a category, paged     common read
  A4  full-text search                       separate system
  A5  show reviews, paged, newest first      common read

DOCUMENT DESIGN
  products/{id} {
    schemaVersion: 3,
    name, description, brand,
    categoryIds: [...],            -- reference: shared, small
    variants: [                    -- embed: owned, ~10-50, bounded
      { sku, size, colour, priceCents, stock, version }
    ],
    images: [ {url, alt} ],        -- embed: owned, bounded
    reviewSummary: { count, avg }  -- DERIVED, updated async
  }
  reviews/{id} { productId, rating, body, createdAt }
                                   -- reference: unbounded (A5)

WHY THIS WORKS
  A1  one read, no joins
  A2  single-document atomic update of one variant's stock
  A3  index on categoryIds + a sort field
  A5  separate collection, indexed by productId + createdAt

WHAT YOU GAVE UP
  reviewSummary can lag reality -> accept it, or recompute
  "all products with stock < 5 across the catalogue" is a
  collection scan unless you index for it explicitly
```

| Metric | Value | Note |
|---|---|---|
| Product page | 1 read | no joins |
| Stock update | atomic | **inside one doc** |
| Reviews | referenced | unbounded |
| Summary | eventually consistent | accepted cost |

> **Access patterns come before schema, always**  
> In relational modelling you can design the entities first and answer new questions later. In a document store the document shape *is* the query plan, so writing the access patterns down first is not optional — a wrong aggregate boundary is discovered only when a query you did not anticipate turns into a full collection scan or an application-side join.

**When to use it**

- **Aggregates read and written as a whole**: product listings, user profiles, content items, configuration, orders with line items.
- **Heterogeneous structure** where documents legitimately differ — multi-tenant custom fields, CMS content types, event payloads.
- **Rapid iteration** where adding fields should not require a migration across a large table.
- **Read-heavy workloads with a known access key**, where one lookup returns everything a page needs.
- **As a read projection** alongside a relational source of truth, materialising the view shape.

**When to avoid it**

- **Do not use it when invariants span documents** — transfers between accounts, inventory across orders, referential integrity between entities.
- **Do not embed unbounded collections.** This is the single most common failure and it appears only at scale.
- **Do not use it as a relational database with no constraints**, assembling joins in application code.
- **Do not rely on “schemaless”** to avoid thinking about schema — version documents and validate them.
- **Do not choose it for analytical aggregation** across many documents; that is a columnar workload.

**Advantages**

- **One read returns a complete aggregate**, with no joins and predictable latency.
- **Single-document writes are atomic**, which covers the majority of real invariants if the boundary is right.
- **Flexible structure per document**, so heterogeneous or evolving data does not require migrations.
- **Natural horizontal scaling**, since documents are self-contained and partition cleanly by key.
- **Application-shaped storage** reduces mapping code between objects and rows.

**Disadvantages**

- **No cheap cross-document transactions**; multi-document consistency is either unavailable, expensive, or manual.
- **No joins**, so queries spanning aggregates become application-side fan-out or duplicated data.
- **Duplicated data across documents** must be kept in sync by the application.
- **Schema drift accumulates** — many shapes coexist, all of which the code must handle.
- **Document size limits** enforce an aggregate boundary you may discover late.
- **Ad-hoc queries are limited** by the indexes you anticipated.

**Trade-offs**

**Document versus relational**

|  | Document | Relational |
|---|---|---|
| Read one aggregate | One operation | Multiple joins |
| Atomicity | Within one document | Across any rows in a transaction |
| Cross-entity query | Application fan-out or duplication | Join |
| Schema change | Per document, lazy | Migration across the table |
| Referential integrity | Application-enforced | Database-enforced |
| Horizontal scaling | Natural | Requires sharding work |
| Ad-hoc analysis | Limited by indexes | Any query |

> **Framing the choice well**  
> “The invariants here are all inside a single order — line items, totals and status change together — so a document store gives me atomic writes without a transaction manager and a one-read product page. What I'm giving up is cross-order queries, so reporting goes to a separate projection rather than being served from this store.”

**How it fails**

**Document store failures**

| Failure | Cause | Fix |
|---|---|---|
| Document exceeds the size limit | Unbounded embedded array | Reference instead; cap embedded entries |
| Write throughput collapses on hot documents | Whole document rewritten on every small change | Split the volatile part into its own document |
| Lost updates | Read-modify-write without a version check | Optimistic concurrency with a version field |
| Application-side joins are slow | Aggregate boundary does not match the query | Restructure documents, or maintain a projection |
| Code full of shape checks | Uncontrolled schema drift | `schemaVersion` field; validation; lazy migration |
| Inconsistent duplicated fields | Denormalised copies updated in only one place | One writer per fact; reconciliation job |
| Slow queries after growth | Query without a supporting index; collection scan | Index the predicate; monitor scanned-versus-returned ratio |

**Limits**

> **Design numbers**
>
> - **Document size cap** is typically a few megabytes — treat 10% of it as your design ceiling.
> - **Embedded arrays** should be bounded and ideally under a few hundred entries.
> - **Whole-document rewrite**: many engines rewrite the document on update, so a 1 MB document with a hot counter is a throughput problem.
> - **Index cost**: each secondary index adds write amplification and memory; audit them.
> - **Cross-document transactions**, where supported, are markedly more expensive than single-document writes — design to avoid them, not to use them.

**Alternatives**

| Option | Better when | Trade |
|---|---|---|
| Relational with JSONB column | You want constraints and joins plus flexible attributes | Single-node scaling limits |
| Document store | Aggregates are self-contained and access is by key | No cross-document consistency |
| Key-value | You never query by anything but the key | No secondary access at all |
| Wide-column | Very high write volume, partition-scoped queries | Rigid access patterns |
| Search engine | Full-text and faceted queries dominate | Not a source of truth |

The first row is underused: a relational table with a JSON column gives document flexibility for the genuinely variable parts while keeping constraints, foreign keys and joins for the parts that need them — often the best answer for a system that is mostly relational with some heterogeneous attributes.

**In real systems**

- **Product catalogues and CMS content** are the canonical fit, since a content item is a self-contained aggregate with variable structure.
- **User profiles and preferences** suit documents because they are read whole and differ per user.
- **MongoDB's guidance** centres on modelling around access patterns and bounding embedded arrays, which reflects where real deployments fail.
- **DynamoDB single-table design** is the same idea taken further: shape the storage entirely around known access patterns.
- **Event payloads and webhook records** are frequently stored as documents because their structure varies by type and they are read individually.

**Common mistakes**

- **Embedding an unbounded collection** such as comments, events or audit entries.
- **Designing documents before writing down access patterns.**
- **Read-modify-write without a version check**, causing lost updates.
- **Assembling joins in application code** as a normal query pattern.
- **Treating “schemaless” as “no schema”**, producing years of undocumented shapes.
- **Putting a hot counter inside a large document** that is rewritten on every update.
- **Choosing a document store because relational felt restrictive**, then rebuilding referential integrity by hand.

**The staff-level view**

The document store decision is really the aggregate-boundary decision, and it is expensive to reverse.

- **Require written access patterns before the schema.** In a document store the shape is the query plan; without the patterns you are guessing.
- **Audit for unbounded arrays in design review.** This failure appears only at scale, long after the code was written.
- **Mandate document versioning and validation from day one.** Retrofitting a version field onto a billion undifferentiated documents is far harder than adding it early.
- **Be explicit that cross-document consistency is the application's job**, and specify how — sagas, idempotency, reconciliation — rather than leaving it implicit.
- **Consider a relational store with JSON columns** before adopting a second database; a new datastore is a permanent operational commitment.

**Go deeper**

A document store keeps an aggregate intact: one key, one document, one read, no joins. Writes to a single document are atomic, which covers most real invariants if the aggregate boundary is drawn correctly. Structure can vary per document, so heterogeneous or evolving data does not require migrations, and self-contained documents partition cleanly for horizontal scaling.

The cost is that atomicity and querying stop at the document boundary. There are no joins, cross-document transactions are absent or expensive, and any duplicated data must be kept in sync by the application. This makes the aggregate boundary the entire design decision: whatever must change together must live together. Write the access patterns down before the schema, because in a document store the document shape *is* the query plan.

Two failures dominate real deployments. Embedding an unbounded collection — comments, events, audit entries — grows the document without limit, rewrites it on every write, and hits the size cap late and painfully; reference those instead. And treating “schemaless” as “no schema” accumulates years of undocumented shapes; carry a `schemaVersion`, validate at the database level, and migrate lazily. Concurrency is handled with a version field and conditional updates, the document-store form of compare-and-set.

Document storage optimises for one thing: reading and writing a complete aggregate in a single operation. Everything good and bad about it follows from that.

**The aggregate boundary is the design.** Whatever must change atomically has to live inside one document, because single-document atomicity is the only cheap consistency available. This inverts relational habits — you model for the write boundary and then make reads fit, rather than modelling entities and letting queries evolve. It also means the access patterns must be written down before the schema: the document shape *is* the query plan, and an unanticipated query becomes a collection scan or an application-side join with no planner to help.

**Embed or reference.** Embed when the child is exclusively owned, bounded in count and size, and read and changed with the parent — variants, images, line items. Reference when it is shared, unbounded, large, or independently updated — reviews, comments, audit events. The dominant production failure in document stores is an unbounded embedded array: it grows past the size cap, and because many engines rewrite the whole document on update, write cost grows with total size rather than with the size of the change. A hot counter inside a large document is a throughput problem for the same reason, and the fix is to split the volatile field into its own document.

**Concurrency.** With no multi-row transactions, the primitive is optimistic concurrency: carry a version field, make updates conditional on the version you read, and retry on mismatch. This is compare-and-set, and it prevents lost updates without locking. Anything requiring atomicity across documents needs application-level machinery — a saga with compensation, idempotency keys, and a reconciliation job that asserts the invariant continuously — and a design that needs this frequently has drawn its boundaries wrong.

**Schema is not optional, only implicit.** “Schemaless” means the schema lives in application code as the set of shapes it can handle, and over years that becomes a decade of variants all of which must still be readable. The discipline is a `schemaVersion` field from day one, database-side validation for the current version so new writes cannot introduce surprises, lazy upgrade on write plus a throttled background migration, and a stated policy for how many versions the read path supports before old ones are force-migrated. Retrofitting this onto a billion undifferentiated documents is far harder than adding it early.

**Choosing it at all.** The honest comparison is often not against a relational database but against a relational database with a JSON column, which gives flexible attributes for the genuinely variable parts while keeping constraints, foreign keys and joins for the rest. Adopting a second datastore is a permanent operational commitment — backups, failover, capacity planning, expertise, on-call knowledge — so the bar should be a workload the existing store genuinely cannot serve. When migrating a transactional system, the deciding artifact is usually the list of invariants that currently span rows, because every one of them becomes application-enforced after the move, and teams consistently underestimate that work.

**Prove it — interview questions**

1. **[Basic] When should you embed versus reference?**

   <details><summary>Model answer</summary>

   Embed when the child is owned exclusively by the parent, bounded in size and count, and always read and changed with it — product variants and images are typical. Reference when the child is shared between parents, unbounded, large, or updated independently — reviews, comments, audit events. The failure to avoid is embedding something unbounded, because the document grows without limit, every write rewrites the whole thing, and you hit the size cap at the worst possible moment.

   </details>

2. **[Basic] What atomicity does a document store give you?**

   <details><summary>Model answer</summary>

   Typically atomic writes within a single document, and nothing cheaper than that across documents. That is why the aggregate boundary is the central design decision: whatever must change together should live in one document. Cross-document transactions exist in some engines but are markedly more expensive, and a design that depends on them routinely has drawn its boundaries wrong.

   </details>

3. **[Senior] How do you handle concurrent updates to the same document?**

   <details><summary>Model answer</summary>

   Optimistic concurrency: include a version field, and make the update conditional on the version you read. If zero documents match, someone else wrote first, so you re-read and retry. This is the document-store equivalent of compare-and-set and it avoids lost updates without locking. For hot documents I would also consider splitting the volatile field into its own smaller document, because many engines rewrite the entire document on update, so a frequently incremented counter inside a large document is a throughput problem as well as a contention one.

   </details>

4. **[Senior] How do you manage schema evolution without migrations?**

   <details><summary>Model answer</summary>

   By making the schema explicit even though the store does not enforce it. Every document carries a `schemaVersion`, the read path handles each supported version, and documents are upgraded lazily on write or by a throttled background job — never in a big-bang migration across billions of documents. I would also enable database-side validation for the current version so new writes cannot introduce unplanned shapes, and I would set a policy for how many versions the code supports before old ones must be force-migrated, otherwise the read path accumulates years of branches nobody dares delete.

   </details>

5. **[Staff] How do you decide between a document store and a relational database with JSON columns?**

   <details><summary>Model answer</summary>

   I look at where the invariants live and how the data is queried. If correctness is contained within single aggregates and access is overwhelmingly by key, a document store is a natural fit and scales horizontally without effort. If invariants span entities — referential integrity, cross-row constraints, transfers — then I want the database enforcing them, and a relational store with a JSON column gives me flexible attributes for the genuinely variable parts while keeping constraints and joins for the rest. I weight this toward the relational option more than people expect, because adopting a second datastore is a permanent operational commitment: backups, failover, capacity, expertise and on-call knowledge, all duplicated. The bar for a new datastore should be a workload the existing one genuinely cannot serve, not a preference about modelling style.

   </details>

6. **[Principal] A team wants to move a transactional system from relational to document storage for scaling reasons. How do you evaluate it?**

   <details><summary>Model answer</summary>

   First I would establish whether the scaling problem is real and whether cheaper options are exhausted — indexes, read replicas, vertical scaling and caching routinely buy an order of magnitude and are reversible. Then I would ask them to enumerate the invariants that currently span rows, because every one of those becomes application-enforced after the move: sagas, idempotency keys, and reconciliation jobs that must be built, tested and operated. That list is usually the deciding artifact, since teams tend to underestimate it substantially. I would also check whether the access patterns are genuinely stable, because document stores serve anticipated queries well and unanticipated ones badly, and transactional systems accumulate new query shapes constantly. If it still looks right, I would push for a strangler approach — move one bounded context first, keep the relational store authoritative for anything with cross-entity invariants, and prove the reconciliation tooling works before extending it.

   </details>

---

### Key-value storage

*The simplest possible contract — get, put, and conditional put by key — which is exactly what makes it scale horizontally without limit.*

**Flow:** `Key` → `Partition function` → `Value store` → `Conditional update`

> **The 30-second version**  
> Get, put and conditional put by key — nothing else. That restriction is what allows near-linear horizontal scaling, and it moves all the design work into key format and lifecycle.

**The problem**

Every query capability a database offers costs something at scale. Joins require colocating data. Secondary indexes require maintaining a second structure on every write. Transactions require coordination. Range scans require ordered placement.

A key-value store removes all of it. The only operations are by key, which means the key alone determines placement, which means any node can serve any key independently, which means adding nodes adds capacity with no coordination. It is the only data model that scales essentially without limit — because it refuses to do anything that would prevent it.

> **The bargain**  
> You give up every access path except the key. In return you get predictable single-digit-millisecond latency, near-linear horizontal scaling, and operational simplicity. The entire design question becomes: **can I express everything I need as a lookup by a key I will always know?**

**Mental model**

A distributed hash map. The key hashes to a partition, the partition lives on a node, and the node returns the value. Nothing else is happening.

1. **Key** — Determines placement and is the only access path. Key design *is* schema design here.
2. **Value** — Opaque bytes to the store, structured only by your application. It may be a serialised object, a counter, a blob.
3. **Partition function** — Usually consistent hashing, so adding or removing a node moves only a fraction of keys.
4. **Conditional write** — Compare-and-set on a version or an expected value. This is your only concurrency primitive, and it is enough for most invariants.
5. **TTL** — Optional expiry per key, which makes the store usable as a cache, a session store, or a lock with automatic release.

> **Composite keys are how you get query capability back**  
> You cannot query by attribute — so you encode the attribute into the key. `user#123#order#2024-06-01` supports “orders for user 123” if the store supports prefix scans over sorted keys. The design work moves entirely into the key format, and once written, applications depend on that format permanently.

**How it works**

**The complete API, and what you build on it**

```text
CORE
  get(key)                   -> value | null
  put(key, value)            -> ok
  delete(key)                -> ok
  put_if(key, value, expect) -> ok | conflict     <- the important one
  expire(key, ttl)

DERIVED PATTERNS
  counter        put_if(k, n+1, expect=n) with retry
  lock           put_if(k, owner, expect=null) + TTL + fencing token
  session        put(sid, blob) with TTL
  idempotency    put_if(idem_key, result, expect=null)
                 -> first writer wins, others read the stored result
  secondary idx  maintain key "by_email#alice@x.com" -> user_id
                 (YOU keep it consistent; the store will not)
```

1. **Design the key first and treat it as permanent** — Applications, backfills and analytics all encode the key format. Changing it later means rewriting every key in the store.
2. **Include a namespace and a version in the key** — `v1:user:123:profile` lets you evolve formats and coexist with old data during a migration. Retrofitting a prefix is far harder.
3. **Use conditional writes for every read-modify-write** — Unconditional `put` after a `get` loses concurrent updates silently. A version-checked write turns that into a visible conflict you can retry.
4. **Keep values small and bounded** — Large values hurt: they are transferred whole, they fragment memory, and a hot large key saturates a node's network. Store big blobs in object storage and keep a reference.
5. **Maintain secondary indexes yourself, and accept they can drift** — A `by_email` key pointing at a user id is an index you must write, delete and reconcile. Without transactions across the two keys, drift is possible — plan a repair job.
6. **Expect hot keys** — Real access is Zipfian. One key can exceed a node's capacity regardless of how many nodes you have. Plan replication of hot keys, client-side caching, or key splitting.

**Idempotency: the canonical key-value pattern**

```text
client sends:  POST /payments  Idempotency-Key: abc123

server:
  result = put_if("idem:abc123", {status:"in_progress"},
                  expect=null)
  if conflict:
      stored = get("idem:abc123")
      if stored.status == "done": return stored.response
      else:                       return 409 in-progress

  ... perform the payment ...

  put("idem:abc123", {status:"done", response:...}, ttl=24h)

WHY THIS FITS PERFECTLY:
  one key, one conditional write, no coordination,
  no transaction, scales to any volume.
```

> **Scans are not a feature you can rely on**  
> Most key-value stores can enumerate keys, but doing so touches every partition and is O(dataset). It is acceptable for an occasional maintenance job with throttling; it is never acceptable in a request path. If you find yourself wanting to scan, the key design is wrong or you need a different store alongside.

**Worked example**

Session storage for 50 million users. Why key-value is the right answer and how to size it.

**Session store design and capacity**

```text
ACCESS PATTERN
  every authenticated request:  get(session_id)
  login:                        put(session_id, blob, ttl=24h)
  logout:                       delete(session_id)
  "log out all devices":        needs user -> sessions index

KEY DESIGN
  v1:sess:{session_id}          -> {user_id, roles, issued_at}
  v1:usersess:{user_id}         -> set of session_ids  (secondary)

CAPACITY
  50M users x 1.5 sessions      = 75M keys
  value ~ 400 bytes             = 30 GB of values
  + key + overhead (~100 B)     ~ 38 GB
  x 2 replicas                  ~ 76 GB    -> a few nodes

THROUGHPUT
  20k rps authenticated traffic -> 20k get/s
  one node handles 100k+ get/s  -> node count driven by
                                   memory, not throughput

WHAT YOU ACCEPT
  the usersess index is maintained by the application and
  can drift if a delete fails -> TTL on sessions bounds the
  damage, plus a periodic reconciliation pass
```

| Metric | Value | Note |
|---|---|---|
| Keys | 75M | sessions |
| Memory | ~76 GB | replicated |
| Lookup | <1 ms | **every request** |
| Index drift | bounded by TTL | accepted |

> **TTL is a correctness tool, not just a cleanup tool**  
> The secondary index can drift because there is no transaction spanning the two keys. But because sessions expire, any drift self-heals within the TTL. Designing so that inconsistency is *bounded in time* is the standard way to live without transactions — and it is why TTL appears in so many key-value designs that have nothing to do with caching.

**When to use it**

- **Access is always by a key you already have**: sessions, tokens, idempotency records, feature flags, user preferences, shopping carts.
- **Extremely high throughput with predictable latency requirements**, where a query planner is overhead.
- **Caches and derived data**, where the authoritative copy lives elsewhere.
- **Coordination primitives**: distributed locks with TTL and fencing, leader election, rate-limit counters.
- **As a serving layer** in front of a richer store that holds the source of truth.

**When to avoid it**

- **Do not use it when you need to query by attributes** you cannot encode into the key.
- **Do not use it for data requiring multi-entity transactions**, unless the store offers them and you have measured the cost.
- **Do not scan in a request path** — it is O(dataset) and touches every partition.
- **Do not store large blobs**; keep them in object storage and store the reference.
- **Do not assume secondary indexes you maintain are consistent** — design a reconciliation path.

**Advantages**

- **Near-linear horizontal scaling**, because keys are independent and placement is a pure function.
- **Predictable low latency**, typically sub-millisecond to low single-digit milliseconds.
- **Operationally simple**: no query planner, no index maintenance, no join optimisation to reason about.
- **Conditional writes give real concurrency control** without transactions or locks.
- **TTL handles expiry natively**, which removes a whole class of cleanup jobs.

**Disadvantages**

- **One access path only.** Anything else requires a second structure you maintain.
- **No joins or aggregation**; cross-key views must be assembled by the application or precomputed.
- **Application-maintained indexes drift**, and there is no constraint to catch it.
- **Key format is effectively permanent** once applications depend on it.
- **Hot keys cannot be solved by adding nodes** — a single key lives on one partition.
- **Range queries require sorted keys**, which not all stores provide.

**Trade-offs**

**Key-value flavours**

| Type | Example use | Strength | Limitation |
|---|---|---|---|
| In-memory (Redis, Memcached) | Cache, session, rate limit | Sub-millisecond; rich data types | Memory-bound; durability is a choice |
| Disk-backed distributed (DynamoDB) | Source-of-truth KV at scale | Managed scaling, conditional writes | Cost per operation; item size limits |
| Embedded (RocksDB, LMDB) | Local state for a stateful service | No network hop | Not shared; needs replication |
| Ordered KV (etcd, FoundationDB) | Config, coordination, ranges | Prefix scans, transactions | Lower throughput ceiling |

The ordered variants matter more than people expect: sorted keys give prefix scans, which is how composite-key designs recover “list all X for Y” without a secondary index.

**How it fails**

**Key-value failures**

| Failure | Cause | Fix |
|---|---|---|
| Hot key saturates one node | Zipfian access; a single key is one partition | Client-side cache, replicate the key, or split it with a suffix |
| Lost updates | `get` then unconditional `put` | Conditional write on version; retry on conflict |
| Secondary index out of sync | No transaction across the two keys | Write index first, reconcile periodically, bound with TTL |
| Scan in a request path times out | O(dataset) operation | Redesign the key; add a maintained index |
| Memory exhaustion / eviction of live data | No TTL, or eviction policy evicts needed keys | Set TTLs; choose the eviction policy deliberately; monitor evictions |
| Large value stalls a node | Multi-megabyte values transferred whole | Store blobs externally; keep values small |
| Key format change requires full rewrite | No version prefix in the key | Include a version segment from day one |

> **The unbounded-key-growth failure**  
> Key-value stores make it trivially easy to write a key and never think about it again. Without a TTL or an explicit deletion path, the keyspace grows forever — and because there is no schema, nobody notices until memory or cost becomes a problem and nobody can determine which keys are still in use. Every key written should have an owner and either a TTL or a documented deletion trigger.

**Limits**

> **Operating numbers**
>
> - **Single node**: 100k+ simple operations/s for in-memory stores; tens of thousands for disk-backed.
> - **Latency**: sub-millisecond in-memory, low single-digit milliseconds for managed distributed stores.
> - **Value size**: keep under tens of kilobytes; hundreds of KB or more is a design smell.
> - **Hot key ceiling**: one partition's throughput, regardless of cluster size.
> - **Overhead**: expect roughly 50–100 bytes per key of metadata beyond key and value size.

**Alternatives**

| Store | Adds | Costs |
|---|---|---|
| Key-value | Baseline: key access, conditional writes, TTL | No other access path |
| Document store | Queries on fields inside the value | Index maintenance; more complex engine |
| Wide-column | Ordered clustering within a partition | Access patterns fixed at design time |
| Relational | Joins, constraints, transactions | Scaling requires sharding |
| Search index | Arbitrary attribute and text queries | Not authoritative; eventual |

**In real systems**

- **Session and token stores** are the archetypal use: keyed by an opaque id, expiring naturally via TTL.
- **DynamoDB's conditional writes** are the standard mechanism for idempotency and optimistic concurrency in serverless architectures.
- **Redis as a rate limiter and distributed lock** relies on atomic operations plus TTL, with fencing tokens for safety.
- **Feature flag and configuration services** distribute a key-value snapshot to clients, since reads vastly outnumber writes.
- **Dynamo's original paper** established consistent hashing plus conditional writes as the pattern that later stores generalised.

**Common mistakes**

- **`get` then unconditional `put`**, losing concurrent updates without any error.
- **No version prefix in keys**, making format evolution a full rewrite.
- **Scanning the keyspace** from a request path.
- **Writing keys with no TTL and no deletion path**, so the keyspace grows forever.
- **Storing large blobs** instead of references to object storage.
- **Assuming an application-maintained index is consistent.**
- **Believing more nodes will fix a hot key.**

**The staff-level view**

Key-value is where architectural simplicity is bought with design discipline up front. The discipline is almost entirely about keys and lifecycles.

- **Treat the key format as a public API.** Namespace, version, and document it; changing it later means rewriting the keyspace.
- **Require a TTL or a documented deletion trigger for every key written.** Unbounded keyspace growth is the slow failure nobody owns.
- **Mandate conditional writes for read-modify-write.** Unconditional puts after reads lose updates silently, and silence is the problem.
- **Make application-maintained indexes come with a reconciliation job.** Without transactions, drift is not a possibility but a certainty over time.
- **Plan hot-key mitigation at design time**, because adding capacity cannot fix a key that exceeds one partition.

**Go deeper**

A key-value store offers access only by key, which means the key alone determines placement, any node can serve any key without coordination, and adding nodes adds capacity near-linearly. Conditional writes — compare-and-set on a version — provide concurrency control without transactions, and TTL provides expiry natively.

The design work moves entirely into keys. Composite keys such as `v1:user:123:orders:2024-06` encode attributes that you would otherwise query by, and ordered stores support prefix scans over them. Secondary access requires a second key you maintain yourself — `by_email:...` pointing at a user id — which also serves as a uniqueness constraint when written conditionally, but can drift because no transaction spans the two keys, so a reconciliation job is part of the design rather than an afterthought.

Three things bite in production. Read-then-unconditional-write loses concurrent updates silently, so every read-modify-write must be conditional. A hot key cannot be fixed by adding nodes, since one key lives on one partition — plan caching, replication or key splitting up front. And keys written without a TTL or a deletion trigger accumulate forever in a store with no schema to tell you what is still in use, which becomes a cost crisis nobody can safely resolve.

Key-value storage is defined by what it refuses to do. No joins, no secondary indexes, no transactions across keys, no query planner. Each of those omissions removes a reason for nodes to coordinate, which is precisely why it scales without a practical ceiling.

**The interface and what you build on it.** Get, put, delete, conditional put, and TTL. From those five operations come counters (conditional increment with retry), distributed locks (conditional put of an owner plus TTL and a fencing token), session stores, rate limiters, idempotency records, and application-maintained secondary indexes. The conditional write is the load-bearing primitive: it is compare-and-set, and it is sufficient for most single-entity invariants without any coordination protocol.

**Key design is schema design, and it is permanent.** Because the key is the only access path, attributes you need to filter by must be encoded into it, and ordered stores then support prefix scans over composite keys — which is how `user#123#order#2024-06` recovers “orders for this user in this month” without an index. Once applications, backfills and analytics encode a key format, changing it means rewriting the keyspace, so a namespace and a version segment belong in every key from the first line of code.

**Consistency without transactions.** A secondary index maintained as a second key can diverge from the primary record, because nothing spans them atomically. Two techniques make this survivable. First, write the index entry with a conditional put, which makes it enforce uniqueness as a side effect — the write that would create a duplicate simply fails. Second, bound the inconsistency in time: TTLs mean drift self-heals, which is why expiry appears in so many key-value designs that have nothing to do with caching. A periodic reconciliation job completes the picture, and should be treated as part of the design rather than as remediation.

**Hot keys are the ceiling.** Real access distributions are heavy-headed, and one key lives on exactly one partition, so a sufficiently popular key exceeds a node's capacity no matter how large the cluster is. The mitigation ladder is client-side or process-local caching with a short TTL for read-hot keys; replication under suffixed names with aggregation on read where the operation is commutative; dedicated placement where the store allows it; and, if writes must be serialised and exceed one partition, a data model change so the entity decomposes. Adding nodes is never the answer.

**Lifecycle is the slow failure.** It is trivially easy to write a key and never think about it again, and with no schema there is nothing to tell you later which keys are still in use. Years on, memory or cost becomes a problem and nobody can safely delete anything. The organisational fix is to treat namespaces as owned resources — a named team, a documented versioned format, a retention policy — and to require a TTL or an explicit deletion trigger for every key written, enforced in a client wrapper rather than in a document. Combined with per-namespace usage and hot-key metrics, that keeps a shared store from becoming an anonymous liability.

**Prove it — interview questions**

1. **[Basic] Why do key-value stores scale so well?**

   <details><summary>Model answer</summary>

   Because the key alone determines placement, so any node can serve any key with no coordination, no joins to colocate, and no global index to maintain. Adding a node adds capacity and — with consistent hashing — moves only a fraction of the keys. The scalability is a direct consequence of the restricted interface: it scales because it refuses to offer the operations that would prevent it.

   </details>

2. **[Basic] How do you implement a counter safely?**

   <details><summary>Model answer</summary>

   With a conditional write, not a read followed by a write. Read the current value and version, then issue a write conditional on that version; if it fails, someone else incremented first, so re-read and retry. Many stores also offer a native atomic increment, which is better still because it avoids the round trip. What you must not do is `get` then unconditional `put` — that silently loses concurrent increments with no error to tell you.

   </details>

3. **[Senior] How do you support “find the user by email” in a key-value store?**

   <details><summary>Model answer</summary>

   By maintaining a second key that acts as an index: `by_email:alice@example.com` mapping to the user id, written alongside the user record. The catch is that there is usually no transaction spanning the two keys, so they can diverge if one write fails. I would write the index entry first with a conditional put so duplicate emails are rejected, then the user record, and add a reconciliation job that walks users and repairs index entries. Where uniqueness is a hard invariant, the conditional write on the index key is what actually enforces it — the index is doing double duty as a uniqueness constraint.

   </details>

4. **[Senior] What do you do about a hot key?**

   <details><summary>Model answer</summary>

   Recognise first that adding nodes does not help, because one key lives on one partition. The options in increasing cost are: cache it client-side or in a local process cache with a short TTL, which handles read-hot keys well; replicate the key under several suffixed names and have clients pick one at random, aggregating on read where the operation is commutative; or give the key dedicated capacity if the store allows explicit placement. If the key is write-hot and the writes must be serialised, that is a genuine ceiling and the fix is to change the data model so the entity decomposes.

   </details>

5. **[Staff] Design an idempotency mechanism for a payments API using a key-value store.**

   <details><summary>Model answer</summary>

   The client sends an idempotency key with the request. The server does a conditional put of `idem:{key}` with state in-progress, expecting the key to be absent. If that succeeds, this is the first attempt and it proceeds with the payment, then overwrites the record with the final response and a TTL covering the retry window — typically 24 hours. If the conditional put conflicts, it reads the existing record: if the state is done, it returns the stored response verbatim so the retry is indistinguishable from the original; if in-progress, it returns a conflict so the client backs off rather than double-charging. This fits key-value perfectly because it is one key, one conditional write, no coordination and no transaction, and it scales to any volume. The subtlety worth naming is crash recovery: a process that dies after claiming the key leaves an in-progress record, so the record needs a timestamp and a policy for when an abandoned attempt may be retried.

   </details>

6. **[Principal] What organisational practices keep a large shared key-value store healthy?**

   <details><summary>Model answer</summary>

   Three, and they are all about lifecycle rather than performance. First, treat key namespaces as owned resources: every prefix has a named team, a documented format including a version segment, and a stated retention policy, so the store does not become an anonymous dumping ground nobody can safely clean. Second, require a TTL or an explicit deletion trigger for every key written, enforced where possible by a client library wrapper — unbounded keyspace growth is a slow, ownerless failure that surfaces as a cost or memory crisis years later with no way to determine what is still in use. Third, publish per-namespace usage and hot-key metrics so that noisy neighbours in a shared cluster are visible before they cause an incident. Beyond that, I would push for the store to be treated as a cache or derived layer wherever possible, with authority living somewhere that has constraints — because a key-value store cannot tell you when your data has become wrong.

   </details>

---

### Wide-column storage

*A partition key spreads data across the cluster and a clustering key orders rows inside each partition — so queries are fast only if you designed the partition for them.*

**Flow:** `Partition key` → `Token routing` → `Partition` → `Clustering order` → `Range read`

> **The 30-second version**  
> Partition key chooses the node, clustering key orders rows within it. Queries are cheap only along that axis, so you design one table per query shape and bound every partition.

**The problem**

Key-value stores scale but offer only point lookups. Relational stores offer rich queries but resist horizontal scaling. Wide-column storage occupies the space between: it scales like a key-value store while supporting ordered range reads inside a partition.

The catch is that this capability is available only along the axis you chose when you designed the table. Get the partition key wrong and you have a key-value store with extra complexity; get the clustering key wrong and your most important query becomes a full-cluster scan.

> **The three ways wide-column designs fail**
>
> - **Hot partition**: a partition key with a heavy head — a celebrity user, a busy tenant, a single day — concentrates load on one node.
> - **Unbounded partition**: a partition that grows forever (all events for a device, all messages in a room) eventually exceeds what one node can hold or read efficiently.
> - **Query the model does not support**: any filter not on the partition key plus a clustering prefix becomes a scatter across every node.

**Mental model**

Picture a distributed map of ordered maps. The outer map is keyed by partition and spread across nodes. The inner map is sorted by clustering key and lives entirely on one node, so scanning a range inside it is a sequential read.

1. **Partition key** — Hashed to a token, determines which node holds the data. All queries must specify it. Its distribution determines whether load is even.
2. **Clustering key** — Orders rows within the partition. Range reads along this order are cheap; anything else is not.
3. **Row** — The unit within a partition, identified by the clustering key. Columns may be sparse and differ per row.
4. **Replication factor** — How many nodes hold each partition; combined with a tunable quorum it decides consistency and availability.
5. **Query contract** — Partition key equality is mandatory; clustering key prefix equality or range is optional. Nothing else is efficient.

> **Tables are written per query, not per entity**  
> In relational modelling you design entities once and write many queries. Here you design a table per query shape and write the same data multiple times. Three access patterns often means three tables, kept consistent by the application. That duplication is not a smell — it is the model working as intended, because storage is cheap and coordination is not.

**How it works**

**Primary key structure decides everything**

```text
PRIMARY KEY ( (partition_key) , clustering_key_1, clustering_key_2 )
               ^ which node        ^ sort order within that node

EXAMPLE: messages in a chat room
  PRIMARY KEY ( (room_id), created_at DESC, message_id )

  efficient:  "last 50 messages in room R"
              -> one partition, sequential read, no merge
  efficient:  "messages in room R between T1 and T2"
  INEFFICIENT: "all messages by user U"
              -> U is not the partition key; scatter over
                 every node; effectively unsupported

FIX for the second query: a SECOND table
  PRIMARY KEY ( (user_id), created_at DESC, message_id )
  same data, written twice, queried two ways
```

1. **Bound every partition** — Add a time or sequence component to the partition key — `(room_id, month)` — so partitions have a maximum size. Unbounded partitions are the most common production failure.
2. **Check the partition key's distribution** — A key with few distinct values, or a heavy-headed distribution, creates hot partitions. Composite keys with a bucket suffix spread a hot entity across several partitions at the cost of a fan-out on read.
3. **Avoid secondary indexes at scale** — Engine-provided secondary indexes typically require querying every node. A second table (a manual materialised view) is almost always better.
4. **Understand the tombstone problem** — Deletes and TTL expiries write tombstones that must be read past until compaction removes them. A partition with many deletes becomes progressively slower to read, which surprises teams badly.
5. **Use lightweight transactions sparingly** — Compare-and-set operations exist but require a consensus round trip and are an order of magnitude more expensive. They are for rare uniqueness checks, not for hot paths.
6. **Tune consistency per query** — Quorum reads and writes give strong-enough consistency; lower levels give lower latency and higher availability. Choose per operation, not globally.

**Bounding and spreading a partition**

```text
PROBLEM: sensor readings
  PRIMARY KEY ( (sensor_id), ts )
  one sensor, 1 reading/sec, forever
  -> 31M rows/year in ONE partition on ONE node

FIX 1  time bucketing (bounds size)
  PRIMARY KEY ( (sensor_id, day), ts )
  -> 86,400 rows per partition; queries specify the day(s)

PROBLEM: one very hot sensor
  even bucketed, sensor_99 takes 40% of writes

FIX 2  bucket suffix (spreads load)
  PRIMARY KEY ( (sensor_id, day, bucket), ts )
  bucket = hash(ts) % 4
  -> writes spread over 4 partitions
  -> reads fan out to 4 and merge  <- the cost you accept
```

> **Read-before-write is an anti-pattern here**  
> The engine is optimised for writes: they append without reading. Introducing a read to check state before writing gives up that advantage and adds a round trip plus a race. Prefer designs where writes are blind — append an event, upsert a full row — and resolve conflicts by last-write-wins timestamps or by reading a range and reducing in the application.

**Worked example**

A notification feed: 100 million users, each with a personal timeline, reading the most recent 50 entries.

**Table design and capacity**

```text
ACCESS PATTERNS
  A1  latest 50 notifications for a user      (99% of reads)
  A2  mark one notification as read           (frequent write)
  A3  notifications for a user in a date range
  A4  count of unread                          <- careful

TABLE
  PRIMARY KEY ( (user_id, month), created_at DESC, notif_id )
  -> A1: one partition, top 50, sequential
  -> A3: one or two partitions
  -> bounded: ~1000 rows/user/month

A2: mark as read
  upsert the row with read_at set
  blind write, no read-before-write

A4: unread count
  NOT a query this model supports cheaply.
  maintain a counter table, or compute at write time,
  or accept an approximate count from the last partition.

CAPACITY
  100M users x 12 partitions/year = 1.2B partitions
  avg partition 1000 rows x 200 B = 200 KB    <- healthy
  write rate 50k/s                            -> spread evenly
                                                 by user_id hash
```

| Metric | Value | Note |
|---|---|---|
| Partition size | ~200 KB | healthy |
| Read A1 | 1 partition | sequential |
| Unread count | not supported | **needs a counter** |
| Distribution | even | hashed user_id |

> **The query you forgot is the one that hurts**  
> A4 — the unread count — is trivial in SQL and awkward here, because counting requires reading the partition. Wide-column modelling punishes queries you did not anticipate, which is why enumerating access patterns before designing tables is not a best practice but a prerequisite. Every late-arriving query costs either a new table plus a backfill, or an expensive scan.

**When to use it**

- **Very high write throughput** with known partition-scoped access: time series, event logs, feeds, metrics, IoT telemetry.
- **Queries that are always scoped to one entity** and want ordered ranges within it.
- **Multi-datacentre replication with tunable consistency**, where availability during partitions matters more than strong consistency.
- **Predictable linear scaling** without the operational work of sharding a relational database.

**When to avoid it**

- **Do not use it when query patterns are unknown or will evolve freely** — the model is fixed at design time.
- **Do not use it for data requiring multi-row transactions or joins.**
- **Do not use it for aggregation across partitions**; that is an analytical workload for a columnar store.
- **Do not rely on engine secondary indexes at scale** — they usually scatter across all nodes.
- **Do not design partitions that grow without bound**, and do not ignore key distribution.

**Advantages**

- **Write throughput is exceptional**, because writes are appends with no read and no coordination.
- **Linear horizontal scaling** with predictable capacity per node.
- **Ordered range reads within a partition** are sequential and fast.
- **Tunable consistency per operation**, so each query can choose its latency/consistency point.
- **Multi-region replication is a first-class feature** rather than a bolt-on.
- **No single point of failure** in a peer-to-peer topology.

**Disadvantages**

- **Query flexibility is fixed at design time**; new access patterns need new tables and backfills.
- **Data is duplicated across query-specific tables**, and the application keeps them consistent.
- **Tombstones from deletes and TTLs degrade read performance** until compaction.
- **No joins, and transactions are limited** to a partition or to expensive lightweight transactions.
- **Hot and unbounded partitions are easy to create** and painful to fix after the fact.
- **Operational expertise required**: compaction strategy, repair, and consistency level tuning are not defaults you can ignore.

**Trade-offs**

**Consistency level trade-offs (RF=3)**

| Read / Write | Consistency | Latency | Availability |
|---|---|---|---|
| W=1, R=1 | Weak — may read stale | Lowest | Survives 2 node failures |
| W=QUORUM, R=QUORUM | Strong enough (W+R > RF) | Moderate | Survives 1 node failure |
| W=ALL, R=1 | Strong reads, fragile writes | High write latency | Any node loss blocks writes |
| W=1, R=ALL | Strong reads, fast writes | High read latency | Any node loss blocks reads |

Quorum for both is the usual choice because W + R > RF guarantees overlap, giving read-your-writes without the fragility of requiring all replicas. The other rows exist for specific cases: analytics tolerating staleness, or configuration data written rarely and read everywhere.

**How it fails**

**Wide-column failures**

| Failure | Cause | Fix |
|---|---|---|
| One node hot, others idle | Partition key with skewed distribution | Add a bucket suffix; re-key |
| Partition too large to read | No bounding component in the partition key | Time or sequence bucketing |
| Reads get slower over time | Tombstone accumulation from deletes/TTL | Avoid delete-heavy patterns; tune compaction; use TTL at table level |
| Query requires a full scatter | Filter not on partition key | Create a query-specific table |
| Duplicated tables diverge | Application writes one and not the other | Write through a single path; reconciliation job |
| Lightweight transactions are slow | Consensus round trip per operation | Reserve for rare uniqueness checks; redesign the hot path |
| Repair overwhelms the cluster | Anti-entropy repair unthrottled | Schedule and throttle repairs; incremental repair |

**Limits**

> **Design targets**
>
> - **Partition size**: aim for under ~100 MB and under a few hundred thousand rows; investigate anything larger.
> - **Replication factor 3** with quorum reads and writes is the standard baseline.
> - **Write path**: no read required, so throughput is bounded by IO and commit log, not by contention.
> - **Tombstone thresholds** are configurable and will warn or fail queries — treat warnings as design feedback.
> - **Partition count** should vastly exceed node count so rebalancing is fine-grained.

**Alternatives**

| Store | Better when | Trade |
|---|---|---|
| Wide-column | High-volume, partition-scoped, ordered access | Fixed access patterns; duplication |
| Key-value | Point lookups only | No ordering or ranges |
| Relational | Rich queries, constraints, transactions | Harder to scale horizontally |
| Time-series database | Metrics with downsampling and retention built in | Narrower data model |
| Columnar warehouse | Aggregation across everything | Not for point writes or low latency |

Purpose-built time-series databases deserve consideration whenever the workload is genuinely metrics: they provide downsampling, retention tiers and aggregation functions that you would otherwise build on top of a wide-column store.

**In real systems**

- **Cassandra and ScyllaDB** are the canonical implementations; their modelling guidance is essentially “one table per query, bound your partitions.”
- **Time-series and IoT platforms** use partition-per-device-per-period as the standard shape, precisely to bound partition growth.
- **Messaging and feed systems** partition by conversation or user with a time-ordered clustering key, which makes “recent N” a sequential read.
- **HBase and Bigtable** offer ordered partitioning by row key range rather than by hash, trading even distribution for range scans across entities.
- **Discord's message storage migration** is a widely cited example of partition sizing and tombstone behaviour driving a re-modelling effort.

**Common mistakes**

- **Partition keys with no bounding component**, producing partitions that grow forever.
- **Ignoring key distribution**, creating hot partitions on day one.
- **Designing tables before enumerating access patterns.**
- **Using engine secondary indexes** and getting a hidden cluster-wide scatter.
- **Delete- or TTL-heavy workloads** that accumulate tombstones and degrade reads.
- **Read-before-write**, discarding the write-path advantage.
- **Using lightweight transactions on a hot path** and paying consensus latency per operation.

**The staff-level view**

Wide-column is a high-performance tool with a narrow contract, and most of the value a Staff engineer adds is in enforcing that contract at design time.

- **Require a written access-pattern list before any table is created.** Every late query is a new table plus a backfill, so the cost of omission is measured in weeks.
- **Review every partition key for both bounding and distribution.** These are two separate checks and teams routinely do only one.
- **Ban engine secondary indexes in hot paths** in favour of explicit query tables, so the scatter cost is visible rather than hidden.
- **Treat delete-heavy access patterns as a red flag**, because tombstones make reads progressively worse in a way that looks like a mystery regression.
- **Ask whether the workload actually needs this.** A single relational node plus read replicas covers far more workloads than teams assume, and it preserves query flexibility that is expensive to give up.

**Go deeper**

Wide-column storage is a distributed map of ordered maps. The partition key is hashed to place data on a node; the clustering key orders rows inside that partition so range reads are sequential. Every query must supply the partition key, which makes the primary key design the entire query plan.

Consequently you design a table per access pattern rather than a table per entity, writing the same data several times and keeping the copies consistent in the application. Storage is cheap; cross-partition coordination is not. Writes are exceptionally fast because they append without reading, and consistency is tunable per operation — quorum reads and writes are the standard choice because W + R > RF guarantees overlap.

Three failure modes dominate. A partition key with no bounding component produces partitions that grow forever until one node cannot hold or scan them; add a time bucket. A skewed distribution produces hot partitions; add a hash bucket suffix and fan out on read. And delete- or TTL-heavy workloads accumulate tombstones that reads must scan past, so read latency degrades steadily in a way that looks like an unexplained regression. All three are modelling problems that adding nodes does not fix.

Wide-column storage sits between key-value and relational: it scales horizontally like the former while supporting ordered range reads like the latter — but only along the single axis chosen at table-design time.

**The primary key is the query plan.** `PRIMARY KEY ((partition_key), clustering_key...)` says which node holds the data and how rows are ordered within it. Queries must specify the partition key for equality and may add a clustering prefix or range. Anything else scatters across every node and is, in practice, unsupported. This inverts relational modelling: rather than designing entities once and writing many queries, you enumerate the queries first and create a table for each, accepting that the same data is written several times and that the application keeps the copies aligned. That duplication is the design working correctly, because storage is cheap and coordination is not.

**Two independent partition-key checks.** Bounding asks whether the partition can grow without limit — all readings for a sensor, all messages in a room — and the fix is a time or sequence component, `(sensor_id, day)`, giving partitions a maximum size. Distribution asks whether load is even, and the fix for a heavy-headed key is a hash bucket suffix so one hot entity spreads across several partitions, paying a fan-out and merge on read. Teams routinely perform one check and not the other, and retrofitting either requires re-keying and backfilling the whole table.

**Tombstones are the non-obvious operational trap.** Deletes and TTL expiries are writes: they insert a marker that reads must scan past until compaction removes it, and compaction cannot remove it before a grace period long enough to prevent deleted data resurrecting from an un-repaired replica. A workload that writes and deletes the same partition repeatedly — a queue implemented as a table is the archetype — accumulates tombstones faster than compaction clears them, and read latency degrades progressively. The remedy is design-level: avoid delete-heavy patterns, prefer table-level TTL, or bucket partitions so that expiry drops whole partitions rather than individual rows.

**Consistency is a per-query decision.** With replication factor three, quorum reads and writes give overlap (W + R > RF) and therefore read-your-writes, while tolerating one node loss — the sensible default. Lower levels trade staleness for latency and availability and are appropriate for analytics or best-effort reads; requiring all replicas makes any node loss block the operation and is rarely worth it. Lightweight transactions provide compare-and-set via a consensus round trip and are an order of magnitude more expensive, so they belong in rare uniqueness checks rather than on hot paths.

**Choosing it at all.** The qualifying conditions are high write volume genuinely beyond a single relational primary, and access patterns that are stable and partition-scoped. Both must hold. A relational primary with read replicas sustains tens of thousands of writes per second and preserves constraints, joins and query flexibility, all of which are expensive to surrender — and a transactional business domain accumulates new query shapes continuously, which is exactly what this model punishes. Time series, event logs, feeds and telemetry qualify; an order management system usually does not. When a cluster degrades, the first question is almost never capacity: check partition skew, partition size, tombstone accumulation and repair currency first, because three of those four are modelling problems that adding nodes only postpones.

**Prove it — interview questions**

1. **[Basic] What do the partition key and clustering key each do?**

   <details><summary>Model answer</summary>

   The partition key is hashed to decide which node holds the data, so every query must specify it and its distribution determines whether load is even. The clustering key orders rows within that partition, so reading a range along it is a sequential read on one node. Together they define the only efficient query shape: partition key equality, plus optionally a prefix or range of the clustering key.

   </details>

2. **[Basic] Why would you store the same data in two tables?**

   <details><summary>Model answer</summary>

   Because a table serves one query shape. If I need messages by room and also messages by user, those require different partition keys, so they are different tables holding the same rows written twice. That duplication is the model working as designed — storage is cheap and cross-partition coordination is not — but it does mean the application is responsible for keeping the copies consistent, ideally by writing through a single code path.

   </details>

3. **[Senior] How do you prevent hot and unbounded partitions?**

   <details><summary>Model answer</summary>

   They are two separate problems needing two fixes. Unbounded growth is solved by adding a bounding component to the partition key — a day or month bucket — so each partition has a maximum size, with queries specifying which buckets to read. Skew is solved by adding a hash-based bucket suffix so one hot entity spreads across several partitions, at the cost of fanning out and merging on read. I would check both at design time, because retrofitting either one means re-keying and backfilling the entire table.

   </details>

4. **[Senior] Why do tombstones matter?**

   <details><summary>Model answer</summary>

   Because deletes are writes. A delete inserts a tombstone marking the row as removed, and reads must scan past tombstones until compaction removes them, which happens only after a grace period long enough to prevent deleted data resurrecting on a repaired replica. A partition that is frequently written and deleted — a queue implemented as a table is the classic example — accumulates tombstones faster than compaction clears them, and reads get progressively slower for reasons that look like an unexplained regression. The fix is to avoid delete-heavy access patterns entirely, preferring table-level TTL or bucketed partitions that are dropped whole.

   </details>

5. **[Staff] When would you choose wide-column over a relational database with read replicas?**

   <details><summary>Model answer</summary>

   When write volume genuinely exceeds what a single primary can sustain, and the access patterns are stable and partition-scoped. Both conditions matter. A relational primary with replicas handles tens of thousands of writes per second and preserves query flexibility, constraints and transactions — all of which are expensive to give up — so I would want evidence that we are approaching that ceiling rather than anticipating it. Equally, I would want the access patterns to be genuinely stable, because every new query in a wide-column system is a new table plus a backfill. Time series, event logs, feeds and telemetry qualify on both counts. A transactional business domain, which accumulates new query shapes continuously and has cross-entity invariants, usually does not.

   </details>

6. **[Principal] A team's wide-column cluster is degrading and they want to add nodes. How do you approach it?**

   <details><summary>Model answer</summary>

   Adding nodes helps only if the load is evenly distributed, so the first question is whether the problem is capacity or skew. I would look at per-node and per-partition metrics: if a small number of partitions dominate, more nodes change nothing and the fix is re-keying with a bucket suffix. Second, I would check partition sizes and tombstone counts, because a cluster whose reads are slowing steadily is usually suffering from unbounded partitions or delete accumulation rather than from insufficient hardware, and both are modelling problems that adding capacity merely postpones. Third, I would review whether repair and compaction are keeping up, since an under-repaired cluster degrades in ways that look like load. Only if distribution is even, partitions are healthy and maintenance is current would I conclude it is a capacity problem — and in my experience that is the least common of the four.

   </details>

---

### Object storage

*Immutable blobs addressed by key over HTTP, with effectively unlimited capacity — and a metadata problem you must solve outside it.*

**Flow:** `Upload intent` → `Object bytes` → `Verification` → `Metadata publish` → `Retrieval`

> **The 30-second version**  
> Immutable blobs addressed by key over HTTP: unlimited capacity, very low cost per byte, no queries and no in-place edits — so keep authoritative metadata in a database.

**The problem**

Block storage forces you to decide capacity in advance, attaches to one machine, and costs 20–50× more per byte than object storage. File systems add the cost of a shared namespace — directory locks, POSIX semantics, metadata servers — that becomes a bottleneck long before capacity does.

Object storage drops all of it: no hierarchy (only key prefixes that look like one), no partial writes, no locks, no POSIX. An object is written whole, read whole or by byte range, and addressed by a key over HTTP. The result scales to exabytes because nothing in the design requires coordination.

> **The trade in one line**  
> You give up in-place mutation, low latency, and a filesystem's semantics. You get durability measured in eleven nines, capacity you never provision, and a price per byte that makes long retention economically possible.

**Mental model**

Treat every object as immutable. You do not edit an object; you write a new one. Keys look like paths but there are no directories — `a/b/c.jpg` is a flat key whose slashes are only a listing convention.

1. **Object** — Bytes plus metadata, identified by a key within a bucket. Written whole via a single PUT or a multipart upload.
2. **Bucket** — A namespace and a policy boundary: region, access control, lifecycle rules, versioning, encryption.
3. **Key** — A flat string. Prefixes enable listing and lifecycle rules; they are not directories, and listing by prefix is the only query you get.
4. **Storage class** — Hot, infrequent-access, archive. Same API, wildly different price and retrieval latency.
5. **Lifecycle policy** — Rules that transition or expire objects by age or prefix — the primary cost lever in any storage-heavy system.

> **Object storage is not a database**  
> There is no query beyond “list keys with this prefix.” Listing is paginated, eventually ordered, and slow at scale. Every system built on object storage needs a **separate metadata store** — a database holding the key, size, checksum, owner, and status — that is the source of truth for what exists. Treating the bucket listing as your index is the most common architectural mistake here.

**How it works**

**The upload pattern that actually works**

```text
NAIVE (do not do this)
  client -> API server (buffers the whole file) -> object storage
  - API server memory and bandwidth scale with upload size
  - a 5 GB upload ties up an application process

PRESIGNED / DIRECT UPLOAD
  1  client -> API: "I want to upload, 2 GB, video/mp4"
  2  API: create metadata row {id, status=pending, owner, size}
     API -> client: presigned PUT url (expires in 15 min)
  3  client -> object storage: PUT bytes directly
  4  client -> API: "done, here is the etag/checksum"
  5  API: verify size + checksum, set status=ready
     (or: object storage event notification triggers verification)

WHY STEP 5 MATTERS
  the object may exist while the metadata says pending, or
  the client may vanish after step 3. The metadata row's
  status is the source of truth; orphan objects are cleaned
  by a lifecycle rule on pending prefixes.
```

1. **Never proxy large uploads through your application** — Presigned URLs let clients talk to storage directly, which removes bandwidth, memory and connection-duration pressure from your servers entirely.
2. **Make the metadata store authoritative** — An object existing in the bucket does not mean it is valid. Status transitions — pending, ready, deleted — live in a database, and the bucket is just where the bytes are.
3. **Use multipart upload for anything large** — It gives parallelism, resumability after a failure, and per-part checksums. Also set a lifecycle rule to abort incomplete multipart uploads, or you pay storage for parts nobody will ever complete.
4. **Prefer content addressing where it fits** — Naming an object by the hash of its contents gives free deduplication, makes uploads idempotent, and makes caching trivially safe since the key never changes meaning.
5. **Design key prefixes for lifecycle and access, not for humans** — Prefixes drive lifecycle rules and access policies. A layout like `tenant/{id}/raw/{yyyy}/{mm}/` lets you expire, tier or restrict whole classes of data with one rule.
6. **Lean on lifecycle tiering hard** — Moving data to infrequent-access and archive classes by age is usually the single largest cost reduction available in a storage-heavy system, and it is a configuration change.

**Cost is a retention decision, not a storage decision**

```text
1 PB of data, illustrative relative costs per month:

  hot standard          1.0x          instant access
  infrequent access     0.55x         instant, retrieval fee
  archive               0.10x         minutes-hours to restore
  deep archive          0.04x         hours to restore

LIFECYCLE RULE
  0-30 days    hot        (recent content, actively served)
  30-90 days   infrequent (occasional access)
  90-365 days  archive    (compliance, rare restore)
  >365 days    delete OR deep archive

RESULT: often a 60-80% reduction in storage spend for a
configuration change and a conversation about retention.

WATCH: retrieval fees and minimum storage durations can make
tiering small, frequently-read objects MORE expensive.
```

> **Deletion is harder than it looks**  
> With versioning enabled, a delete writes a delete marker and the old versions remain — and keep costing money — until a lifecycle rule expires non-current versions. Teams enable versioning for safety, delete data for compliance, and discover years later that nothing was actually removed. If you must be able to prove deletion, verify it explicitly, and remember that backups and replicas are separate copies with their own lifecycles.

**Worked example**

A video platform storing originals and transcoded renditions. Where the cost and the correctness problems actually are.

**Layout, lifecycle and the metadata contract**

```text
KEY LAYOUT
  v1/originals/{video_id}                 <- write once, rarely read
  v1/renditions/{video_id}/{profile}.m3u8 <- served constantly
  v1/uploads/pending/{upload_id}          <- ephemeral

METADATA (in the database, authoritative)
  videos(id, owner, status, duration, original_key,
         original_sha256, created_at)
  renditions(video_id, profile, key, bytes, ready_at)

LIFECYCLE RULES
  v1/uploads/pending/   expire after 24h
                        abort incomplete multipart after 7d
  v1/originals/         -> infrequent at 30d, archive at 180d
  v1/renditions/        stay hot (served via CDN)

THE COST MATH (1 PB originals, 300 TB renditions)
  all hot:                        1 PB x 1.0 + 0.3 PB x 1.0
  with tiering on originals:      1 PB x ~0.12 + 0.3 PB x 1.0
  -> roughly a 65% reduction in total storage spend

THE CORRECTNESS RULE
  a video is "ready" only when the DB says so.
  an object present in the bucket with no metadata row is
  garbage, collected by the pending lifecycle rule.
```

| Metric | Value | Note |
|---|---|---|
| Originals | 1 PB | tiered to archive |
| Renditions | 300 TB | hot, CDN-fronted |
| Saving | ~65% | **config change** |
| Authority | database | not the bucket |

> **Why originals can be archived and renditions cannot**  
> Originals are written once and read only when re-transcoding is needed — a rare, tolerant-of-latency operation. Renditions are on the serving path. This distinction, access frequency versus latency sensitivity, is what determines the storage class, and noticing it is usually worth more than any engineering optimisation in a media system.

**When to use it**

- **Large binary content**: images, video, audio, documents, backups, model artifacts, build outputs.
- **Data lakes and analytical storage**, where columnar files are read by query engines directly.
- **Long retention**, where archive tiers make years of data affordable.
- **Static asset serving**, fronted by a CDN.
- **Anything write-once, read-many** where objects are naturally immutable.

**When to avoid it**

- **Do not use it for low-latency random access** — tens of milliseconds per request, not microseconds.
- **Do not use it for frequently mutated data**; there is no partial update, so a small change rewrites the whole object.
- **Do not use bucket listing as an index.** It is slow, paginated, and does not answer questions about state.
- **Do not proxy large uploads or downloads through application servers.**
- **Do not store many tiny objects** — per-object overhead, request cost and listing cost dominate. Pack them.

**Advantages**

- **Effectively unlimited capacity** with no provisioning decision.
- **Extremely high durability** through automatic multi-device, often multi-zone, replication.
- **Cost per byte 20–50× lower than block storage**, with archive tiers cheaper still.
- **HTTP-native**, so it integrates with CDNs, browsers and signed-URL access control directly.
- **Server-side features** — encryption, versioning, lifecycle, replication, object lock — that you would otherwise build.

**Disadvantages**

- **No partial updates**; any change rewrites the object.
- **Latency in the tens of milliseconds**, far above block storage or a cache.
- **No query capability** beyond prefix listing, which forces a separate metadata store.
- **Per-request costs** make many-small-object workloads expensive.
- **Eventual behaviours** in listing and cross-region replication, even where object reads are strongly consistent.
- **Egress charges** can dominate the bill for read-heavy public content.

**Trade-offs**

**Storage options compared**

|  | Object | Block | File (NFS/SMB) |
|---|---|---|---|
| Access | Whole object or byte range via HTTP | Random block, attached to one host | POSIX, shared |
| Latency | Tens of ms | Sub-ms | Low ms |
| Capacity | Unlimited, unprovisioned | Provisioned per volume | Provisioned |
| Cost per TB | Lowest; archive lower still | Highest | High |
| Mutation | Rewrite whole object | In place | In place |
| Concurrency | Last writer wins | Filesystem semantics | Locking protocols |
| Scaling | Automatic | Vertical per volume | Metadata server bottleneck |

**How it fails**

**Object storage failures**

| Failure | Cause | Fix |
|---|---|---|
| Orphaned objects accumulate | Upload succeeded, metadata write failed | Metadata is authoritative; lifecycle rule expires pending prefixes |
| Bill grows unexpectedly | Incomplete multipart uploads, old versions, no tiering | Abort-incomplete rule, non-current version expiry, lifecycle tiering |
| Deleted data still exists | Versioning retains non-current versions | Expire non-current versions; verify deletion explicitly |
| Listing times out | Millions of keys under one prefix | Partition prefixes; keep an index in a database |
| Tiny-object workload is expensive | Per-request and per-object overhead dominates | Pack small objects into larger files |
| Upload fails at 90% repeatedly | Single PUT without multipart on an unreliable network | Multipart upload with resumable parts |
| Public data leak | Bucket or object ACL misconfiguration | Block public access at the account level; presigned URLs for sharing |

> **Presigned URL scope**  
> A presigned URL grants whoever holds it the exact permission it encodes, for its lifetime, with no further authentication. Keep expiry short, scope to one key and one method, and never use them for operations whose replay is harmful. Long-lived presigned URLs leak into logs, chat messages and browser histories, and they cannot be revoked individually.

**Limits**

> **Numbers worth knowing**
>
> - **Durability**: typically 11 nines for standard classes; availability is the lower number and the one that affects you.
> - **Object size**: single PUT commonly capped around 5 GB; multipart extends to terabytes.
> - **Latency**: tens of milliseconds first-byte for standard classes; archive restores take minutes to hours.
> - **Cost ratio**: archive roughly 10× cheaper than hot storage; block storage roughly 20–50× more expensive than hot object.
> - **Minimum storage duration** applies to colder tiers — tiering an object that is deleted next week can cost more than leaving it hot.

**Alternatives**

| Option | When | Trade |
|---|---|---|
| Object storage | Large immutable content, long retention | Latency; no query |
| Block storage | Databases, low-latency random IO | Cost; provisioning; single-host attachment |
| Managed file storage | Legacy apps needing POSIX | Cost; metadata scaling limits |
| CDN in front of object storage | Read-heavy public content | Cache invalidation; still pays origin egress on misses |
| Database BLOB column | Small, transactional binaries | Bloats backups and replication |

The last row is a common mistake worth naming: storing images in a database column couples binary size to backup duration, replication bandwidth and buffer cache efficiency. Keep bytes in object storage and the reference in the row.

**In real systems**

- **Media platforms** store originals in archive tiers and serve renditions from hot storage behind a CDN — the cost structure of the entire business.
- **Data lakes** keep Parquet files in object storage queried directly by engines like Spark, Trino and DuckDB, decoupling storage from compute.
- **Backup and disaster recovery** rely on object lock and versioning for immutability against ransomware and accidental deletion.
- **Container registries and package repositories** are object stores with a metadata database in front — exactly the architecture described here.
- **S3's move to strong read-after-write consistency** removed a long-standing source of subtle bugs in pipelines that wrote then immediately read.

**Common mistakes**

- **Using bucket listing as an index** instead of a metadata database.
- **Proxying large uploads through application servers.**
- **No lifecycle rule to abort incomplete multipart uploads**, silently paying for orphaned parts.
- **Enabling versioning without expiring non-current versions**, so deletes never reduce cost.
- **Storing millions of tiny objects** where per-request cost dominates.
- **Long-lived, broadly scoped presigned URLs.**
- **Assuming an object's presence means the record is valid**, when the metadata row says otherwise.

**The staff-level view**

In storage-heavy systems the architecture decisions are minor compared with the retention decisions. That is where a Staff engineer should spend attention.

- **Make lifecycle policy a design artifact**, not an afterthought. Retention and tiering usually dominate cost by an order of magnitude over anything in the code.
- **Insist the metadata store is authoritative**, with objects treated as content-addressed bytes. This single rule eliminates most consistency bugs in these systems.
- **Mandate presigned direct upload and download.** Proxying bytes through application servers is a scaling ceiling teams hit repeatedly.
- **Audit for incomplete multipart uploads and non-current versions.** Both are invisible, both cost money, and neither appears in any dashboard by default.
- **Treat deletion as a verifiable requirement**, not an API call, wherever compliance is involved — versioning, replication and backups each retain copies independently.

**Go deeper**

Object storage stores immutable blobs addressed by a flat key, written whole and read whole or by byte range over HTTP. It has no hierarchy, no locking and no partial updates, which is exactly why it scales to exabytes and costs 20–50× less per byte than block storage. Keys that look like paths are only a listing and lifecycle convention.

Two architectural rules follow. First, a separate metadata database must be authoritative: the only query available is prefix listing, which is slow and says nothing about state, so the database holds key, size, checksum, owner and status, and an object without a metadata row is garbage rather than data. Second, clients should upload and download directly via short-lived presigned URLs with multipart for large files — proxying bytes through application servers is a scaling ceiling teams hit repeatedly.

Cost is dominated by retention rather than by engineering. Lifecycle rules that tier by age — hot, infrequent access, archive — routinely cut storage spend by more than half as a configuration change. The invisible costs to audit are incomplete multipart uploads and non-current versions left by versioning, neither of which appears on a default dashboard, and both of which mean data you believe you deleted is still being paid for.

Object storage is defined by immutability and the absence of a namespace. An object is written whole, never edited in place, and addressed by a flat key; the slashes in that key are a listing convention, not directories. Removing hierarchy, locking and partial writes removes every reason for nodes to coordinate, which is what buys unlimited capacity, eleven-nines durability and a price per byte that makes decade-long retention economically possible.

**It is not a database, and the consequences are structural.** The only query is prefix listing: paginated, slow at scale, and silent about state. Therefore every system built on object storage needs a metadata store — a database row per object holding key, size, checksum, owner and status — and that row, not the object's presence, is the source of truth. This single rule eliminates most of the consistency bugs in these systems: an object with no metadata row is garbage to be collected, and a metadata row in `pending` state means the bytes may exist but are not yet valid.

**The upload path is where designs fail.** Proxying uploads through application servers ties process memory, bandwidth and connection duration to file size, which becomes a ceiling quickly. The working pattern is: the API creates a pending metadata row and returns a short-lived presigned URL; the client uploads directly using multipart, which gives parallelism and resumability; the API then verifies size and checksum from a callback or a storage event and marks the row ready. Clients that vanish mid-upload leave orphaned parts, so a lifecycle rule aborting incomplete multipart uploads is mandatory — those parts are billed and are invisible in normal listings.

**Cost is a retention conversation, not an engineering one.** Lifecycle tiering by age — hot for content on the serving path, infrequent access for occasional reads, archive for compliance copies — routinely reduces storage spend by 60–80% as a configuration change. Two caveats: colder tiers carry retrieval fees and minimum storage durations, so tiering small, frequently read objects can cost more, not less. And versioning, usually enabled for safety, retains non-current versions after a delete until an expiry rule removes them, which means teams enable versioning, delete data for compliance, and discover years later that nothing was removed and everything was billed.

**Deletion deserves particular care.** Where compliance requires it, deletion is a verifiable outcome rather than an API call, because several mechanisms retain independent copies: non-current versions, cross-region replication into a bucket with its own lifecycle, backups and snapshots, and object lock, which may legally prevent deletion for its retention period — a conflict best discovered at design time rather than at audit. The implementation is a pipeline covering every copy plus an automated job that asserts absence and reports it.

**Organisationally**, storage is the one resource where doing nothing costs money forever. The durable fix is that every bucket or prefix has a named owner and a stated retention policy before it is provisioned, with the platform default being a short expiry that teams must consciously extend rather than indefinite retention; cost attribution reaching the team that generated the data; and continuous audits for incomplete multipart uploads, stale versions and objects sitting in a class more expensive than their access pattern justifies, surfaced as tickets rather than as an annual cleanup.

**Prove it — interview questions**

1. **[Basic] Why is object storage cheaper than block storage?**

   <details><summary>Model answer</summary>

   Because it offers far less. There is no in-place mutation, no POSIX semantics, no locking, no attachment to a host, and no provisioning — so it needs no coordination between nodes and can use commodity hardware with erasure coding across many devices. Every capability it omits is one that would require synchronisation, and synchronisation is what makes storage expensive to scale.

   </details>

2. **[Basic] Why do you need a separate metadata store?**

   <details><summary>Model answer</summary>

   Because the only query object storage offers is “list keys with this prefix,” which is paginated, slow at scale, and says nothing about state. A database holding key, size, checksum, owner and status answers the questions the application actually asks and, crucially, is the authority on whether an object is valid. An object present in the bucket with no metadata row is garbage, not data — and that distinction is what makes upload flows reliable.

   </details>

3. **[Senior] Describe a reliable large-file upload flow.**

   <details><summary>Model answer</summary>

   The client asks the API for permission; the API creates a metadata row with status pending and returns a short-lived presigned URL. The client uploads directly to object storage using multipart, which gives parallelism and resumability. On completion the API verifies size and checksum — either from a client callback or from a storage event notification — and flips the status to ready. The metadata row is authoritative throughout, so a client that vanishes mid-upload leaves a pending row and orphaned parts, which a lifecycle rule aborting incomplete multipart uploads cleans up. The key property is that application servers never touch the bytes.

   </details>

4. **[Senior] How would you reduce the cost of a petabyte-scale store?**

   <details><summary>Model answer</summary>

   I would start with lifecycle policy rather than code, because it usually dominates. Classify data by access frequency and latency tolerance: content on the serving path stays hot, source material read only for reprocessing moves to infrequent access and then archive, and anything past the retention requirement is deleted. Then I would audit the invisible costs — incomplete multipart uploads and non-current versions from versioning, neither of which appears on a dashboard by default and both of which can be a large fraction of the bill. I would also check object size distribution, because millions of tiny objects cost more in per-request and per-object overhead than in bytes, and packing them into larger files can be a significant saving.

   </details>

5. **[Staff] A compliance requirement says user data must be deleted within 30 days. What do you actually have to do?**

   <details><summary>Model answer</summary>

   Treat deletion as a verifiable outcome rather than an API call, because several mechanisms retain copies independently. With versioning enabled, a delete writes a marker and previous versions persist until a non-current-version expiry rule removes them. Cross-region replication has produced a second copy in another bucket with its own lifecycle. Backups and snapshots are separate copies entirely. Object lock, if enabled for ransomware protection, may legally prevent deletion until its retention period expires — which is a direct conflict to resolve at design time rather than at audit time. So the implementation is: a deletion pipeline that covers every copy, lifecycle rules that expire non-current versions, and an automated verification job that asserts absence across all locations and reports it, because the requirement is to prove deletion, not to have attempted it.

   </details>

6. **[Principal] How do you keep storage cost from growing unbounded across many teams?**

   <details><summary>Model answer</summary>

   By making retention an explicit, owned decision at the point data is created. Every bucket or prefix needs a named owner and a stated retention and tiering policy before it is provisioned, enforced by the platform rather than by review — the default for a new prefix should be a short expiry that teams must consciously extend, not indefinite retention. Alongside that, cost attribution has to reach the team that generated the data, because storage growth is invisible to whoever creates it when the bill lands centrally. And I would run continuous audits for the specific invisible costs — incomplete multipart uploads, non-current versions, objects in a class more expensive than their access pattern justifies — surfaced as tickets to owners rather than as a quarterly cleanup project. The structural insight is that storage is the one resource where doing nothing costs money forever, so the defaults must expire rather than persist.

   </details>

---

### Time-series data modeling

*Append-only measurements ordered by time, where retention, rollup and cardinality decide cost and queryability more than anything else.*

**Flow:** `Event time` → `Time bucket` → `Raw samples` → `Rollup` → `Retention`

> **The 30-second version**  
> Series key plus ordered samples. Cost scales with series count, not samples, so cardinality discipline, rollup tiers and a lateness policy are the whole design.

**The problem**

Time-series data has a shape that breaks general-purpose stores. It is written constantly and never updated, it is queried almost exclusively as ranges, its value decays sharply with age, and its volume is determined by a cross-product — number of series × sample rate × retention — that grows faster than anyone expects.

Store it naively in a relational table and you get a table with billions of rows, an index larger than the data, and a query that scans a year to draw a one-hour chart. Store it naively in a metrics system and you get a cardinality explosion that takes the monitoring platform down — often during the incident you were trying to observe.

> **The three ways time-series systems fail**
>
> - **Cardinality explosion**: adding a high-variability label (user id, request id, container id) multiplies series count by its cardinality, and memory grows with series, not with samples.
> - **Unbounded retention**: keeping raw resolution forever costs linearly in storage and in query time, for data nobody reads at that resolution.
> - **Late and out-of-order data**: events arriving after their window closed silently corrupt aggregates unless the design accounts for them.

**Mental model**

A time series is identified by a **series key** — a metric name plus a set of labels — and contains an ordered sequence of (timestamp, value) pairs. Total cost is series count × samples per series, and the two scale very differently: samples are cheap and compress well; series are expensive because each carries an index entry and in-memory state.

1. **Series key** — Metric name plus labels. Every distinct combination is a separate series. This is where cardinality lives.
2. **Sample** — Timestamp plus value. Highly compressible — delta-of-delta on timestamps, XOR on floats — often under two bytes each.
3. **Resolution** — How often samples are taken. Determines raw volume and the finest question you can ever ask.
4. **Rollup** — Pre-aggregated lower-resolution copies (1m → 5m → 1h → 1d), which is how old data stays queryable and affordable.
5. **Retention** — How long each resolution is kept. Usually tiered: raw for days, coarse for years.

> **Cardinality is the dimension that kills you**  
> Ten metrics with five labels of ten values each is 10 × 10⁵ = one million series. Add a label with 10,000 values — user id, pod name, request path with ids in it — and you have ten billion. Sample volume grows linearly with time; series count grows multiplicatively with label choices, and it is bounded only by discipline.

**How it works**

**Where the volume comes from**

```text
VOLUME = series_count x samples_per_series

series_count = product of label cardinalities, per metric
  http_requests{method(5), status(8), endpoint(50), pod(200)}
    = 5 x 8 x 50 x 200 = 400,000 series for ONE metric

samples_per_series = retention / resolution
  15s resolution, 30 days = 172,800 samples

raw volume = 400,000 x 172,800 = 69 billion samples
  at ~2 bytes compressed  = ~138 GB for one metric

REMOVE ONE LABEL (pod):
  2,000 series -> 345 million samples -> ~0.7 GB
  a 200x reduction from one modelling decision.
```

1. **Choose labels by the queries you will run** — A label exists so you can group or filter by it. If you will never `GROUP BY pod`, do not make pod a label — put it in logs or traces where high cardinality belongs.
2. **Never put unbounded values in labels** — User ids, request ids, email addresses, full URLs with path parameters, error messages. Each one turns a metric into an index of your entire user base.
3. **Bucket time explicitly in general-purpose stores** — If you must use a relational or wide-column store, partition by time — `(series_id, day)` — so old partitions can be dropped whole rather than deleted row by row.
4. **Roll up aggressively and early** — Keep raw for days, 1-minute for weeks, 1-hour for months, 1-day for years. Nobody debugs a six-month-old incident at 15-second resolution.
5. **Decide the lateness policy up front** — Define a watermark — how long a window stays open for late events — and what happens to arrivals after it: drop, reprocess, or route to a correction path.
6. **Prefer dropping whole partitions to deleting rows** — Time-partitioned storage makes retention a metadata operation. Row-by-row deletion creates tombstones and compaction load.

**Rollup and retention tiers**

```text
RESOLUTION   RETENTION   USE
-----------  ----------  --------------------------------
raw (15s)    7 days      incident debugging, alerting
1 minute     30 days     recent trends, weekly review
5 minutes    90 days     capacity planning
1 hour       1 year      seasonal patterns, reporting
1 day        5 years     long-term trends, compliance

STORAGE EFFECT (one series)
  raw only, 5 years : 10.5M samples
  tiered as above   : 40k + 43k + 26k + 8.7k + 1.8k
                    = ~120k samples     -> ~85x less

CAUTION: rollups must preserve what you will ask.
  avg of avgs is WRONG for percentiles.
  store count+sum for averages; store histogram buckets
  for percentiles; store min/max separately.
```

> **Percentiles do not survive naive rollup**  
> You cannot average p99 values across time buckets or across instances — the result is not a percentile of anything. To keep percentiles queryable after rollup, store histogram buckets (or a sketch such as t-digest/HdrHistogram) rather than pre-computed quantiles, and compute the quantile at query time from merged buckets.

**Worked example**

Monitoring 500 services across 5,000 containers. Model the metrics so the platform survives.

**Cardinality budget in practice**

```text
NAIVE LABELS
  service(500) x instance(5000) x endpoint(100)
  x method(5) x status(10)
  = 1.25 BILLION series      -> platform dies

WHAT ACTUALLY GETS QUERIED
  "error rate by service and status"        -> service, status
  "latency by service and endpoint"         -> service, endpoint
  "which instance is the outlier?"          -> service, instance
                                               (but only for a
                                                few key metrics)

BUDGETED DESIGN
  http_requests_total{service, endpoint, method, status}
    500 x 100 x 5 x 10 = 2.5M series
  http_request_duration_bucket{service, endpoint, le}
    500 x 100 x 12 buckets = 600k series
  instance_health{service, instance}
    500 x 5000 = 2.5M series   <- instance label ONLY here

  total ~5.6M series           -> tractable

WHAT MOVED ELSEWHERE
  per-request detail  -> traces (sampled)
  error messages      -> logs (indexed by service+time)
  user-level analysis -> analytics warehouse
```

| Metric | Value | Note |
|---|---|---|
| Naive | 1.25B series | unusable |
| Budgeted | 5.6M series | **200× smaller** |
| Instance label | one metric only | deliberate |
| Detail | traces and logs | right tool |

> **The three-pillar division of labour**  
> Metrics answer “how much, how often, how fast” across aggregates and must stay low cardinality. Traces answer “what happened to this specific request” and are sampled. Logs answer “what exactly did the code say” and are indexed by time plus a coarse key. Nearly every cardinality disaster is a question being asked of the wrong pillar — trying to get per-user or per-request answers out of metrics.

**When to use it**

- **Metrics and monitoring**, the archetypal case.
- **IoT and sensor telemetry**, where volume is high and value decays with age.
- **Financial tick data and trading**, where ordered time ranges are the entire access pattern.
- **Application and business event counters**, aggregated over time windows.
- **Capacity planning and forecasting**, which need long retention at coarse resolution.

**When to avoid it**

- **Do not use it for per-entity records** that must be individually retrieved and updated — that is transactional data.
- **Do not put high-cardinality identifiers in labels.** This is the single most damaging mistake in the topic.
- **Do not keep raw resolution indefinitely** for data nobody queries at that resolution.
- **Do not pre-compute percentiles** if you intend to aggregate or roll them up later.
- **Do not ignore out-of-order arrival**; define the lateness policy before it corrupts aggregates silently.

**Advantages**

- **Extreme compression**: delta-of-delta timestamps and XOR-encoded floats bring samples to a couple of bytes.
- **Append-only writes** with no read-modify-write, giving very high ingest throughput.
- **Retention is a partition drop**, not a delete storm, when time bucketing is used.
- **Range queries are sequential reads**, which is the dominant access pattern.
- **Rollups make long history affordable** without losing the ability to see trends.

**Disadvantages**

- **Cardinality is a hard operational limit**, and exceeding it degrades or kills the whole platform, not just one query.
- **Rollups lose information irreversibly** — you cannot recover detail you did not keep.
- **Late data is awkward**: windows must either stay open (memory) or be corrected (complexity).
- **Not a general query engine**: joins, arbitrary filters and per-entity lookups are not what it does.
- **Label schema is effectively permanent**; changing labels breaks dashboards, alerts and historical continuity.

**Trade-offs**

**Where to put the data**

| Store | Strength | Weakness |
|---|---|---|
| Purpose-built TSDB (Prometheus, InfluxDB) | Compression, rollups, retention, query language | Cardinality limits; not for per-entity records |
| Wide-column (Cassandra) | Huge ingest, partition-scoped ranges | You build rollups and retention yourself |
| Relational with time partitioning | Familiar; joins with other data | Volume ceiling; index overhead |
| Columnar warehouse | Arbitrary analytical queries over history | Not for low-latency dashboards |
| Object storage + query engine | Cheapest long-term retention | Higher query latency |

A common mature architecture uses a TSDB for recent, high-resolution operational data and ships rolled-up history to a columnar store or object storage for long-term analysis — hot for alerting, cheap for trends.

**How it fails**

**Time-series failures**

| Failure | Cause | Fix |
|---|---|---|
| Monitoring platform OOMs | Cardinality explosion from a new label | Cardinality limits per metric; reject or drop offending series; review label changes |
| Dashboards slow to a crawl | Querying raw resolution over long ranges | Query rollups for long ranges; enforce range/resolution rules |
| Percentiles are wrong after rollup | Averaging pre-computed quantiles | Store histogram buckets or sketches; compute at query time |
| Aggregates silently incorrect | Late-arriving events after the window closed | Explicit watermark and a correction path |
| Deletes cause compaction storms | Row-level retention deletion | Time-partitioned storage; drop whole partitions |
| Cannot answer a new question about history | Rollup discarded the needed dimension | Keep a longer raw window for critical metrics; archive raw to object storage |
| Alerts flap after a resolution change | Rollup boundaries alter the signal shape | Align alert windows to rollup boundaries; test alerts against rolled-up data |

**Limits**

> **Numbers to plan with**
>
> - **Compressed sample size**: roughly 1–2 bytes with good encoding.
> - **Series memory**: in-memory TSDBs need on the order of kilobytes of overhead per active series — series count, not sample count, drives RAM.
> - **Practical cardinality**: single-node Prometheus-style systems become difficult past a few million active series.
> - **Rollup savings**: tiered retention typically reduces long-term volume by one to two orders of magnitude.
> - **Watermark**: a few minutes of lateness tolerance is common; longer means more open windows and more memory.

**Alternatives**

| Question | Right tool | Why not metrics |
|---|---|---|
| How many, how fast, trend over time | Time series | — |
| What happened to request X | Distributed trace | Per-request cardinality is unbounded |
| What did the code say | Logs | Message text is unbounded |
| Per-user behaviour analysis | Analytics warehouse | User id cardinality is unbounded |
| Current state of entity X | Transactional database | Time series is append-only history |

**In real systems**

- **Prometheus** made the cardinality trade explicit: fast, simple, in-memory index — and a hard practical ceiling on active series.
- **Gorilla (Facebook)** introduced the delta-of-delta and XOR compression that most modern time-series engines now use.
- **Downsampling in Thanos, Cortex and Mimir** exists specifically to make long retention affordable while keeping recent data at full resolution.
- **IoT platforms** partition by device and time period, dropping whole partitions for retention rather than deleting rows.
- **Financial tick stores** keep full resolution because the value does not decay — the rare case where aggressive rollup is wrong.

**Common mistakes**

- **Adding a high-cardinality label** such as user id, request id or pod name to a widely used metric.
- **Averaging percentiles** across time or instances.
- **Keeping raw resolution forever** because deleting felt risky.
- **Deleting old rows individually** instead of dropping time partitions.
- **No lateness policy**, so out-of-order events silently corrupt aggregates.
- **Using metrics to answer per-entity questions** that belong in traces or logs.
- **Changing labels casually**, breaking dashboards, alerts and historical continuity.

**The staff-level view**

Time-series design is mostly governance: deciding what may become a label, and for how long data is kept.

- **Publish a cardinality budget per metric and enforce it in the ingestion path**, rejecting or dropping series beyond the limit rather than letting one bad deploy take the platform down.
- **Make label additions a reviewed change.** A single new label can multiply series count by four orders of magnitude, and it is usually added casually.
- **Route questions to the right pillar.** Most cardinality disasters are someone trying to answer a per-request or per-user question with metrics.
- **Standardise retention tiers** so teams choose from a small menu rather than inventing per-service policies that nobody can reason about in aggregate.
- **Store histogram buckets rather than pre-computed percentiles**, platform-wide. Retrofitting this is painful and the wrong-percentile bug is silent.

**Go deeper**

A time series is a series key — metric name plus labels — with an ordered sequence of timestamped values. Samples compress to roughly a byte or two each using delta-of-delta and XOR encoding, so they are cheap. Series are expensive, because each carries index entries and in-memory state, and series count is the *product* of label cardinalities. Adding one label with ten thousand values multiplies your footprint by ten thousand.

That makes label choice the central design decision: a label should exist only because you will group or filter by it. User ids, request ids, container names and URLs with embedded parameters belong in traces, logs or a warehouse, not in metric labels. Nearly every cardinality incident is a per-request or per-user question being asked of the metrics pillar.

The other two levers are rollup and lateness. Tiered retention — raw for days, one minute for weeks, one hour for months, one day for years — cuts long-term volume by one to two orders of magnitude, but rollups must preserve what you will ask: store histogram buckets rather than pre-computed percentiles, because averaging quantiles produces a number that is not a percentile of anything. And define a watermark for how long a window accepts late events, plus what happens after, or out-of-order arrivals silently corrupt aggregates.

Time-series data has a distinctive shape — append-only, queried as ranges, value decaying with age — and a cost model that surprises people: it is dominated by the number of distinct series rather than by the number of samples.

**Why cardinality dominates.** A sample compresses to one or two bytes with delta-of-delta timestamp encoding and XOR float encoding. A series, by contrast, carries index entries and per-series in-memory state measured in kilobytes. Series count is the product of label cardinalities, so it grows multiplicatively while sample volume grows only linearly with time. A metric with method, status, endpoint and pod labels at 5, 8, 50 and 200 values is 400,000 series; removing the pod label leaves 2,000 — a 200× reduction from one modelling decision. This is why a single casually added label can take down a monitoring platform, and why the failure arrives during a deploy rather than gradually.

**Route the question to the right pillar.** Metrics answer aggregate questions — how many, how often, how fast — and must stay low cardinality. Traces answer “what happened to this specific request” and are sampled precisely because per-request identity is unbounded. Logs answer “what did the code say,” indexed by time and a coarse key. Per-user behavioural analysis belongs in a warehouse. Almost every cardinality disaster is a legitimate question asked of the wrong system, so publishing this routing rule is more effective than asking people to be careful.

**Rollups, and what they destroy.** Tiered retention keeps raw resolution for days, one-minute for weeks, one-hour for months and one-day for years, typically reducing long-term volume by one to two orders of magnitude while preserving every question anyone actually asks of old data. The trap is that aggregation must preserve the statistic you will need: averages require count and sum rather than pre-averaged values, and percentiles cannot be recovered from stored quantiles at all — averaging p99s produces a number that is not a percentile of anything. Store histogram buckets or a mergeable sketch and compute quantiles at query time. This decision has to be made early, because the period before the change has no correct percentiles and never will.

**Time and lateness.** Events arriving after their window has closed will either be silently dropped or will silently mutate aggregates that downstream systems already consumed. Neither is acceptable as an accident. Define a watermark — how long a window stays open — and a policy for arrivals beyond it: drop with a counter, extend the window at the cost of memory, or emit a correction. Always export late-event volume as a metric, because it is the signal that the watermark is set wrong.

**Storage mechanics.** Retention should be a partition drop, not a delete storm: time-bucketed partitioning, whether in a purpose-built TSDB or a wide-column store keyed by `(series, day)`, makes expiry a metadata operation instead of a compaction load. And a mature architecture usually splits the workload — a TSDB for recent high-resolution operational data feeding alerts and dashboards, with rolled-up history shipped to columnar storage or object storage for long-term analysis at a fraction of the cost.

**Governance is the real control.** Cardinality budgets enforced at ingestion, so an offending metric is dropped or rejected with an alert to its owner rather than exhausting shared memory — the failure must be local, because otherwise one team's change blinds everyone during an incident. Label sets reviewed as schema rather than emerging from code. Instrumentation libraries that make histograms easier than quantiles and reject unbounded label types. And a standard menu of retention tiers, so the aggregate cost of the platform is something anyone can reason about.

**Prove it — interview questions**

1. **[Basic] What is cardinality and why does it matter?**

   <details><summary>Model answer</summary>

   Cardinality is the number of distinct time series, which is the product of the label value counts for a metric. It matters because cost and memory scale with series count rather than with sample count — each series carries index entries and in-memory state, while samples themselves compress to a couple of bytes. Adding a label with ten thousand distinct values multiplies your series count by ten thousand, which is how a single casual change takes down a monitoring platform.

   </details>

2. **[Basic] Why roll up old data?**

   <details><summary>Model answer</summary>

   Because the value of resolution decays with age while its cost does not. Nobody debugs a six-month-old incident at fifteen-second granularity, but everybody wants to see six months of trend. Tiering — raw for days, one minute for weeks, one hour for months, one day for years — typically reduces long-term volume by one to two orders of magnitude while preserving every question anyone actually asks of old data.

   </details>

3. **[Senior] Why can't you average p99 values when rolling up?**

   <details><summary>Model answer</summary>

   Because a percentile is a property of a distribution, and averaging two percentiles does not produce the percentile of the combined distribution — the result is not a quantile of anything. To keep percentiles correct after aggregation, store histogram buckets or a mergeable sketch such as t-digest, then compute the quantile at query time from the merged representation. This has to be decided early, because retrofitting it means you have no correct historical percentiles for the period before the change.

   </details>

4. **[Senior] How do you handle late-arriving data?**

   <details><summary>Model answer</summary>

   By defining a watermark — an explicit statement of how long a time window remains open for late arrivals — and a policy for what happens after it closes. The options are to drop late events and record how many were dropped, to keep windows open longer at the cost of memory, or to route late data to a correction path that emits an amended aggregate. What you cannot do is leave it undefined, because then late events either silently disappear or silently mutate aggregates that downstream systems have already consumed. I would also always emit a metric for late-event volume, since it is the signal that the watermark is wrong.

   </details>

5. **[Staff] Design the metrics model for a platform with 5,000 containers.**

   <details><summary>Model answer</summary>

   I would start from the queries rather than from the data. Per-service aggregates — request rate, error rate, latency distribution by service, endpoint, method and status — are what dashboards and alerts need, and that stays in the low millions of series. The instance or container label is the dangerous one, so I would allow it on a small number of health and resource metrics where finding an outlier instance genuinely matters, and nowhere else. Latency would be stored as histogram buckets rather than pre-computed quantiles so percentiles survive aggregation. Everything per-request goes to sampled traces, everything textual goes to logs indexed by service and time, and per-user analysis goes to the warehouse. Then I would enforce a per-metric cardinality limit in the ingestion path, so a bad deploy degrades one metric rather than the platform.

   </details>

6. **[Principal] How do you prevent cardinality incidents organisationally?**

   <details><summary>Model answer</summary>

   By treating cardinality as a governed resource with enforcement in the path, not as guidance. Each metric gets a cardinality budget enforced at ingestion, so exceeding it drops or rejects the offending series and raises an alert against the owning team rather than exhausting shared memory — the failure must be local, because the alternative is that one team's change blinds everyone during an incident. Label sets belong in a reviewed schema rather than being emergent from code, since the multiplicative effect of a new label is rarely obvious to the person adding it. Instrumentation libraries should make the safe thing easy: histogram helpers rather than quantile helpers, and label constructors that reject unbounded types. And I would publish the routing rule explicitly — aggregates to metrics, per-request to traces, text to logs, per-user to the warehouse — because most incidents here are a legitimate question asked of the wrong system.

   </details>

---

### Graph data modeling

*When relationships are the data and traversal depth is variable, store edges as first-class objects — and bound every traversal or it will not return.*

**Flow:** `Entry vertex` → `Edge filter` → `Bounded traversal` → `Candidate vertices` → `Result`

> **The 30-second version**  
> Store relationships as first-class typed edges so variable-depth traversal is a pointer walk rather than a join. Then bound every traversal by depth, type, degree and result count.

**The problem**

Relational databases handle relationships well until the depth becomes variable. “Friends of this user” is one join. “Friends of friends” is two. “Anyone connected within six hops” is a recursive query whose cost explodes, and “the shortest path between these two accounts” cannot be expressed as a fixed join at all.

The structural issue is that a relational join is a set operation over indexes: each hop scans and matches. A graph store instead stores adjacency directly, so traversing an edge is a pointer dereference whose cost does not depend on the size of the whole dataset.

> **The dividing line**  
> Use a graph store when the **traversal depth is variable or unknown** and relationships are the primary query subject. Use a relational store when depth is fixed and shallow — which, honestly, is most systems. Two or three joins is not a graph problem.

**Mental model**

Vertices are entities; edges are relationships with their own type, direction and properties. Crucially, edges are objects, not foreign keys — they can carry data (`since`, `weight`, `role`) and can be indexed and filtered independently.

1. **Vertex** — An entity with properties: person, account, device, product, permission.
2. **Edge** — A typed, directed relationship with its own properties. `(alice)-[:FOLLOWS {since:2021}]->(bob)`.
3. **Adjacency** — Each vertex holds direct references to its edges, so one hop is a dereference rather than an index lookup.
4. **Traversal** — The query: start somewhere, follow edges matching a pattern, bounded by depth and filters.
5. **Supernode** — A vertex with an enormous edge count. The defining scaling problem of graph systems.

> **Traversal cost is exponential in depth**  
> With average degree `d`, a depth-`k` traversal visits up to `d^k` vertices. At `d = 100` and `k = 4` that is 100 million. Every graph query must be bounded by depth, by edge-type filters, and by a result limit — and even then, one supernode in the path can make a bounded query unbounded.

**How it works**

**Why adjacency beats joins at depth**

```text
RELATIONAL: friends-of-friends-of-friends
  SELECT ... FROM friends f1
  JOIN friends f2 ON f2.a = f1.b
  JOIN friends f3 ON f3.a = f2.b
  WHERE f1.a = 42
  -> three index lookups + hash joins, each producing an
     intermediate set that grows multiplicatively
  -> cost depends on TABLE SIZE and intermediate cardinality

GRAPH: (u {id:42})-[:FRIEND*3]->(x)
  -> follow pointers from vertex 42's adjacency list
  -> cost depends on the NEIGHBOURHOOD size, not the dataset
  -> "index-free adjacency" is the term

BUT: at depth 3 with degree 200, the neighbourhood is
     8 million vertices. Bounded traversal is not optional.
```

1. **Model edges with types and direction from the start** — An untyped edge forces every traversal to consider all relationships. Typed edges let the engine prune aggressively, and they make queries readable.
2. **Bound every traversal** — Maximum depth, edge-type filter, and a result limit. An unbounded traversal in a connected social graph reaches most of the dataset.
3. **Handle supernodes explicitly** — A celebrity with 50 million followers makes any traversal through them catastrophic. Options: refuse to traverse through high-degree vertices, split them into shards, or handle them with a separate materialised path.
4. **Denormalise hot paths** — If “mutual friends count” is on a hot page, precompute it. Graph traversal is fast relative to joins, not fast in absolute terms at high fan-out.
5. **Choose the storage model deliberately** — Native graph stores give index-free adjacency but scale vertically. Distributed graph layers over wide-column stores scale horizontally but lose some traversal efficiency.
6. **Keep an alternative for global questions** — “How many total edges of type X exist?” or “rank all vertices by centrality” are analytical queries better served by a batch graph-processing job than by an online traversal.

**Bounding a traversal properly**

```text
UNBOUNDED (will not return on a real social graph)
  MATCH (a:User {id:42})-[:FOLLOWS*]->(b:User) RETURN b

BOUNDED
  MATCH p = (a:User {id:42})-[:FOLLOWS*1..3]->(b:User)
  WHERE b.active = true
    AND ALL(v IN nodes(p) WHERE v.degree < 10000)   -- skip supernodes
  RETURN DISTINCT b
  LIMIT 500

FOUR BOUNDS APPLIED
  1  depth      *1..3
  2  edge type  :FOLLOWS only
  3  supernode  degree filter on intermediate vertices
  4  result     LIMIT 500
```

> **Write amplification of denormalised graph views**  
> Precomputing “friends of friends” or a follower list per user makes reads fast and makes writes expensive: one new edge invalidates or updates many derived rows. For a celebrity account, a single follow event can fan out to millions of updates. This is the same fan-out-on-write versus fan-out-on-read trade that feed systems face, and the answer is usually hybrid.

**Worked example**

Fraud detection: find accounts connected to a known-fraudulent one through shared devices, addresses or payment instruments.

**Why this is genuinely a graph problem**

```text
THE QUESTION
  "Given fraudulent account F, find accounts within 4 hops
   via shared device, IP, address or card."

RELATIONAL
  4 self-joins across 4 different link tables,
  with UNION at each level -> a query nobody can maintain,
  and intermediate result sets in the millions

GRAPH MODEL
  (:Account)-[:USED_DEVICE]->(:Device)
  (:Account)-[:PAID_WITH]->(:Card)
  (:Account)-[:SHIPPED_TO]->(:Address)

  MATCH p = (f:Account {id:$fraud})
            -[:USED_DEVICE|PAID_WITH|SHIPPED_TO*1..4]-
            (other:Account)
  WHERE ALL(v IN nodes(p) WHERE v.degree < 500)
  RETURN other, length(p) AS distance
  ORDER BY distance LIMIT 200

THE SUPERNODE PROBLEM IS CENTRAL HERE
  a shared corporate IP address connects 100,000 accounts.
  without the degree filter, every query returns "everyone".
  -> the degree filter is not an optimisation, it is the
     difference between a signal and noise.
```

| Metric | Value | Note |
|---|---|---|
| Depth | 4 hops | bounded |
| Edge types | 3 | filtered |
| Degree cap | <500 | **removes supernodes** |
| Result | 200 | limited |

> **Supernode filtering is domain logic, not tuning**  
> A shared office IP is technically an edge and semantically meaningless. Excluding high-degree intermediate vertices is not a performance hack — it encodes the domain rule that a connection through something everyone shares carries no information. Recognising that the performance fix and the correctness fix are the same thing is what distinguishes a good graph model from a slow one.

**When to use it**

- **Variable-depth traversal**: fraud rings, network topology, dependency graphs, access paths.
- **Shortest path and connectivity questions**, which relational stores cannot express naturally.
- **Recommendation via co-occurrence** — “people who bought X also bought Y” expressed as a two-hop traversal.
- **Permission and authorisation graphs (ReBAC)**, where “does user U have access to resource R through any group or role chain?” is inherently a reachability query.
- **Knowledge graphs and entity resolution**, where relationships carry as much meaning as entities.

**When to avoid it**

- **Do not use it for fixed shallow joins.** Two or three joins is a relational problem, and a relational database will do it better.
- **Do not use it for aggregate analytics** across the whole graph — that is a batch processing or columnar workload.
- **Do not use it as a primary transactional store** for entity data that happens to have relationships.
- **Do not run unbounded traversals**, ever, in an online path.
- **Do not ignore supernodes** — they turn every query into a full scan.

**Advantages**

- **Traversal cost depends on the neighbourhood**, not on total dataset size.
- **Relationships are first-class**, carrying type, direction and properties that can be filtered and indexed.
- **Variable-depth queries are expressible** and readable, which recursive SQL is not.
- **Schema is flexible**: adding a new relationship type does not require restructuring existing data.
- **Path results carry structure**, so you get *how* two things are connected, not just *that* they are.

**Disadvantages**

- **Horizontal scaling is genuinely hard** — partitioning a graph cuts edges, and cut edges become network hops during traversal.
- **Supernodes break the cost model** and must be handled as a first-class design concern.
- **Global and aggregate queries are weak** compared with columnar stores.
- **Smaller ecosystem**: fewer engineers, less tooling, less operational knowledge than relational.
- **Another datastore to operate**, with its own backup, failover and capacity story.
- **Query cost is hard to predict**, since it depends on graph shape rather than on table statistics.

**Trade-offs**

**Modelling relationships: three options**

|  | Relational join table | Graph store | Denormalised adjacency lists |
|---|---|---|---|
| Fixed 1–3 hop queries | Excellent | Fine | Excellent |
| Variable depth | Painful recursive CTEs | Natural | Not possible |
| Shortest path | Impractical | Native | Not possible |
| Write cost | One row per edge | One edge | Fan-out on write |
| Horizontal scaling | Shard by one side | Hard | Easy |
| Operational maturity | Highest | Moderate | Highest |

> **Framing the decision honestly**  
> “The access-path question — can this user reach this resource through any group chain — is genuinely variable depth, so it is a graph query. But it is one query among many, so rather than moving the whole system I'd keep the relational store authoritative and maintain a graph projection for reachability, rebuilt from the source of truth.”

**How it fails**

**Graph failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Query never returns | Unbounded depth in a connected graph | Depth limit, edge-type filter, result limit |
| One query melts the cluster | Traversal through a supernode | Degree cap on intermediate vertices; shard the supernode |
| Results are meaningless | Connections via ubiquitous shared attributes | Exclude high-degree vertices as a domain rule |
| Scaling out makes traversal slower | Partitioning cut edges; hops became network calls | Partition by community; replicate hot vertices; keep graph on one node if it fits |
| Write throughput collapses | Denormalised adjacency fan-out on a celebrity edge | Hybrid: fan out for normal vertices, read-time merge for supernodes |
| Graph diverges from the source of truth | Dual writes to two stores | Derive the graph from an event stream; rebuild capability |
| Unpredictable query latency | Cost depends on local graph shape | Enforce limits; time-box queries; precompute hot paths |

**Limits**

> **Design numbers**
>
> - **Traversal fan-out**: `d^k`. Degree 100 at depth 4 is 100 million vertices — bound everything.
> - **Supernode threshold**: treat vertices above roughly 10,000 edges as requiring special handling in online queries.
> - **Single-node capacity**: native graph engines commonly handle billions of edges on one large machine, which covers most non-social-network use cases.
> - **Partitioning penalty**: a cut edge becomes a network hop; graphs with high cross-partition edge ratios scale poorly.
> - **Online versus batch**: PageRank-style global algorithms belong in a batch graph processing framework, not in an online query.

**Alternatives**

| Approach | Best when | Trade |
|---|---|---|
| Relational join tables | Fixed, shallow relationships | Recursive queries are painful |
| Recursive CTE in SQL | Occasional variable depth, modest fan-out | Hard to bound and optimise |
| Native graph database | Traversal is the primary workload | Scaling and operational cost |
| Graph layer over wide-column | Very large graphs needing horizontal scale | Loses index-free adjacency benefits |
| Precomputed reachability / closure table | Reads dominate, graph changes rarely | Expensive to maintain on write |
| Batch graph processing | Global algorithms over the whole graph | Not interactive |

The closure-table option is underrated for authorisation: precomputing “which users can reach which resources” makes checks a single indexed lookup, at the cost of recomputation when the group hierarchy changes — which is usually rare.

**In real systems**

- **Fraud and anti-money-laundering systems** are the clearest genuine fit, since the question is literally “what is connected to this, and how.”
- **Google's Zanzibar** models authorisation as a relationship graph with reachability checks, and solves the supernode problem with denormalised, cached expansion.
- **Social networks** rarely use general-purpose graph databases at scale, instead building custom adjacency services — because partitioning a social graph is the hard part and general engines do not solve it.
- **Network and infrastructure topology** tools model dependencies as graphs to answer impact-analysis questions across variable depth.
- **Knowledge graphs** in search and assistants use graph models for entity relationships with rich edge semantics.

**Common mistakes**

- **Adopting a graph database for two-hop joins** that a relational store handles better.
- **Unbounded traversals** in an online request path.
- **Ignoring supernodes**, so one shared attribute connects everything to everything.
- **Dual-writing** to a relational store and a graph store, guaranteeing divergence.
- **Expecting horizontal scaling** to work the way it does for key-value stores.
- **Running global graph algorithms online** instead of in a batch job.
- **Untyped edges**, which prevent the engine from pruning anything.

**The staff-level view**

The main Staff-level contribution here is usually deciding *not* to adopt a graph database, and instead identifying the one query that genuinely needs graph semantics.

- **Demand the specific query that justifies it.** “Our data is connected” is not a reason; “we need reachability across variable depth in an online path” is.
- **Prefer a graph projection over a graph migration.** Keep the transactional store authoritative and derive a graph view from its event stream, so the graph is rebuildable and not a second source of truth.
- **Make supernode handling explicit domain logic**, written down and reviewed, not a tuning parameter buried in a query.
- **Enforce query bounds at the platform level** — maximum depth, timeout, result cap — so no single query can take the cluster down.
- **Consider a closure table first** for authorisation-style reachability, since precomputation is often cheaper than a whole new datastore.

**Go deeper**

Graph modelling makes edges first-class objects with type, direction and properties, and stores adjacency directly so following an edge is a dereference whose cost depends on the neighbourhood rather than on dataset size. That makes variable-depth questions — reachability, shortest path, ring detection — expressible and fast, where recursive SQL becomes unmaintainable.

Two constraints dominate. Traversal cost grows as degree to the power of depth, so every online query needs four bounds: maximum depth, edge-type filter, a degree cap excluding supernodes, and a result limit. And supernodes — a celebrity account, a shared corporate IP — break the cost model entirely; excluding them is usually a domain rule as much as a performance fix, since a connection through something everyone shares carries no information.

Adopt it narrowly. Fixed two- or three-hop queries are relational problems, and horizontal scaling is genuinely hard because partitioning cuts edges and cut edges become network hops. For authorisation-style reachability, a precomputed closure table often delivers most of the value without a new datastore; where a graph is warranted, prefer a projection derived from the transactional store's change stream over dual writes, which guarantee divergence.

Graph storage earns its place when relationships are the query subject and traversal depth is variable. Everything else about it — the scaling difficulty, the supernode problem, the unpredictable query cost — follows from that same property.

**Index-free adjacency.** A relational join is a set operation: each hop performs index lookups and produces an intermediate result whose size depends on table cardinality. A graph store keeps direct references from each vertex to its edges, so one hop is a pointer dereference and cost depends on the local neighbourhood. That is what makes depth-4 traversal feasible where four self-joins are not. It does not make traversal cheap in absolute terms — with average degree 100, depth 4 still reaches 100 million vertices.

**Therefore, bound everything.** Four bounds, applied together: a maximum depth; an edge-type filter so the engine prunes irrelevant relationships; a degree cap on intermediate vertices; and a result limit. Any one alone is insufficient. Beyond the query, the platform should enforce a timeout and a maximum traversal budget, because a single unbounded query in a connected graph can consume the cluster, and in a social or fraud graph nearly everything is connected to nearly everything within a few hops.

**Supernodes are the defining problem.** A vertex with millions of edges — a celebrity, a shared office IP, a popular device model — makes any path through it catastrophic in cost and usually meaningless in signal. Excluding high-degree intermediate vertices is therefore domain logic rather than tuning: a connection through something everyone shares tells you nothing. The alternative treatments are sharding the supernode's edges across multiple physical vertices, or handling it with a separately materialised path, but the first move should be to ask whether traversing it is semantically valid at all.

**Scaling is the honest weakness.** Partitioning a graph cuts edges, and every cut edge converts a dereference into a network hop; good partitioning requires community detection, which is expensive and unstable as the graph evolves. This is why very large social graphs are served by purpose-built adjacency services with application-specific partitioning rather than by general-purpose graph engines, and why a single large machine holding billions of edges is often the right answer for non-social workloads.

**Adopt narrowly, derive rather than migrate.** The question that justifies a graph store should be nameable and genuinely variable-depth; “our data is connected” is not one. For authorisation-style reachability — can this user reach this resource through any group chain — a precomputed closure table often delivers most of the value with one indexed lookup and no new datastore, because hierarchies change rarely and are read constantly. Where a graph is warranted, keep the transactional store authoritative and build the graph as a projection from its change stream, so it is rebuildable; dual-writing to two stores guarantees eventual divergence, and authorisation is precisely where that is unacceptable.

**The organisational cost is real.** A graph engine is a permanent commitment with a smaller talent pool, less mature tooling, and query costs that depend on graph shape rather than on table statistics — which makes “why did this query melt the cluster today when it was fine yesterday” genuinely harder to answer. Once it exists, work gets routed to it because it is interesting, so platform-level query bounds and a clearly scoped mandate matter more than the technology choice itself.

**Prove it — interview questions**

1. **[Basic] When is a graph database actually the right choice?**

   <details><summary>Model answer</summary>

   When traversal depth is variable or unknown and relationships are the query subject — reachability, shortest path, connectivity, ring detection. Fixed shallow queries of two or three hops are relational problems and a relational database will do them better with more mature tooling. The honest test is whether you can write the query as a fixed set of joins; if you can, you probably do not need a graph store.

   </details>

2. **[Basic] What is a supernode and why does it matter?**

   <details><summary>Model answer</summary>

   A vertex with an enormous number of edges — a celebrity account, a shared corporate IP, a common device model. It matters because traversal cost is proportional to fan-out, so any path through a supernode explodes: one query touches a huge fraction of the graph. It is also usually semantically meaningless, since a connection through something everyone shares carries no information, which means excluding high-degree vertices improves correctness as well as performance.

   </details>

3. **[Senior] How do you bound a traversal safely?**

   <details><summary>Model answer</summary>

   Four bounds together. A maximum depth, because cost grows as degree to the power of depth. An edge-type filter, so the engine prunes relationships that are irrelevant to this question. A degree cap on intermediate vertices, to exclude supernodes. And a result limit. Any one of these alone is insufficient — a depth-3 query with no type filter through a supernode still visits millions of vertices. I would also enforce a query timeout at the platform level so that a query nobody bounded cannot take the cluster down.

   </details>

4. **[Senior] Why is horizontal scaling hard for graphs?**

   <details><summary>Model answer</summary>

   Because partitioning cuts edges, and a cut edge turns a pointer dereference into a network round trip. Unlike key-value data, where partitions are independent by construction, a graph's value is precisely in the connections that cross any partition you draw. Good partitioning requires community detection so that densely connected regions stay together, which is expensive to compute and unstable as the graph changes. This is why very large social graphs are usually served by purpose-built adjacency services with application-specific partitioning rather than by general graph engines.

   </details>

5. **[Staff] A team wants to migrate an authorisation system to a graph database. How do you evaluate it?**

   <details><summary>Model answer</summary>

   First I would confirm the query is genuinely variable-depth reachability — “can this user reach this resource through any chain of group and role memberships” — because if the hierarchy is bounded to two or three levels it is a relational problem. If it is genuinely variable, the next question is whether an online traversal is needed or whether precomputation works: authorisation hierarchies change rarely and are read constantly, which is the ideal profile for a closure table, where every reachable pair is materialised and a check becomes one indexed lookup. That avoids adopting a new datastore entirely. If the graph is too large or too dynamic for closure, I would still keep the relational store authoritative and derive a graph projection from its change stream, so the graph is rebuildable rather than a second source of truth — because dual-writing to two stores guarantees eventual divergence and authorisation is exactly where you cannot tolerate it.

   </details>

6. **[Principal] What organisational risks come with adopting a graph database?**

   <details><summary>Model answer</summary>

   The main one is that it is a permanent operational commitment with a much smaller talent pool than relational. Every on-call rotation now needs someone who understands its failure modes, its backup and restore story, and why a query that ran yesterday melted the cluster today — and query cost depending on local graph shape rather than on table statistics makes that genuinely harder to reason about than SQL. The second risk is that once it exists, teams route things to it that do not belong there, because it is the interesting new datastore, and unbounded traversals are easy to write. So if we adopt one I would scope it narrowly to the query class that justified it, enforce depth, timeout and result limits at the platform level so no single query can be catastrophic, and keep the authoritative data elsewhere with the graph derived and rebuildable. The strategic question I would put to the team first is whether a projection or a closure table gets us 90% of the value without the commitment.

   </details>

---

### Columnar analytical storage

*Store values column by column so analytical scans read only what they need, compress ten times better, and run vectorised over whole blocks.*

**Flow:** `Rows` → `Column encoding` → `Row groups` → `Predicate pruning` → `Aggregation`

> **The 30-second version**  
> Store by column so queries read only the fields they need, compress 5–20×, and execute vectorised. Layout — partitioning and sort order — matters more than the format itself.

**The problem**

An analytical query — “average order value by region for the last quarter” — touches three columns out of fifty and aggregates over a hundred million rows. A row store must read all fifty columns of every row because rows are stored contiguously, so it reads perhaps 95% more bytes than the query needs.

It is worse than the byte count suggests. Row storage interleaves values of different types, which defeats compression, and it processes one row at a time through a tuple-at-a-time execution model, which defeats the CPU's vector units and cache.

> **The three multiplicative wins**  
> **IO**: read 3 columns instead of 50 — often a 10–20× reduction. **Compression**: adjacent values in a column are the same type and often similar, so run-length, dictionary and delta encodings achieve 5–20× versus 2–3× for rows. **CPU**: values of one type in a contiguous array can be processed with SIMD instructions, tens of values per cycle. These multiply, which is why analytical queries can be 100× faster on the same hardware.

**Mental model**

Transpose the table. Instead of storing row 1 then row 2, store all of column A, then all of column B. A query then reads only the columns it names.

1. **Column chunk** — All values for one column within a row group, encoded and compressed independently.
2. **Row group / stripe** — A horizontal slice of the table (typically 100k–1M rows) containing one chunk per column. The unit of parallelism and pruning.
3. **Statistics** — Per-chunk min, max, null count, and sometimes a bloom filter. These let the engine skip chunks entirely.
4. **Encoding** — Dictionary for low-cardinality strings, run-length for sorted or repetitive values, delta for sequences, bit-packing for small integers.
5. **Vectorised execution** — Process a batch of values at once through SIMD rather than one row at a time.

> **Sort order is the most under-used lever**  
> Statistics only prune if the filter column correlates with physical layout. Sorting or clustering by the column you filter on most — usually time, sometimes tenant — turns predicate pruning from theoretical into dramatic, often eliminating 99% of row groups before any data is read. It also makes run-length encoding far more effective, so it improves compression at the same time.

**How it works**

**Why the same query reads 30x less data**

```text
TABLE: 100M orders, 50 columns, ~500 bytes/row  = 50 GB

QUERY: SELECT region, AVG(total)
       FROM orders WHERE order_date >= '2024-01-01'
       GROUP BY region

ROW STORE
  scan all rows -> 50 GB read (or an index + random IO)

COLUMNAR, unsorted
  read order_date, region, total only
    3 cols x 100M x ~8 B  = 2.4 GB
    compressed ~4x        = 600 MB      -> 80x less IO

COLUMNAR, sorted by order_date
  row-group statistics show min/max date per group
  skip every group whose max < 2024-01-01
    ~10% of groups survive = 60 MB      -> 800x less IO

The sort order did more than the format did.
```

1. **Choose a sort/cluster key that matches your dominant filter** — Almost always time for event data; sometimes tenant for multi-tenant analytics. This single choice dominates query performance.
2. **Partition physically by a coarse key, then sort within** — Directory-level partitioning (`/dt=2024-06-01/`) prunes before any file is opened; sorting within files prunes row groups after.
3. **Size files deliberately** — Many tiny files destroy performance — per-file overhead, metadata listing, and lost compression. Target files in the hundreds of megabytes and compact small ones.
4. **Let the encoding do the work** — Low-cardinality strings should be dictionary-encoded; sorted columns should be run-length encoded. Most engines choose automatically, but verify — a mis-encoded column can be an order of magnitude larger.
5. **Accept that point updates are expensive** — Changing one value means rewriting a column chunk or a whole file. Use table formats (Iceberg, Delta, Hudi) that layer merge-on-read or copy-on-write semantics over immutable files.
6. **Avoid `SELECT *`** — It defeats the entire premise. In a columnar store, selecting fewer columns is a direct, linear reduction in work.

**Encodings and what they exploit**

```text
DICTIONARY     "US","US","GB","US" -> [0,0,1,0] + dict{0:"US",1:"GB"}
               wins on low-cardinality strings (country, status)

RUN-LENGTH     1,1,1,1,1,2,2,2     -> (1 x5)(2 x3)
               wins on sorted or repetitive columns

DELTA          1000,1001,1003,1006 -> 1000,+1,+2,+3
               wins on timestamps, ids, monotonic sequences

BIT-PACKING    values 0-7 stored in 3 bits, not 32
               wins on small-range integers

COMBINED with a general compressor (zstd/snappy) on top,
5-20x total is typical. Row stores manage 2-3x because
adjacent bytes are unrelated types.
```

> **Small files are the most common self-inflicted wound**  
> A streaming pipeline writing every minute creates 1,440 files a day per partition. Each carries metadata overhead, each is opened separately, and none compresses well. Query planning alone can take longer than the scan. Compaction into large files is not an optimisation — it is a prerequisite for the format to work at all.

**Worked example**

Clickstream analytics: 5 billion events a year, queried by date range and dimension.

**Layout decisions and their effect**

```text
DATA
  5B events/year x ~40 columns, ~300 B/row raw = 1.5 TB/year

LAYOUT
  physical partition:  /dt=YYYY-MM-DD/
  sort within file:    (user_segment, event_time)
  file size target:    512 MB
  format:              Parquet + zstd

STORAGE
  raw 1.5 TB -> compressed ~150 GB      (10x)

QUERY: "daily active users by segment, last 30 days"
  partition pruning:  365 days -> 30 days       (12x)
  column pruning:     40 cols -> 3 cols         (13x)
  row-group stats on segment                    (2-5x)
  -> scans roughly 100-300 MB instead of 1.5 TB

WHAT THIS DESIGN CANNOT DO WELL
  "show me the full event record for user 12345"
  -> a point lookup, scanning every partition.
     Keep a separate serving store keyed by user for that.
```

| Metric | Value | Note |
|---|---|---|
| Raw | 1.5 TB/yr | 40 columns |
| Compressed | ~150 GB | 10× |
| Query scan | ~200 MB | **pruned** |
| Point lookup | poor | wrong tool |

> **Columnar is a scan format, not a lookup format**  
> The same properties that make aggregation fast make single-record retrieval slow: there is no primary key index, and reconstructing one row means touching every column chunk. Systems that need both analytical scans and point lookups run two stores — a columnar lake for analysis and a row or key-value store for serving — fed from the same pipeline.

**When to use it**

- **Analytical queries** that aggregate over many rows and few columns.
- **Data warehouses and lakes**, where the dominant access is scan-and-aggregate.
- **Event and log analysis** over long time ranges.
- **Reporting and BI workloads**, where query shapes vary but all are aggregate.
- **Cheap long-term retention**, since compression plus object storage makes years of history affordable.

**When to avoid it**

- **Do not use it for transactional workloads** — point reads, single-row updates and inserts of one row at a time.
- **Do not use it where low-latency single-record lookup matters**; there is no index for that.
- **Do not write small files continuously** without a compaction strategy.
- **Do not use `SELECT *`**, which discards the format's main advantage.
- **Do not expect row-level mutation to be cheap**, even with table formats that support it.

**Advantages**

- **Reads only the columns referenced**, often a 10–20× IO reduction.
- **Compression of 5–20×** because adjacent values share type and often value.
- **Vectorised, SIMD-friendly execution** over contiguous typed arrays.
- **Predicate pruning via statistics** skips entire row groups and files before reading.
- **Separates storage from compute**, so files on object storage can be queried by many engines independently.
- **Open formats** (Parquet, ORC) avoid vendor lock-in for the data itself.

**Disadvantages**

- **Point lookups are slow**, since one row is scattered across all column chunks.
- **Writes are batch-oriented**; single-row inserts and updates are expensive.
- **Small-file proliferation** degrades performance severely and requires ongoing compaction.
- **Schema evolution is constrained** — adding columns is easy, changing types or reordering is not.
- **No transactions or constraints** in the raw format; table formats add them at a cost.
- **Query latency is seconds, not milliseconds**, which rules it out of serving paths.

**Trade-offs**

**Row versus column storage**

|  | Row store | Column store |
|---|---|---|
| Point read by key | Milliseconds via index | Slow — scans column chunks |
| Aggregate over many rows | Reads all columns | Reads only what is needed |
| Single-row insert/update | Cheap | Expensive — rewrite or merge-on-read |
| Compression | 2–3× | 5–20× |
| CPU model | Tuple at a time | Vectorised / SIMD |
| Typical latency | Sub-millisecond | Seconds |
| Best fit | OLTP | OLAP |

The two are complements rather than competitors. The standard architecture runs a row store for transactions and replicates into a columnar store for analysis via change data capture, because trying to serve both workloads from one engine compromises both.

**How it fails**

**Columnar failure modes**

| Failure | Cause | Fix |
|---|---|---|
| Queries slow despite the format | Data not sorted by the filter column, so no pruning | Sort/cluster by the dominant predicate; re-write the data |
| Query planning takes longer than the scan | Millions of small files | Compaction into large files; a table format with manifests |
| Compression is poor | High-cardinality or randomly ordered columns | Sort to enable run-length; check encoding choices |
| Cannot update a record | Immutable files | Table format with merge-on-read or copy-on-write |
| Point lookups time out | No index; wrong tool | Separate serving store keyed appropriately |
| Schema change breaks readers | Type change or column reorder | Additive-only evolution; use format-level schema evolution |
| Cost higher than expected | Scanning unpruned data on a per-byte pricing model | Partitioning and sorting; enforce query limits |

**Limits**

> **Sizing guidance**
>
> - **File size**: aim for 128 MB–1 GB per file; compact anything smaller.
> - **Row group size**: typically 100k–1M rows, balancing pruning granularity against per-group overhead.
> - **Compression**: 5–20× typical; zstd generally beats snappy on ratio at moderate CPU cost.
> - **Partition count**: too many partitions means small files and heavy metadata; too few means poor pruning. Aim for partitions of at least a few hundred megabytes.
> - **Latency floor**: seconds for interactive queries; this is not a sub-100 ms serving technology.

**Alternatives**

| Option | Best for | Trade |
|---|---|---|
| Columnar files on object storage (Parquet) | Cheap, open, engine-agnostic analytics | Higher latency; needs compaction |
| Columnar MPP warehouse | Interactive BI at scale | Cost; data lives in the vendor's system |
| Real-time OLAP (Druid, ClickHouse, Pinot) | Sub-second analytical queries on fresh data | More operational complexity |
| Row store with indexes | Mixed workload at modest scale | Analytical queries degrade as data grows |
| Pre-aggregated cubes / materialised views | Fixed dashboards | Only anticipated questions |

Real-time OLAP engines occupy an important middle ground: columnar storage with an ingestion path designed for streaming and query latency in the tens of milliseconds, which is what user-facing analytics dashboards actually need.

**In real systems**

- **Parquet and ORC** are the de facto open formats, giving engine independence over data stored in object storage.
- **Snowflake, BigQuery and Redshift** are columnar engines whose pricing models make column and partition pruning a direct cost lever.
- **Apache Iceberg, Delta Lake and Hudi** add transactions, schema evolution and row-level updates on top of immutable columnar files.
- **ClickHouse and Druid** apply columnar storage to low-latency, high-ingest analytics, powering user-facing dashboards.
- **Change data capture into a lake** is the standard pattern: a row store handles transactions and streams changes into columnar storage for analysis.

**Common mistakes**

- **Writing small files continuously** with no compaction.
- **Not sorting by the dominant filter column**, so statistics prune nothing.
- **`SELECT *`** in a format designed around column pruning.
- **Using the lake for point lookups** instead of a serving store.
- **Over-partitioning**, producing tiny partitions and heavy metadata.
- **Raw Parquet directories with no table format**, making updates and schema change painful.
- **Expecting sub-second latency** from a scan-oriented format.

**The staff-level view**

Most columnar performance problems are layout problems, and layout is decided once, early, by whoever writes the ingestion pipeline.

- **Make sort order and partitioning a reviewed design decision**, not a default. It typically matters more than the engine choice.
- **Require a compaction strategy before any streaming pipeline writes to the lake.** Small files are the most common and most expensive failure.
- **Adopt a table format early.** Retrofitting Iceberg or Delta onto a directory of raw Parquet is far harder than starting with it, and it is what makes updates, deletes and schema evolution tractable.
- **Separate serving from analytics explicitly.** Attempting point lookups from the lake produces a slow, expensive system that satisfies neither workload.
- **On per-byte pricing, treat pruning as a cost control.** Enforcing partition filters in query tooling is a direct budget lever.

**Go deeper**

Columnar storage transposes the table: all values of one column are stored contiguously. An analytical query touching three columns of fifty reads only those three, compression reaches 5–20× because adjacent values share a type and often a value, and execution is vectorised over typed arrays using SIMD. These three effects multiply, which is why aggregate queries can run two orders of magnitude faster on identical hardware.

Layout matters more than format. Per-chunk statistics only prune if the filter column correlates with physical order, so sorting or clustering by the dominant predicate — usually time — can eliminate 99% of row groups before any data is read, while directory-level partitioning eliminates whole files before they are opened. Getting this wrong means owning the right format and none of its benefit.

The failure that dominates real deployments is small files: a streaming pipeline writing every minute produces thousands of tiny files whose metadata overhead and poor compression make query planning slower than the scan. Continuous compaction, managed by a table format such as Iceberg or Delta so swaps are atomic, is a prerequisite rather than an optimisation. And columnar is a scan format, not a lookup format — point reads need a separate serving store.

Columnar storage is a physical layout choice whose consequences reach into IO, compression and CPU simultaneously, which is why the gains on analytical workloads are multiplicative rather than incremental.

**The three wins.** Column pruning means a query naming three of fifty columns reads roughly 6% of the bytes. Compression improves from 2–3× to 5–20× because a column chunk contains values of one type, frequently similar or repeated, making dictionary, run-length, delta and bit-packing encodings highly effective before a general compressor is even applied. And execution becomes vectorised: a contiguous array of one type can be filtered and aggregated with SIMD instructions processing many values per cycle, rather than materialising and interpreting one tuple at a time. Multiplied together, 100× on aggregate queries is ordinary.

**Layout dominates format.** Per-row-group statistics — min, max, null count, sometimes bloom filters — allow the engine to skip chunks entirely, but only if the filter column correlates with physical order. Sorting by the dominant predicate, usually time for event data, turns pruning from theoretical into decisive and simultaneously improves compression by making run-length encoding effective. Directory-level partitioning prunes before a file is even opened. In practice, well-partitioned and well-sorted data in an adequate format beats poorly laid-out data in the best format by an order of magnitude.

**Small files are the characteristic failure.** Streaming ingestion writing every minute produces thousands of files per partition per day. Each carries footer and metadata overhead, each must be opened and planned, and none compresses well because encodings need volume to work. Query planning alone can exceed the scan time. Continuous compaction into files of a few hundred megabytes is therefore a prerequisite, and a table format — Iceberg, Delta, Hudi — is what makes it safe, providing atomic swaps so readers never observe a partial rewrite, along with schema evolution and row-level updates layered over immutable files.

**Know what it cannot do.** A single row is scattered across every column chunk and there is no primary key index, so point lookups are slow by construction. Single-row inserts and updates are expensive even with merge-on-read. Query latency is seconds, not milliseconds. These are not deficiencies to be tuned away — they are the inverse of the properties that make scanning fast. Architectures needing both run a row store for transactions and stream changes into columnar storage for analysis, which is what change data capture into a lake exists to do.

**Cost control is layout plus enforcement.** On per-byte-scanned pricing, pruning is the budget. That means treating partitioning and sort order as reviewed design decisions matched to the real query mix; running compaction continuously so pruning is not defeated by fragmentation; requiring partition predicates in query tooling so unbounded scans are rejected by default; and serving frequent dashboard queries from materialised aggregates rather than rescanning raw events every minute. Layered on top is the same retention argument that governs all storage — tier and eventually delete raw events while retaining pre-aggregated history, which is usually the single largest saving available.

**Prove it — interview questions**

1. **[Basic] Why is columnar storage faster for analytics?**

   <details><summary>Model answer</summary>

   Three effects that multiply. It reads only the columns the query names, often a 10–20× reduction in bytes. It compresses far better — 5–20× versus 2–3× — because adjacent values share a type and are frequently similar, enabling dictionary, run-length and delta encoding. And it executes vectorised over contiguous typed arrays, using SIMD instructions instead of processing one row at a time. Together these routinely give two orders of magnitude on aggregate queries.

   </details>

2. **[Basic] Why is it bad for point lookups?**

   <details><summary>Model answer</summary>

   Because a single row is scattered across every column chunk, so reconstructing it means touching all of them, and there is no primary key index to locate it. The format is optimised for scanning many rows of few columns, which is the exact opposite shape. Systems needing both run two stores — a row or key-value store for serving and a columnar store for analysis — fed from the same pipeline.

   </details>

3. **[Senior] What matters more, the file format or the data layout?**

   <details><summary>Model answer</summary>

   The layout, usually by a wide margin. Columnar format alone might give a 10–20× improvement from column pruning and compression. But sorting by the dominant filter column lets row-group statistics eliminate most groups before reading anything, and physical partitioning eliminates whole files before they are even opened — together often another 10–100×. I have seen well-partitioned, well-sorted data in a mediocre format outperform poorly laid-out data in the best format by an order of magnitude.

   </details>

4. **[Senior] What is the small-file problem and how do you avoid it?**

   <details><summary>Model answer</summary>

   A streaming pipeline writing frequently creates thousands of tiny files per partition. Each carries metadata and open overhead, none compresses well because encodings need volume to be effective, and query planning — just listing and reading footers — can take longer than the scan itself. The fix is a compaction job that rewrites small files into large ones, ideally in the hundreds of megabytes, run continuously as part of the pipeline rather than as periodic remediation. A table format like Iceberg or Delta makes this safe by handling the atomic swap so readers never see a partial state.

   </details>

5. **[Staff] Design the storage layer for a product analytics system with 5 billion events per year.**

   <details><summary>Model answer</summary>

   I would write Parquet with zstd to object storage, partitioned by date at the directory level and sorted within files by the next most common filter — typically a tenant or segment identifier — with files targeted at roughly 512 MB. Ingestion would land small files quickly for freshness, with a compaction job merging them into the target size within minutes, managed by a table format so the swap is atomic and readers are never inconsistent. That layout gives partition pruning by date, row-group pruning by segment, and column pruning per query, which typically reduces a query over a year of data to scanning a few hundred megabytes. Crucially I would keep a separate serving store for per-user record lookups, because that access pattern is the inverse of what this layout is good at, and trying to serve both from the lake produces a system that is slow and expensive for both.

   </details>

6. **[Principal] How do you control cost in a per-byte-scanned analytics platform?**

   <details><summary>Model answer</summary>

   By making pruning a property of the platform rather than of each analyst's SQL. Layout comes first — partitioning and sort order chosen for the actual query mix, reviewed as a design decision, with compaction running continuously so pruning is not defeated by small files. Then enforcement: query tooling that requires a partition predicate, rejecting unbounded scans by default, and per-team budgets with visible attribution so the cost of an expensive query lands on whoever ran it. Then substitution: the most frequent dashboard queries should be served from materialised aggregates rather than scanning raw events, since a dashboard refreshing every minute over a year of data is pure waste. And finally lifecycle: raw events tiered to cheaper storage classes and eventually deleted, with pre-aggregated history retained instead — the same retention argument as any storage system, which is usually the largest single lever.

   </details>

---
