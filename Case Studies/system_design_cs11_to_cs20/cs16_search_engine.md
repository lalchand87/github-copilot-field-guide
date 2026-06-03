# Case Study 16 — Search Engine (Typeahead + Full-Text)

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Trie vs Redis for typeahead, Inverted Index, Query pipeline.

---

## Business Context

Search serves two distinct UX patterns: (1) typeahead autocomplete while the user is still typing (< 50ms), and (2) full search results with ranking and faceting after the user submits a query (< 500ms). These have fundamentally different architectures and data structures.

For a payment platform: searching payment history ("show me payments to Amazon in January"), merchant lookup, user search for sending money.

**Business goals driving architecture:**
- Typeahead: as fast as possible (< 50ms); users abandon after 100ms
- Full search: relevant results; faceted filtering; pagination
- Index freshness: new merchants and users searchable within 60s
- Personalization: payment history search is user-specific (not shared index)

---

## Recognition Framework

### Signals from the Problem
```
- Typeahead: prefix-based ("goo" → Google, Gooogle, Goodreads); must be fast
- Full search: relevance ranking + boolean queries + faceting
- Two different workloads: typeahead (pure speed) vs full search (quality + speed)
- Global search: shared index (merchants, public entities) vs personal search (user's transactions)
- Index updates: new entities must appear quickly (streaming index update)
- Scale: 1B documents searchable; 100K queries/sec
```

### Typeahead vs Full-Text — Different Data Structures

```
Typeahead (autocomplete while typing):
  Query: "goo" → suggestions: ["Google Pay", "Goobe Commerce", "Goodreads Shop"]
  Requirement: < 50ms; prefix match; rank by popularity
  Data structure: Trie (prefix tree) OR inverted index on n-grams
  
  Option A: In-memory Trie
    Store all entity names in a prefix tree
    Each node represents a character; leaf = entity
    Lookup "goo": traverse g→o→o → return all children entities
    Pros: O(prefix_length) lookup; very fast
    Cons: memory intensive; hard to distribute; 10M entities × avg 50 bytes = 500 MB
    Use when: entity list fits in memory; single node is acceptable
  
  Option B: Redis with ZRANGEBYLEX
    Store: ZADD typeahead:merchants 0 "google pay:merch-123"
           ZADD typeahead:merchants 0 "goobe commerce:merch-456"
    Query: ZRANGEBYLEX typeahead:merchants "[goo" "[goo\xff" LIMIT 10
    Pros: distributed; easy horizontal scaling; sorted alphabetically
    Cons: only prefix match (not substring match); score-based ranking needs separate key
    Use when: horizontal scale needed; prefix match is sufficient
  
  Option C: Elasticsearch with edge n-grams
    Index-time: "Google" → "G", "Go", "Goo", "Goog", "Googl", "Google"
    Query-time: "goo" matches "Goo" token → hits "Google", "Goobe", etc.
    Pros: full ES power; relevance ranking; faceting; substring matching
    Cons: index size 5-10× larger; latency 50-200ms (not ideal for typeahead)
    Use when: typeahead must support substring ("oog" matches "Google") + ranking

CHOSEN:
  Typeahead: Redis ZRANGEBYLEX (fast prefix, distributed)
    + fallback to Elasticsearch for low-traffic queries
  Full search: Elasticsearch (relevance ranking, faceting, complex queries)
```

### Patterns Selected
| Pattern | Signal that triggered it |
|---|---|
| Redis ZRANGEBYLEX | Fast prefix typeahead; O(log N + M); distributed |
| Elasticsearch | Full-text search; inverted index; BM25 ranking; faceting |
| Kafka → ES Sync | Async indexing; entity updates propagate within 60s |
| Query Understanding | Parse query intent before hitting ES (entity detection, spelling correction) |
| Personalized Ranking | User's transaction history boosts relevant merchants |

---

## Problem Statement

Search system for a payment platform. Two modes: (1) typeahead autocomplete for merchant/user search while typing, (2) full-text search of payment history and merchant catalog with faceting.

---

## Functional Requirements
- Typeahead: prefix suggestions as user types (merchants, users, transaction descriptions)
- Full search: payment history, merchant catalog, user directory
- Faceted filtering: by date range, amount range, category, merchant
- Personalization: rank personal transaction history above global results
- Index freshness: new entities searchable within 60 seconds
- Spelling correction: "Amazin" → "Amazon"

## Non-Functional Requirements
- Typeahead latency: **< 50ms p99**
- Full search latency: **< 500ms p99**
- Throughput: **100,000 queries/sec** combined
- Index freshness: **< 60 seconds**
- Index size: **1B documents**

---

## Capacity Estimation
```
Typeahead:
  100K queries/sec (all search types); 70% are typeahead = 70K typeahead/sec
  Each query: 1-4 characters; Redis ZRANGEBYLEX
  Redis handles 100K+ ops/sec per node → single node fine for typeahead

Full search:
  30K full-text queries/sec
  Elasticsearch: 1B documents; 10 shards × 3 replicas = 30 ES nodes
  Each node: 1B / 10 = 100M docs; at 1 KB average = 100 GB per shard

Indexing:
  New entities: ~10K/sec (payments create merchant records; new users join)
  Index update: Kafka → ES consumer; batch 1K docs/call; < 60s per doc
```

---

## Core Patterns
| Pattern | Why |
|---|---|
| Redis ZRANGEBYLEX | O(log N + M) prefix typeahead; < 5ms; distributed |
| Elasticsearch | Inverted index; BM25 ranking; aggregations for faceting |
| Kafka Async Indexing | Decouple write path from ES; 60s freshness acceptable |
| Query Understanding | Spell-correct → entity detection → ES query construction |
| Two-Phase Search | Phase 1: Retrieve (ES) → Phase 2: Re-rank (personalization) |

---

## Data Model

```
Redis Typeahead (sorted sets; lexicographically sorted):
  typeahead:merchants → sorted set
    Each member: "{name_lowercase}:{merchant_id}:{popularity_score_padded}"
    Example: "amazon pay:merch-123:00000987234"
    
    Query: ZRANGEBYLEX typeahead:merchants "[amazon" "[amazon\xff" LIMIT 0 10
    Returns: all merchants with names starting with "amazon"
    
    Popularity ranking: since ZRANGEBYLEX ignores score, encode popularity in the key
    Sort by: first letter → name alphabetical → popularity within same name
    Better approach: separate keys per first_letter for efficiency

  typeahead:users:{user_id} → personal autocomplete (people you've paid)
    Members: "{name}:{user_id}:{frequency_count_padded}"
    Searched first; personalized results rank above global
```

```json
// Elasticsearch: merchant catalog index
{
  "merchant_id": "merch-123",
  "name": "Amazon Pay",
  "name_autocomplete": "Amazon Pay",  // analyzed with edge_ngram tokenizer
  "category": "E_COMMERCE",
  "description": "Online marketplace and payment service",
  "country": "IN",
  "transaction_count": 987234,        // for popularity boost
  "created_at": "2020-01-01T00:00:00Z"
}

// Elasticsearch: payment transaction index (per user; user-scoped shard)
{
  "payment_id": "pay-abc",
  "user_id": "u-12345",               // mandatory filter on all personal queries
  "merchant_name": "Amazon Pay",
  "merchant_id": "merch-123",
  "description": "Order #12345 - Electronics",
  "amount": 5000,                     // in paise
  "currency": "INR",
  "category": "SHOPPING",
  "status": "SUCCESS",
  "paid_at": "2024-01-15T14:00:00Z"
}
```

---

## API Design

```
GET /api/v1/search/typeahead?q=amaz&context=merchant&limit=5
Response 200 (< 50ms):
  {
    "suggestions": [
      {"id": "merch-123", "name": "Amazon Pay", "category": "E_COMMERCE",
       "icon_url": "...", "match_type": "PREFIX"},
      {"id": "merch-456", "name": "Amazon Business", ...}
    ],
    "query_time_ms": 3
  }

GET /api/v1/search?q=amazon+electronics&type=payments&from=2024-01-01&to=2024-01-31
  &amount_min=1000&amount_max=50000&category=SHOPPING&cursor=&limit=20
Response 200 (< 500ms):
  {
    "results": [
      {
        "payment_id": "pay-abc",
        "merchant_name": "Amazon Pay",
        "description": "Order #12345 - <em>Electronics</em>",  // highlighted match
        "amount": 5000,
        "paid_at": "2024-01-15",
        "score": 0.89
      }
    ],
    "total": 23,
    "facets": {
      "by_category": [{"SHOPPING": 15}, {"FOOD": 5}, {"OTHER": 3}],
      "by_merchant": [{"Amazon Pay": 20}, {"Amazon Business": 3}],
      "amount_range": {"min": 100, "max": 45000, "avg": 3200}
    },
    "spelling_suggestion": null,
    "query_time_ms": 287
  }
```

---

## High-Level Architecture

```
Query Pipeline:

Client → Search API
  │
  ├── If typeahead (< 4 chars, context=merchant/user):
  │     1. Normalize: lowercase, trim
  │     2. Redis ZRANGEBYLEX typeahead:{context} "[{q}" "[{q}\xff" LIMIT 10
  │     3. Merge with personalized typeahead:{context}:{user_id}
  │     4. Rank: personal results first; then global by popularity
  │     5. Return in < 5ms (Redis round-trip + merge)
  │
  └── If full search (user submits complete query):
        1. Query Understanding:
           - Spell correction: Levenshtein distance on known terms
           - Entity detection: "amazon" → merchant entity; "jan 2024" → date filter
           - Query expansion: "UPI" → also search "unified payments interface"
        2. Build Elasticsearch query:
           {
             "bool": {
               "must": [
                 {"multi_match": {"query": q, "fields": ["description", "merchant_name^2"]}}
               ],
               "filter": [
                 {"term": {"user_id": current_user_id}},  // personal data isolation
                 {"range": {"paid_at": {"gte": from, "lte": to}}},
                 {"range": {"amount": {"gte": amount_min, "lte": amount_max}}}
               ],
               "boost": {"term": {"merchant_id": favorite_merchants}}  // personalization
             },
             "aggs": {
               "by_category": {"terms": {"field": "category"}},
               "amount_stats": {"stats": {"field": "amount"}}
             }
           }
        3. Execute ES query → results + facets
        4. Re-rank with personalization (transaction frequency boost)
        5. Return paginated results with facets

Indexing Pipeline:

New Entity Created (payment, merchant, user) →
  Service emits to Kafka "search-index-updates" topic →
  Index Consumer:
    Batch 1,000 docs → Elasticsearch bulk API
    Update Redis typeahead sorted sets (ZADD)
  Lag: < 60 seconds from entity creation to searchability
```

---

## Detailed Components

### Inverted Index (How Elasticsearch Works Under the Hood)
```
Document: "Amazon Pay charges for Prime subscription renewal"

Tokenization → lowercasing → stopword removal:
  Tokens: [amazon, pay, charge, prime, subscription, renewal]

Inverted index entry:
  amazon → [doc1, doc5, doc12, ...]
  pay    → [doc1, doc3, doc9, ...]
  prime  → [doc1, doc7, ...]

Query: "amazon prime"
  Retrieve: docs matching "amazon" AND "prime" = intersection
  Score: BM25 (TF-IDF variant) = term frequency × inverse document frequency
  Term frequency: "amazon" appears 3× in doc1 → higher score
  IDF: "amazon" appears in 10K docs → less distinctive than "prime" (100 docs)

Why BM25 beats TF-IDF:
  TF-IDF: score grows unboundedly with term frequency (1× vs 100× both penalized equally)
  BM25: diminishing returns on term frequency (1→2 occurrences: big boost; 10→11: tiny boost)
  BM25 saturation parameter (k1): controls how quickly TF saturates
  BM25 field length normalization (b): longer documents get slight penalty (term appears due to length)
```

### Typeahead Ranking with Redis
```
Problem: ZRANGEBYLEX returns alphabetical results, not popularity-ranked.
Merchants "Amazon Pay" (10M transactions) and "Amazonia Foods" (100 transactions)
both match prefix "amaz" — Amazon Pay should rank first.

Solution: Encode popularity in the key itself, then sort within prefix group.

Key format: "{reversed_popularity_score}:{name}:{id}"
  reversed = 999999999 - actual_score (so highest popularity sorts first alphabetically)
  "000000001:" (9,999,998 transactions) sorts before "999999899:" (100 transactions)

ZADD typeahead:merchants 0 "0000001234566:amazon pay:merch-123"
ZADD typeahead:merchants 0 "9999999900:amazonia foods:merch-456"

ZRANGEBYLEX typeahead:merchants "[amaz" "[amaz\xff" LIMIT 10
→ Returns: amazon pay first (lower string sorts first), then amazonia foods
→ Extract name and ID from each result string

Alternative: separate sorted set per prefix (memory intensive but simpler):
ZADD typeahead:prefix:amaz {popularity_score} {merchant_id}
→ ZREVRANGE typeahead:prefix:amaz 0 9 → top 10 by popularity
→ But: too many keys (one per prefix per entity)
```

### Query Understanding Pipeline
```java
public SearchQuery understand(String rawQuery, UserContext user) {
    // Step 1: Normalize
    String normalized = rawQuery.toLowerCase().trim();

    // Step 2: Spell correction (if query has no results)
    // Use Elasticsearch "did you mean" suggestion or custom Levenshtein on known terms
    // Only trigger if initial query returns 0 results

    // Step 3: Date expression detection
    // "last week" → date range filter; "january 2024" → specific month filter
    DateRange dateRange = DateExpressionParser.parse(normalized);

    // Step 4: Amount detection
    // "over 5000" → amount filter; "between 100 and 1000" → range
    AmountRange amountRange = AmountExpressionParser.parse(normalized);

    // Step 5: Entity detection
    // "amazon" → known merchant; add merchant_id boost
    List<String> detectedMerchants = merchantEntityDetector.detect(normalized);

    // Step 6: Remove extracted entities from text query
    String cleanedQuery = removeExtracted(normalized, dateRange, amountRange);

    return SearchQuery.builder()
        .textQuery(cleanedQuery)
        .dateRange(dateRange)
        .amountRange(amountRange)
        .merchantBoosts(detectedMerchants)
        .userId(user.getUserId())  // always scope to user's personal data
        .build();
}
```

---

## Scaling Strategy
```
Phase 1 (0 → 1K queries/sec):
  Elasticsearch single node; no Redis typeahead
  Basic full-text on PostgreSQL with pg_trgm
  Manual index building on new entities

Phase 2 (1K → 10K queries/sec):
  Elasticsearch cluster (3 nodes; primary + 2 replicas)
  Redis for typeahead (merchant names in sorted set)
  Kafka → ES async indexing

Phase 3 (10K → 100K queries/sec):
  ES sharding by user_id for personal search (user's payments on same shard)
  Redis Cluster for typeahead
  Query cache: Redis cache for top-1K popular queries (cache hit rate > 60%)
  Query understanding service (ML-based intent detection for complex queries)

Phase 4 (semantic search):
  Vector embeddings for semantic matching (not just keyword)
  "coffee" matches "latte" (semantic) not just exact keyword
  Elasticsearch KNN search (dense_vector field + knn query)
  Embedding model: text-embedding-ada-002 or sentence-transformers
```

---

## Reliability + Security

```
Personal data isolation:
  Every payment search query MUST include user_id as mandatory filter
  ES query builder enforces this; cannot be bypassed by client
  Separate ES indices per user (for very large users) or shared with user_id filter

Index recovery:
  If ES cluster data lost: replay all entities from Cassandra/PostgreSQL → ES via Kafka
  Full re-index time: depends on data volume; typically hours to days
  Mitigation: ES snapshot to S3 daily; restore from snapshot + replay last 24h from Kafka

Typeahead injection:
  Merchant names sanitized before adding to Redis keys (no control characters)
  ZRANGEBYLEX query escaping: special characters in user input escaped before range query
```

---

## Observability
```
Metrics:
  typeahead_latency_p99          (target < 50ms; alert > 100ms)
  fulltext_latency_p99           (target < 500ms; alert > 1s)
  es_query_latency_p99           (ES-internal latency; target < 300ms)
  index_freshness_lag_seconds    (entity creation → ES searchable; target < 60s)
  redis_typeahead_hit_rate       (% served by Redis vs ES fallback)
  spell_correction_rate          (how often "did you mean" triggered)
  zero_result_rate               (% of queries returning 0 results; alert > 5%)

Alerts:
  fulltext_latency_p99 > 1s      → ES cluster overloaded; add nodes or replicas
  index_freshness_lag > 120s     → Kafka consumer lag; scale ES indexers
  zero_result_rate > 10%         → Index issue or query understanding bug
```

---

## Chaos Testing
```
Experiment 1: Elasticsearch Cluster Node Failure
  Inject:    Kill one ES data node (1 of 10 shards goes offline)
  Expected:  Replica shards promote; queries continue with slight latency increase
  Expected:  Indexing continues to remaining nodes
  Expected:  Healing: ES auto-reallocates shards to remaining nodes (may take minutes)
  Red flag:  Queries fail immediately (no replica promotion)

Experiment 2: Typeahead Under High Concurrency
  Inject:    70K typeahead queries/sec (peak load)
  Expected:  Redis serves all queries in < 5ms (Redis handles 200K+ ops/sec)
  Expected:  No connection pool exhaustion
  Expected:  p99 latency stays < 50ms including network overhead
  Red flag:  Redis becomes bottleneck; latency > 100ms

Experiment 3: Index Freshness Test
  Inject:    Create 1,000 new merchant records; measure time to appear in typeahead and search
  Expected:  Typeahead: appears within 10s (Redis updated by Kafka consumer)
  Expected:  Full search: appears within 60s (ES bulk index consumer)
  Red flag:  New merchants not searchable after 2 minutes (Kafka consumer lag)
```

---

## Monthly Cost Estimate
```
Scale: 1B documents, 100K queries/sec

Elasticsearch:
  30 nodes × r6g.2xlarge ($500/mo)                   = $15,000/mo
  (30 nodes = 10 shards × 3 replicas; 100M docs each)

Redis (typeahead):
  3 × cache.r6g.xlarge ($260/mo)                     = $780/mo
  (typeahead sorted sets: ~10 GB total for 10M merchants)

Kafka (indexing pipeline):
  3 × kafka.m5.large ($150/mo)                       = $450/mo

Search API instances:
  8 × c5.2xlarge ($280/mo)                           = $2,240/mo
  (Query understanding + ES client)

PostgreSQL (entity metadata, query logs):
  db.r6g.large ($200/mo)                             = $200/mo
─────────────────────────────────────────────────────────────────
Total: ~$18,670/month
ES is 80% of cost — distributed full-text search is compute and storage intensive
```

---

## Architecture Review Checklist
```
□ Source of truth?
  → PostgreSQL/Cassandra: entity data (merchants, users, transactions) — authoritative.
  → Elasticsearch: search index (derived; rebuilt from source on failure).
  → Redis: typeahead cache (derived; rebuilt from ES/PostgreSQL on loss).
  → ES and Redis are both expendable; source systems are the truth.

□ Consistency model?
  → Typeahead: eventual (Kafka → Redis update; < 10s lag).
  → Full-text search: eventual (Kafka → ES; < 60s lag).
  → Source systems: strong (PostgreSQL/Cassandra authoritative).

□ Failure mode?
  → ES down: full-text search unavailable; typeahead continues (Redis).
  → Redis down: typeahead falls to ES (slower but correct); full search unaffected.
  → Kafka consumer lag: index freshness degrades; source systems unaffected.

□ SCC LENS:
  STATE: PostgreSQL/Cassandra (source), ES (search index), Redis (typeahead cache)
  COORDINATION: Kafka fan-out from source → ES indexer + Redis updater
  CONCENTRATION: hot search queries (popular merchants) → query cache in Redis;
                 ES query cache for repeated identical queries
```

---

*Next: Case Study 17 — Monitoring and Alerting System*
