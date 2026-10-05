# Curriculum · Search

[← System Design index](../README.md)

> 8 lessons in **Search**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Search** (8): [Inverted indexes](#inverted-indexes) · [Text analysis and normalization](#text-analysis-and-normalization) · [BM25 relevance ranking](#bm25-relevance-ranking) · [Autocomplete and prefix search](#autocomplete-and-prefix-search) · [Faceted search](#faceted-search) · [Vector similarity search](#vector-similarity-search) · [Approximate nearest neighbors](#approximate-nearest-neighbors) · [Hybrid retrieval and reranking](#hybrid-retrieval-and-reranking)

## Search

### Inverted indexes

*Map each term to the list of documents containing it, so a query intersects small lists instead of scanning every document.*

**Flow:** `Documents` → `Text analysis` → `Posting lists` → `Candidate retrieval` → `Results`

> **The 30-second version**  
> Map terms to the documents containing them, so a query intersects short posting lists instead of scanning the corpus — with positions for phrases and compression to keep it in memory.

**The problem**

Finding documents containing a word by scanning every document is linear in the corpus. At ten million documents averaging a kilobyte each, that is ten gigabytes read per query. A database index on a text column does not help, because a B-tree indexes whole values, and nobody searches for an entire document.

The inverted index turns the mapping around. Instead of document → words, it stores word → documents. A query for a term becomes a lookup of that term's list rather than a scan, and a multi-term query becomes an intersection of a few short lists.

> **Why it is called inverted**  
> A forward index maps a document to its contents, which is how documents are naturally stored. Inverting it maps each term to the documents containing it. That single transposition converts search from a scan proportional to corpus size into a lookup proportional to the number of matching documents — the same reason a book's index beats reading the book.

**Mental model**

The index is a dictionary from term to posting list. Each posting records a document identifier and usually the positions and frequency of that term within it. Query execution is set operations over these lists.

1. **Term dictionary** — The sorted set of all terms, with a pointer to each term's posting list. Kept compact enough to stay in memory.
2. **Posting list** — The document identifiers containing a term, in sorted order, usually with term frequency and positions.
3. **Positions** — Where in the document the term occurs, which is what makes phrase queries possible.
4. **Segment** — An immutable chunk of the index. New documents create new segments; searching merges across them.
5. **Skip pointers** — Structures that let intersection jump forward rather than walking every posting, which is what makes conjunctive queries fast.

> **Intersection order is the whole optimisation**  
> A query for `rare AND common` should start from the rare term's short posting list and check each candidate against the common one, not the reverse. Processing the shortest list first means the number of comparisons is bounded by the rarest term rather than the commonest — which is why term statistics are stored and why query planning in a search engine is mostly about deciding this order.

**How it works**

**Index structure and query execution**

```text
DOCUMENTS
  1: "the quick brown fox"
  2: "the lazy brown dog"
  3: "quick brown foxes jump"

AFTER ANALYSIS (lowercase, stop words removed, stemmed)
  1: quick, brown, fox
  2: lazy, brown, dog
  3: quick, brown, fox, jump

INVERTED INDEX
  brown -> [1, 2, 3]
  dog   -> [2]
  fox   -> [1(pos 3), 3(pos 3)]
  jump  -> [3]
  lazy  -> [2]
  quick -> [1(pos 1), 3(pos 1)]

QUERY  quick AND brown
  quick -> [1, 3]        (2 postings)
  brown -> [1, 2, 3]     (3 postings)
  start with the SHORTER list, probe the longer
  -> [1, 3]

PHRASE QUERY  "quick brown"
  intersect -> [1, 3]
  then check positions: brown at pos(quick)+1 in both
  -> [1, 3]
  -> positions are why phrase search is possible at all
```

1. **Analyse consistently at index and query time** — The same tokenisation, lowercasing and stemming must apply to both, or a query for “running” will not match an indexed “run”. Mismatched analysers are the most common cause of “search returns nothing”.
2. **Compress posting lists aggressively** — Document ids are sorted, so delta encoding plus variable-byte or bitpacking reduces them enormously — often to a byte or two per posting, which is what keeps the index in memory.
3. **Keep segments immutable and merge in the background** — New documents form new segments; deletions are tombstones. Periodic merging compacts them, which is the same log-structured trade as an LSM tree.
4. **Use skip pointers for conjunctions** — Without them, intersecting a two-element list with a ten-million-element list walks ten million postings. With them it performs a handful of jumps.
5. **Accept near-real-time rather than immediate visibility** — A document is searchable once its segment is flushed and reopened, typically within a second. Immediate visibility would mean writing a segment per document.
6. **Store what you need to rank, not just to match** — Term frequency and document length must be in the index, because computing them at query time would require reading the documents.

**Why compression matters more than it sounds**

```text
POSTING LIST for a common term: 5,000,000 document ids
  raw 32-bit ids:  5M x 4 B = 20 MB

DELTA ENCODING (store gaps, not absolute ids)
  ids sorted: 4, 9, 17, 22, 30...
  deltas:     4, 5,  8,  5,  8...
  -> small numbers

VARIABLE-BYTE / BITPACKING on the deltas
  typical: 1-2 bytes per posting
  -> 5M x 1.5 B = 7.5 MB   (nearly 3x smaller)

WHY IT MATTERS
  the index must fit in memory to be fast
  3x smaller means 3x more of the index cached
  intersection reads fewer bytes, so it is also FASTER
  -> compression improves both capacity and latency,
     which is unusual
```

> **Analysis decisions are effectively permanent**  
> Changing the analyser — adding stemming, changing tokenisation, altering stop words — invalidates the entire index, because existing postings were produced by the old rules. The corpus must be fully reindexed, which for a large index is hours or days and requires running both versions during the transition. This makes analysis a design decision to get right early rather than tune later.

**Worked example**

Sizing an index for ten million documents and understanding where the cost lives.

**Index size and query cost**

```text
CORPUS  10M documents, ~500 unique terms each after analysis

POSTINGS
  10M x 500 = 5 billion postings
  compressed at ~1.5 bytes each = ~7.5 GB

TERM DICTIONARY
  ~2M unique terms (Zipf: a few very common, a long tail)
  term string + pointer ~ 30 bytes
  = ~60 MB   <- small; stays in memory easily

POSITIONS (needed for phrase queries)
  roughly doubles or triples the posting size
  -> ~20 GB with positions
  -> a real decision: phrase queries are expensive in space

QUERY COST  "distributed systems"
  distributed -> 50,000 postings
  systems     -> 2,000,000 postings
  intersect starting from the shorter list
  with skip pointers: ~50,000 probes, not 2,000,000
  -> a few milliseconds

THE ZIPF SHAPE
  the top 100 terms cover a large fraction of all postings
  stop-word removal cuts index size substantially -
  but breaks phrase queries containing those words
  ("to be or not to be" becomes unsearchable)
```

| Metric | Value | Note |
|---|---|---|
| Postings | 5 billion | ~7.5 GB compressed |
| With positions | ~20 GB | **phrase queries cost space** |
| Dictionary | 60 MB | always in memory |
| Query | ~50k probes | not 2M |

> **Stop words are a genuine trade, not an obvious win**  
> Removing the hundred commonest terms shrinks the index substantially, because their posting lists are enormous. But it makes any phrase containing them unsearchable — a famous quotation composed entirely of stop words disappears. Modern engines generally keep stop words and rely on compression and skip pointers instead, which is the better trade once memory is cheap and correctness is visible.

**When to use it**

- **Full-text search** over documents, articles, products, messages or logs.
- **Any query matching on words rather than whole values**, which relational indexes cannot serve.
- **Faceted and filtered search**, where posting lists intersect naturally with filter sets.
- **Log and event search**, where the corpus is enormous and queries are term-based.
- **As a secondary index alongside a source of truth**, populated asynchronously.

**When to avoid it**

- **Do not use it as a system of record.** It is a derived index, rebuildable from the source, and lacks transactional guarantees.
- **Do not use it for exact-value lookups** that a relational index serves better and more cheaply.
- **Do not expect immediate visibility**, since documents become searchable on segment refresh rather than on write.
- **Do not change the analyser casually**, which invalidates the index and forces a full reindex.
- **Do not index fields you never search on**, which costs space and write throughput for nothing.

**Advantages**

- **Query cost is proportional to matching documents**, not to corpus size.
- **Compression is extremely effective**, because sorted document ids delta-encode to a byte or two.
- **Multi-term queries are set operations**, which are fast and composable with filters.
- **Positions enable phrase and proximity queries**, which no value-based index can do.
- **Segments are immutable**, which makes caching, replication and backup straightforward.

**Disadvantages**

- **Write amplification**: one document produces hundreds of posting updates across many terms.
- **Near-real-time, not real-time**, with visibility delayed by the refresh interval.
- **Deletions are tombstones** until merged, so a delete-heavy corpus degrades until compaction.
- **Analysis choices are effectively permanent** without a full reindex.
- **Memory-hungry**, since performance depends on keeping the index resident.
- **Not transactional**, so it must be treated as derived data with a rebuild path.

**Trade-offs**

**Indexing decisions and their cost**

| Decision | Benefit | Cost |
|---|---|---|
| Store positions | Phrase and proximity queries | 2–3× index size |
| Remove stop words | Much smaller index | Phrases with common words become unsearchable |
| Aggressive stemming | Better recall | Worse precision; “university” matches “universe” |
| Frequent refresh | Fresher results | More segments, more merging, higher CPU |
| Index every field | Anything is searchable | Space and write throughput for unused fields |

The refresh interval is the most commonly mistuned setting: shortening it to make documents visible faster multiplies segment count and merge load, which degrades query latency. A one-second refresh is usually right, and demands for sub-second visibility should be questioned rather than configured.

**How it fails**

**Inverted index failures**

| Failure | Cause | Fix |
|---|---|---|
| Query returns nothing for obvious terms | Index-time and query-time analysers differ | Use the same analyser; test with an analyse endpoint |
| Phrase queries do not work | Positions not indexed, or stop words removed | Index positions; reconsider stop-word removal |
| Index much larger than expected | Positions plus indexing unused fields | Index only searched fields; reconsider positions per field |
| Search slow after heavy deletion | Tombstones not yet merged | Force merge during quiet periods; avoid delete-heavy patterns |
| Documents not searchable immediately | Refresh interval | Expected; explain it, or read from the source of truth for read-after-write |
| Cluster unstable after a refresh change | Too many small segments; merge pressure | Longer refresh; tune merge policy |
| Reindex required for a small change | Analyser modification | Plan analysis early; run parallel indexes during migration |

**Limits**

> **Sizing numbers**
>
> - **Compressed postings**: roughly 1–2 bytes each with delta encoding and bitpacking.
> - **Positions** typically double or triple posting size — a per-field decision.
> - **Term dictionary** is small relative to postings and should stay fully in memory.
> - **Refresh interval** of around one second is the usual default; shorter multiplies merge cost.
> - **Query cost** is bounded by the rarest term's posting list when intersection starts there.

**Alternatives**

| Structure | Best for | Limitation |
|---|---|---|
| Inverted index | Term-based full-text search | Not transactional; write amplification |
| B-tree index | Exact and range lookups on whole values | Cannot search within text |
| Trigram index | Substring and fuzzy matching | Larger; less precise |
| Vector index | Semantic similarity | No exact term matching |
| Full scan with regex | Small corpora, ad-hoc patterns | Linear in corpus size |
| Columnar store | Analytical aggregation | Not for relevance ranking |

Trigram indexes deserve a mention for substring and typo-tolerant matching, which an inverted index over whole terms cannot do — searching for `ostgre` finds nothing in a term index but matches trigrams. They are larger and less precise, so they are usually a complement rather than a replacement.

**In real systems**

- **Lucene** is the foundation beneath Elasticsearch, OpenSearch and Solr, and its segment-based immutable design is why those systems behave as they do.
- **PostgreSQL's GIN indexes** implement inverted indexing for full text and JSON containment inside a relational database.
- **Log search platforms** index enormous corpora with term-based queries, which is why they use inverted indexes rather than scanning.
- **Segment merging in Lucene** mirrors LSM compaction, with the same read, write and space amplification trade-offs.
- **Product search** combines inverted indexes for term matching with filters and facets expressed as additional posting-list intersections.

**Common mistakes**

- **Different analysers at index and query time**, so queries match nothing.
- **Removing stop words** and losing phrase queries that contain them.
- **Indexing every field**, paying space and write cost for fields nobody searches.
- **Treating the search index as a source of truth**, with no rebuild path.
- **Very short refresh intervals**, multiplying segments and merge load.
- **Delete-heavy workloads**, accumulating tombstones that degrade queries until merged.
- **Changing the analyser in place** without planning the reindex.

**The staff-level view**

Search indexes are derived data, and treating them as anything else is the source of most operational trouble.

- **Treat the index as rebuildable, never authoritative.** The rebuild path must exist and be exercised, because analyser changes, mapping mistakes and corruption all require it.
- **Settle analysis early.** It is effectively permanent without a full reindex, so tokenisation, stemming and stop-word decisions deserve real attention before the corpus is large.
- **Question sub-second freshness requirements.** Shorter refresh intervals multiply segment and merge load, and near-real-time is almost always sufficient once someone asks what the requirement is actually for.
- **Index only what is searched.** Space and write throughput are consumed by every indexed field, and unused fields are a silent, permanent cost.
- **Plan for reindex during migration**, running old and new indexes in parallel with a switch, since a full reindex of a large corpus takes hours or days.

**Go deeper**

An inverted index transposes the natural document-to-terms mapping into terms-to-documents. A query becomes a lookup of posting lists rather than a scan, so cost scales with matching documents rather than corpus size. Multi-term queries are set intersections, and storing term positions additionally enables phrase and proximity matching.

Two implementation properties make it practical. Posting lists compress to a byte or two per entry using delta encoding on sorted document ids, which keeps the index memory-resident and simultaneously reduces bytes read during intersection. And skip pointers let a conjunction start from the rarest term's short list and jump through the common one, so a query costs the rare term's length rather than the common term's.

The constraints follow from the structure. Analysis — tokenisation, stemming, stop words — must match exactly between index and query time, and changing it invalidates the entire index and requires a full reindex. Segments are immutable, so writes create new segments and deletions are tombstones until merged, giving near-real-time rather than immediate visibility. And the index is derived data: rebuildable, non-transactional, and never a system of record.

The inverted index is the structure that makes text search tractable, and its properties explain nearly everything about how search systems behave operationally.

**The transposition and what it buys.** Storing term to document rather than document to terms converts a scan proportional to corpus size into a lookup proportional to matching documents. Multi-term conjunctions become list intersections, and because posting lists are sorted, intersection can use skip pointers to jump rather than walk. The critical optimisation is starting from the rarest term: a query combining a fifty-thousand-posting term with a two-million-posting term should probe from the smaller side, bounding work by the rare term. Term statistics exist in the index precisely to make this decision possible.

**Compression is load-bearing.** Sorted document identifiers delta-encode to small numbers, which variable-byte encoding or bitpacking stores in one or two bytes rather than four. The threefold reduction matters more than it appears, because search latency depends on the index being memory-resident — a smaller index means more of it cached and fewer bytes read during intersection, so compression improves capacity and latency simultaneously, which is unusual.

**Positions are the expensive feature.** Phrase and proximity queries require knowing where each term occurs in each document, which typically doubles or triples posting size. This is worth treating per field rather than globally: titles and short fields benefit, long descriptions matched only on individual terms may not. The related decision is stop words — discarding the commonest terms shrinks the index substantially since their posting lists dominate, but it makes phrases composed of common words unsearchable. Modern engines generally retain them and rely on compression and skipping instead, because the correctness loss is visible and the space is cheap.

**Segments make it log-structured.** New documents form new immutable segments; deletions write tombstones; background merging compacts them. This is the same structure and the same trade-offs as an LSM tree, including the consequence that a delete-heavy corpus accumulates tombstones and degrades reads until merged. It also means documents become searchable at segment refresh rather than at write, giving near-real-time visibility — typically a second. Shortening the refresh interval to chase immediacy multiplies segment count and merge load, degrading query latency, which makes it one of the most commonly mistuned settings.

**Analysis is effectively permanent.** The index contains whatever the analyser produced, so index-time and query-time analysis must match exactly — a stemmed index queried without stemming returns nothing, which is the most common cause of inexplicably empty search results. Changing tokenisation, stemming or stop words invalidates every existing posting and requires a full reindex, which for a large corpus takes hours or days and usually means running two indexes in parallel during the switch. That makes analysis a decision to get right early rather than tune later.

**The index is derived data.** It is not transactional, is eventually consistent with its source, and will need rebuilding for analyser changes, mapping errors, corruption or version upgrades. Treating it as a system of record means data loss when any of those occur. The discipline is a database as the source of truth, asynchronous propagation to the index, and a rebuild pipeline that is exercised rather than assumed — because the first attempt at a full reindex should not happen during an incident, and its duration and capacity requirements need to be known in advance.

**Prove it — interview questions**

1. **[Basic] What does an inverted index invert?**

   <details><summary>Model answer</summary>

   The natural mapping. Documents normally map to their contents — document one contains these words. An inverted index maps each term to the documents containing it. That transposition means a query for a word becomes a lookup of that word's posting list rather than a scan of every document, so cost is proportional to the number of matching documents rather than to the corpus size.

   </details>

2. **[Basic] Why do posting lists compress so well?**

   <details><summary>Model answer</summary>

   Because document identifiers are stored in sorted order, so you can store the gaps between them rather than the absolute values. Those gaps are small numbers, which variable-byte encoding or bitpacking represents in one or two bytes instead of four. That routinely gives around a threefold reduction, which matters disproportionately because the index needs to be resident in memory — smaller means more of it cached, and it also means less data read during intersection, so compression improves both capacity and latency at once.

   </details>

3. **[Senior] Why does intersection order matter?**

   <details><summary>Model answer</summary>

   Because the number of comparisons is bounded by whichever list you walk. A query for a rare term and a common term should iterate the rare term's short posting list and probe the common one, using skip pointers to jump rather than scanning. Done that way a query might perform fifty thousand probes; done in reverse it walks two million postings. This is why term statistics are stored in the index and why query planning in a search engine is largely about choosing this order.

   </details>

4. **[Senior] Why must index-time and query-time analysis match?**

   <details><summary>Model answer</summary>

   Because the index contains whatever the analyser produced, and a query is matched against those stored terms. If documents were stemmed at index time so “running” became “run”, but the query is not stemmed, then searching for “running” looks for a term that does not exist in the index and returns nothing. This mismatch is the most common cause of a search that inexplicably returns no results, and because the stored postings reflect the old analysis, changing the analyser requires reindexing the entire corpus rather than just changing a setting.

   </details>

5. **[Staff] What are the space implications of supporting phrase queries?**

   <details><summary>Model answer</summary>

   Phrase queries need positional information — where each term occurs within each document — because after intersecting the posting lists you must verify that the terms appear adjacently. Storing positions typically doubles or triples the size of the postings, which is a substantial cost when the index needs to be memory-resident. It is worth treating as a per-field decision: a title field almost certainly wants positions, while a long description field that is only ever matched on individual terms may not. The related trade is stop words — removing them shrinks the index considerably since their posting lists are enormous, but it makes phrases composed of common words unsearchable, which is why modern engines generally keep them and rely on compression instead.

   </details>

6. **[Principal] How should a search index relate to the system of record?**

   <details><summary>Model answer</summary>

   As strictly derived data with a proven rebuild path. The index is not transactional, is eventually consistent with the source, and will at various points need to be rebuilt — for an analyser change, a mapping mistake, a corrupted segment, or a version upgrade. Treating it as authoritative means data loss when any of those happen. Practically, that means the source of truth is a database, changes propagate to the index asynchronously, and there is an exercised pipeline that can rebuild the whole index from the source. I would also insist that the rebuild is tested rather than assumed, because a full reindex of a large corpus takes hours or days and the first time anyone attempts it should not be during an incident — and because a migration typically requires running the old and new index in parallel and switching, which is a capacity question that needs answering in advance.

   </details>

---

### Text analysis and normalization

*Turn raw text into the tokens that go in the index, choosing rules that make intended matches succeed and unintended ones fail.*

**Flow:** `Raw text` → `Tokenizer` → `Normalization` → `Search tokens` → `Query matching`

> **The 30-second version**  
> Turn text into tokens through a pipeline of filtering, tokenising and normalising — applied identically at index and query time, and chosen per field from the nature of the content.

**The problem**

A user searches for “running shoes” and expects to match a product titled “Men's Running Shoe — Nike”. Literal matching fails on almost every axis: case, plurality, punctuation, word order and the extra terms. Something must reduce both the document and the query to a common form where they meet.

That reduction is analysis, and every choice within it trades recall against precision. Aggressive normalisation matches more things, including things the user did not mean. Conservative normalisation matches only near-exact text, which users experience as the search being broken.

> **Analysis defines what “the same” means**  
> The index stores tokens, not text, so the analyser is a declaration of which strings are considered equivalent. Deciding that “Running” and “run” are the same token, or that “C++” and “C” are the same, or that “iphone-15” is one token or two, determines every match and every miss. It is the most consequential and least visible decision in a search system.

**Mental model**

Analysis is a pipeline: characters are filtered, split into tokens, then each token is transformed. The same pipeline must run over documents at index time and over queries at search time, producing tokens that meet.

1. **Character filters** — Operate on the raw string before tokenisation — stripping HTML, mapping characters, normalising Unicode forms.
2. **Tokenizer** — Splits text into tokens. The rules for punctuation, hyphens, numbers and non-Latin scripts are all decided here.
3. **Lowercasing** — Almost always applied, but it makes case-sensitive distinctions impossible — which matters for codes and identifiers.
4. **Stemming or lemmatisation** — Reduces inflected forms to a common root. Stemming is crude and fast; lemmatisation is linguistically correct and slower.
5. **Synonyms and stop words** — Expand or discard tokens according to explicit lists, which is where domain knowledge enters.

> **Language is not a setting you can add later**  
> Tokenisation, stemming and stop words are all language-specific, and many are script-specific. Chinese and Japanese have no spaces, so tokenisation requires segmentation. German compounds words, so decompounding is needed. A system built assuming English and later required to support other languages usually needs per-language fields and a full reindex, because one analyser cannot serve them all.

**How it works**

**The pipeline, and what each stage decides**

```text
INPUT  "The Men's Running-Shoes (Size 10) - Nike®"

CHARACTER FILTERS
  strip HTML, normalise Unicode, map (R) -> (r)
  -> "The Men's Running-Shoes (Size 10) - Nike(r)"

TOKENIZER  (standard: split on non-alphanumeric)
  -> [The, Men's, Running, Shoes, Size, 10, Nike, r]
  NOTE: the hyphen split "Running-Shoes" into two tokens.
        A different tokenizer would keep it as one.
        This single choice changes what matches.

LOWERCASE
  -> [the, men's, running, shoes, size, 10, nike, r]

POSSESSIVE / APOSTROPHE HANDLING
  -> [the, men, running, shoes, size, 10, nike, r]

STOP WORDS (if enabled)
  -> [men, running, shoes, size, 10, nike, r]

STEMMING
  -> [men, run, shoe, size, 10, nike, r]

INDEXED TOKENS: men, run, shoe, size, 10, nike, r

QUERY "running shoes" -> [run, shoe] -> MATCHES
QUERY "Running Shoe"  -> [run, shoe] -> MATCHES
QUERY "shoes running" -> [shoe, run] -> MATCHES (unordered)
```

1. **Choose the tokenizer from the domain, not the default** — Product codes, identifiers, URLs and version numbers all break under standard tokenisation. `iPhone-15` becoming two tokens may be correct or disastrous depending on what users type.
2. **Prefer light stemming for precision, heavier for recall** — Aggressive stemmers conflate unrelated words — `university` and `universe` share a stem under some algorithms. Where precision matters, a lighter stemmer or a lemmatiser is better.
3. **Index the same field several ways** — A `title` field can be indexed raw, analysed, and as edge n-grams. Queries then target whichever behaviour is appropriate, and ranking can weight exact matches above stemmed ones.
4. **Treat synonyms as a maintained asset** — They encode domain knowledge — `tv` and `television`, `laptop` and `notebook` — and they are the highest-leverage relevance improvement available in most product searches.
5. **Apply synonym expansion at query time where possible** — Expanding at index time bakes the list in and requires a reindex to change; expanding at query time lets synonyms evolve, at some query cost.
6. **Test analysis explicitly** — Search engines expose an analyse endpoint showing the tokens a string produces. Using it during development prevents the entire class of “the query matches nothing” bugs.

**Recall versus precision, decided by analysis**

```text
AGGRESSIVE (high recall, low precision)
  heavy stemming, broad synonyms, fuzzy matching
  query "shoe" matches: shoes, shoe, shoeing, shoed
                        and via synonyms: footwear, sneaker
  + users almost always find something
  - results include things they did not mean
  use: discovery, browsing, large catalogues

CONSERVATIVE (high precision, low recall)
  minimal stemming, no synonyms, exact tokens
  query "shoe" matches: shoe, shoes
  + results are clearly relevant
  - zero-result queries are common, which users read as
    "this search is broken"
  use: technical documentation, code, legal text, identifiers

THE FIELD-SPECIFIC ANSWER
  index the same content multiple ways and weight them:
    title.exact     (no stemming)    boost 5
    title.analysed  (stemmed)        boost 2
    body.analysed   (stemmed)        boost 1
  -> an exact title match outranks a stemmed body match
  -> recall from the analysed fields, precision from ranking
```

> **Zero results are worse than imperfect results**  
> Users interpret an empty result set as the search being broken and frequently leave. Users interpret a list containing some irrelevant items as a search that needs refining. That asymmetry argues for favouring recall in the analyser and recovering precision through ranking — matching broadly, then ordering well — rather than matching narrowly and returning nothing.

**Worked example**

Analysing a product catalogue, and the specific decisions that determine whether search works.

**Field-by-field analysis decisions**

```text
PRODUCT  "Sony WH-1000XM5 Wireless Noise-Cancelling Headphones"

PROBLEM 1  the model number
  standard tokenizer -> [wh, 1000xm5]
  user types "WH1000XM5" -> [wh1000xm5] -> NO MATCH
  user types "XM5"       -> [xm5]       -> NO MATCH
  FIX: a dedicated model-number field with a tokenizer
       that preserves alphanumerics, plus edge n-grams
       so partial model numbers match

PROBLEM 2  the hyphen
  "Noise-Cancelling" -> [noise, cancelling]
  query "noise cancelling" matches
  query "noisecancelling" does not
  FIX: acceptable; or add a shingle/concatenation filter

PROBLEM 3  British vs American spelling
  indexed "cancelling"; user types "canceling"
  FIX: synonym pair, or a stemmer that normalises both

PROBLEM 4  the brand
  "Sony" must match exactly and rank highly
  stemming must not touch it
  FIX: a keyword (unanalysed) brand field for filtering,
       plus the analysed title for text matching

RESULTING MAPPING
  title          analysed, stemmed, synonyms at query time
  title.exact    unanalysed, high boost
  brand          keyword, used for filters and facets
  model          alphanumeric-preserving + edge n-grams
  description    analysed, stemmed, low boost
```

| Metric | Value | Note |
|---|---|---|
| Fields | one source | indexed 4 ways |
| Model numbers | n-grams | **partial matching** |
| Brand | keyword | exact, facetable |
| Synonyms | query time | editable without reindex |

> **Index the same content several ways rather than choosing one analysis**  
> The instinct is to pick the analyser that best balances the trade-offs. The better approach is to index a field multiple times with different analysis and let ranking decide: an exact title match scores highest, a stemmed title match next, a stemmed description match lowest. That gives recall from the permissive fields and precision from the scoring, rather than forcing a single compromise that is wrong for some queries.

**When to use it**

- **Every full-text search system**, since analysis is not optional — the default is simply a choice someone else made.
- **Multi-language corpora**, where per-language analysis is required rather than optional.
- **Domain-specific vocabulary**, where synonyms encode knowledge users will not type.
- **Identifiers and codes**, which need deliberately different analysis from prose.
- **Autocomplete and prefix matching**, which need n-gram analysis rather than whole-token matching.

**When to avoid it**

- **Do not use the default analyser for identifiers, codes or URLs**, which standard tokenisation destroys.
- **Do not apply aggressive stemming to short critical fields** such as brand or product names.
- **Do not bake synonyms in at index time** unless they are genuinely stable, since changing them requires a reindex.
- **Do not assume one analyser serves multiple languages**; tokenisation and stemming are language-specific.
- **Do not change analysis without planning the reindex**, which for a large corpus is a project.

**Advantages**

- **Matches what users mean rather than what they typed**, which is the entire purpose of search.
- **Synonyms encode domain knowledge** and are usually the highest-value relevance improvement available.
- **Multi-field indexing** gives recall and precision simultaneously rather than forcing a compromise.
- **Language-specific analysis** makes non-English search work properly rather than approximately.
- **Cheap at query time**, since the expensive work happens once at index time.

**Disadvantages**

- **Effectively permanent** — changes require a full reindex.
- **Aggressive normalisation loses information**, conflating words that are genuinely distinct.
- **Language-specific**, so multilingual support multiplies configuration and index size.
- **Invisible when wrong**: queries return nothing or the wrong things, with no error to diagnose.
- **Synonym lists require ongoing maintenance**, and stale ones actively harm relevance.

**Trade-offs**

**Normalisation choices**

| Technique | Improves | Costs |
|---|---|---|
| Lowercasing | Case-insensitive matching | Cannot distinguish acronyms from words |
| Stemming | Recall across word forms | Precision; conflates unrelated words |
| Lemmatisation | Correct root forms | Slower; needs language models |
| Stop-word removal | Index size, query speed | Phrases with common words break |
| Synonyms | Recall for domain vocabulary | Maintenance; can introduce noise |
| N-grams | Partial and fuzzy matching | Large index growth |

> **Framing the design**  
> “I'd index the title three ways — exact, stemmed, and edge n-grams for prefix matching — and let ranking prefer exact matches over stemmed ones. Synonyms go at query time so the merchandising team can edit them without a reindex, and model numbers get their own field with a tokenizer that preserves alphanumerics, because standard tokenisation splits them and users type them without punctuation.”

**How it fails**

**Analysis failures**

| Symptom | Cause | Fix |
|---|---|---|
| Query returns nothing for an obvious term | Analyser mismatch between index and query | Same analyser both sides; verify with an analyse endpoint |
| Model numbers unsearchable | Standard tokenizer splitting on punctuation | Dedicated field with alphanumeric-preserving tokenizer |
| Unrelated results | Over-aggressive stemming conflating words | Lighter stemmer or lemmatiser; boost exact matches |
| Phrases fail | Stop words removed | Retain stop words; rely on compression |
| Non-English search barely works | English analyser applied to all languages | Per-language fields and analysers |
| Relevance degrades over time | Synonym list unmaintained | Treat synonyms as an owned, reviewed asset |
| Cannot fix relevance without a reindex | Synonyms and rules applied at index time | Move to query-time expansion where possible |

**Limits**

> **Practical guidance**
>
> - **Analysis must match exactly** between index and query time, or matching silently fails.
> - **Changing analysis requires a full reindex** — hours to days for a large corpus.
> - **Query-time synonym expansion** costs query performance but allows editing without reindexing.
> - **Edge n-grams** for prefix matching grow the index substantially — scope them to short fields.
> - **Per-language fields** multiply index size; scope them to the languages actually served.

**Alternatives**

| Approach | Handles | Limitation |
|---|---|---|
| Token analysis | Word-form and case variation | Not semantic; not typo-tolerant |
| Fuzzy matching (edit distance) | Typos | Expensive; imprecise on short terms |
| N-grams / trigrams | Substrings and partial matches | Large index |
| Phonetic algorithms | Sound-alike names | Language-specific; noisy |
| Vector embeddings | Semantic similarity | No exact matching; opaque |
| Query rewriting / spell correction | Misspellings and intent | Separate system to maintain |

Modern search increasingly combines token analysis with vector embeddings: the inverted index handles exact and lexical matching while embeddings handle semantic similarity, with results fused. Neither replaces the other — a user searching for a specific model number needs exact matching, while one describing a problem needs semantics.

**In real systems**

- **Elasticsearch and Solr analyser chains** expose this pipeline directly as character filters, tokenizer and token filters, which is why their documentation reads as a catalogue of these decisions.
- **E-commerce search** relies heavily on curated synonym lists, which are typically owned by merchandising rather than engineering because they encode commercial knowledge.
- **CJK segmentation** requires dedicated tokenizers, since there are no spaces between words — a reminder that tokenisation is not universal.
- **PostgreSQL text search configurations** bundle a parser, dictionaries and stemmers per language, making the language-specific nature explicit.
- **Multi-field mappings** — indexing one source field several ways — are the standard production pattern for balancing recall and precision.

**Common mistakes**

- **Different analysis at index and query time**, producing silent non-matching.
- **Default tokenizer on identifiers and codes**, splitting them into unsearchable fragments.
- **Aggressive stemming on brand and product names**, conflating distinct entities.
- **Stop-word removal**, breaking phrase queries that contain them.
- **One analyser for all languages**, making non-English search barely function.
- **Index-time synonyms**, requiring a reindex to improve relevance.
- **Never testing analysis output**, so token-level bugs are found by users.

**The staff-level view**

Analysis is where search quality is actually determined, and it is usually configured once by whoever set up the cluster and never revisited.

- **Treat the analyser as a product decision**, not infrastructure configuration. It determines what users can find, and the defaults encode assumptions about English prose that may not match your content at all.
- **Index important fields multiple ways** and resolve the recall-precision tension in ranking rather than in analysis, which avoids a single compromise that is wrong for some queries.
- **Move synonyms and rules to query time** where the engine allows, so relevance can be improved without a multi-hour reindex — which is the difference between iterating weekly and iterating quarterly.
- **Own the synonym list as a maintained asset** with a clear owner, usually outside engineering, since it encodes commercial and domain knowledge.
- **Budget for reindexing as a routine operation**, because analysis will change, and a team that cannot reindex confidently cannot improve relevance.

**Go deeper**

Analysis converts raw text into the tokens the index actually stores: character filtering, tokenisation, lowercasing, stemming, and synonym or stop-word handling. Because the index contains only the analyser's output, the same pipeline must run over queries — a stemmed index queried without stemming returns nothing, silently, which is the most common search failure there is.

Every choice trades recall against precision. Aggressive stemming and broad synonyms match more, including things the user did not mean; conservative analysis matches only near-exact text and produces empty result sets, which users read as the search being broken. The resolution is not to choose but to index the same content several ways — exact, stemmed, n-grammed — and let ranking prefer exact matches, giving recall from the fields and precision from the scoring.

Analysis must also be chosen per field from the content. Standard tokenisation destroys model numbers and identifiers by splitting on punctuation; brand names should be unanalysed keywords so they can drive facets; prose wants stemming and synonyms. And because changing analysis invalidates the index, prefer query-time synonym expansion where possible so relevance can be improved without a multi-hour reindex.

Analysis is the most consequential decision in a search system and the least visible: it defines which strings are considered the same, and therefore what is findable at all.

**The pipeline and its decisions.** Character filters operate on the raw string — stripping markup, normalising Unicode, mapping symbols. The tokenizer splits text, and its treatment of punctuation, hyphens, numbers and scripts determines whether `WH-1000XM5` is one token or several. Lowercasing is nearly universal but forecloses case-sensitive distinctions. Stemming reduces inflected forms crudely and quickly, while lemmatisation does so correctly and slowly. Synonyms and stop words then add or remove tokens according to explicit lists. Each stage is a policy about equivalence.

**Index-time and query-time analysis must match exactly.** The index holds only tokens, so a query is matched against those. Stemming documents but not queries means searching for a term that does not exist in the index, producing an empty result with no error — which is why this is the most common and most confusing failure in search, and why engines expose an analyse endpoint that should be used routinely during development.

**Recall and precision need not be traded globally.** The instinct is to choose one analyser balancing them; the better approach is multi-field indexing. The same source content is indexed exact, stemmed, and perhaps as edge n-grams, and ranking weights an exact title match above a stemmed title match above a stemmed description match. Recall comes from the permissive fields, precision from the scoring. The asymmetry that justifies leaning toward recall is behavioural: users interpret zero results as a broken search and leave, while they interpret imperfect results as needing refinement.

**Analysis is content-specific, not system-wide.** Prose benefits from stemming and synonyms. Identifiers, model numbers, URLs and codes are destroyed by standard tokenisation and need alphanumeric-preserving tokenisation plus n-grams for partial matching. Brand names should be unanalysed keywords, both to prevent stemming from conflating them and so they can drive filters and facets. And language matters structurally: tokenisation, stemming and stop words are all language-specific, and scripts without spaces require segmentation — so a system built assuming English generally needs per-language fields and a full reindex to support anything else.

**Configuration placement determines iteration speed.** Anything applied at index time is baked into the stored tokens and can only be changed by reindexing, which for a large corpus is hours or days. Moving synonyms and rules to query time costs some query performance but allows relevance to be improved in an afternoon rather than a quarter. Since synonym lists — encoding domain vocabulary users will not type — are typically the highest-leverage relevance improvement available, and since they belong to whoever understands the domain rather than to engineering, that iteration speed matters more than the query cost.

**Why analysis limits search quality in practice.** It is usually set once by whoever provisioned the cluster, using defaults designed for English prose, and never revisited. Every later relevance effort — boosting, personalisation, learning to rank — operates only on documents that matched, so anything the analyser rendered unfindable is invisible to all of it. Teams that have not made reindexing routine cannot change it, so their relevance work drifts toward whatever they can adjust quickly. Making reindex an unremarkable operation is therefore a prerequisite for improving search at all.

**Prove it — interview questions**

1. **[Basic] What does an analyser do?**

   <details><summary>Model answer</summary>

   It turns raw text into the tokens actually stored in the index. The pipeline typically strips or maps characters, splits the text into tokens, lowercases them, reduces inflected forms to a root, and applies synonym or stop-word rules. The same pipeline must run over queries, so that a search for “Running Shoes” produces the same tokens as the indexed “running-shoe” and they meet.

   </details>

2. **[Basic] Why does analysis have to match between indexing and querying?**

   <details><summary>Model answer</summary>

   Because the index contains only what the analyser produced, and a query is matched against those stored tokens. If documents were stemmed so “running” became “run”, but the query is not stemmed, the search looks for a token that does not exist and returns nothing. There is no error — just an empty result set — which makes this the most common and most confusing failure in search systems, and why engines provide an analyse endpoint to inspect the tokens a string produces.

   </details>

3. **[Senior] How do you balance recall and precision through analysis?**

   <details><summary>Model answer</summary>

   By not choosing. Rather than picking one analyser that compromises between them, index the same content several ways — an exact unanalysed version, a stemmed version, perhaps n-grams for prefix matching — and let ranking resolve the tension. An exact title match is boosted above a stemmed title match, which is boosted above a stemmed description match. That gives recall from the permissive fields and precision from the scoring. The asymmetry that justifies favouring recall is that users read zero results as the search being broken, while they read imperfect results as needing refinement.

   </details>

4. **[Senior] Why prefer query-time synonym expansion?**

   <details><summary>Model answer</summary>

   Because index-time expansion bakes the synonym list into the stored tokens, so changing it requires reindexing the entire corpus — which for a large index is hours or days. That turns relevance improvement into a quarterly project rather than a weekly iteration. Query-time expansion costs some query performance, since the query expands into more terms, but it means a merchandising team can add a synonym pair and see the effect immediately. Given that synonym lists are usually the highest-leverage relevance improvement available, the ability to iterate on them quickly is worth the query cost.

   </details>

5. **[Staff] How would you handle product model numbers in search?**

   <details><summary>Model answer</summary>

   With a dedicated field and a different analyser, because standard tokenisation destroys them. A model like `WH-1000XM5` splits on the hyphen into fragments, so a user typing `WH1000XM5` or just `XM5` matches nothing. I would index model numbers in their own field with a tokenizer that preserves alphanumeric sequences, plus edge n-grams so partial model numbers match as the user types, and keep them out of the stemmed general-text analysis entirely. More broadly this illustrates the principle that analysis should be chosen per field from the nature of the content: prose wants stemming and synonyms, identifiers want exactness and partial matching, and brand names want to be unanalysed keywords so they can also drive facets and filters.

   </details>

6. **[Principal] Why is analysis often the limiting factor on search quality?**

   <details><summary>Model answer</summary>

   Because it determines what is findable at all, and it is usually configured once, by whoever provisioned the cluster, using defaults designed for English prose. Every subsequent relevance effort — ranking tuning, boosting, personalisation — operates only on documents that matched, so a document the analyser made unfindable is invisible to all of it. Compounding this, analysis changes require a full reindex, so teams that have not made reindexing routine cannot iterate on the thing that matters most, and relevance work drifts toward the parts they can change quickly. My priorities would therefore be to make reindexing an exercised, unremarkable operation; to move as much configuration as possible to query time where it can be iterated; and to treat the synonym list as an owned asset belonging to whoever understands the domain, since it encodes knowledge engineers do not have and users will not type.

   </details>

---

### BM25 relevance ranking

*Score documents by how unusual their matching terms are, with diminishing returns on repetition and a correction for document length.*

**Flow:** `Query terms` → `Term statistics` → `Frequency saturation` → `Length adjustment` → `Ranked documents`

> **The 30-second version**  
> Score by how rare the matching terms are, with repetition saturating and long documents penalised — a strong baseline that ranks text and nothing else.

**The problem**

An inverted index tells you which documents match. It does not tell you which are best. A query for “distributed systems” might match fifty thousand documents, and returning them in arbitrary order — or by insertion order — is indistinguishable from returning nothing useful.

The obvious scoring rule, counting term occurrences, fails immediately. A document containing “distributed” forty times is not forty times more relevant than one containing it twice; it is probably spam or a glossary. And a long document contains more of every word by construction, so raw counts favour length rather than relevance.

> **BM25 is three corrections to word counting**  
> **Rare terms matter more** — matching “Kubernetes” says more than matching “the”. **Repetition saturates** — the tenth occurrence adds far less than the second. **Length is normalised** — a term appearing three times in a tweet means more than three times in a book. Each correction fixes a specific way that naive counting misranks, and together they remain competitive with far more complex approaches.

**Mental model**

Each query term contributes a score to each matching document. The contribution is large when the term is rare in the corpus, grows with occurrences but with diminishing returns, and is discounted if the document is longer than average. Document scores are the sum across query terms.

1. **Term frequency** — How often the term appears in this document. More is better, but with saturation rather than linearly.
2. **Inverse document frequency** — How rare the term is across the corpus. A term in ten documents out of ten million is enormously more informative than one in nine million.
3. **Length normalisation** — Documents longer than average are penalised, since they contain more of everything by chance.
4. **k1** — Controls how quickly term frequency saturates. Typically around 1.2 — higher means repetition keeps counting for longer.
5. **b** — Controls how strongly length is normalised. Typically 0.75 — zero disables it entirely, one applies it fully.

> **Saturation is what separates BM25 from its predecessors**  
> Classic TF-IDF multiplies term frequency linearly, so a document repeating a term a hundred times scores a hundred times higher. BM25 applies a saturating function, so the score approaches a ceiling — the difference between two and three occurrences is meaningful, while the difference between fifty and a hundred is negligible. That single change makes it robust against keyword stuffing and against documents that are simply long.

**How it works**

**The formula, and what each part does**

```text
score(D, Q) = sum over terms t in Q of:

    IDF(t)  x  ( f(t,D) x (k1 + 1) )
               -------------------------------------------
               f(t,D) + k1 x (1 - b + b x |D|/avgdl)

  f(t,D)   term frequency in document D
  |D|      document length
  avgdl    average document length in the corpus
  k1       saturation control (~1.2)
  b        length normalisation (~0.75)

  IDF(t) = ln( (N - n(t) + 0.5) / (n(t) + 0.5) + 1 )
  N      total documents
  n(t)   documents containing t

READING IT
  the numerator grows with f, the denominator also grows
  with f -> the ratio SATURATES toward (k1 + 1)
  |D|/avgdl > 1 makes the denominator larger -> long
  documents score lower for the same term frequency
  IDF multiplies the whole thing -> rare terms dominate
```

1. **Understand that IDF does most of the work** — In a typical query, the rarest term contributes far more to the ranking than the common ones. That is why searching for “the Kubernetes operator” ranks on `Kubernetes` and `operator`, with `the` contributing almost nothing.
2. **Tune b down for short, uniform fields** — Length normalisation matters when documents vary greatly in length. For a title field where everything is a few words, penalising the slightly longer ones is noise — setting b near zero is often better.
3. **Tune k1 for repetition semantics** — In technical documentation, repeating a term genuinely signals relevance, so a higher k1 helps. In marketing copy, repetition signals nothing, so a lower one is better.
4. **Score fields separately and combine with boosts** — A match in the title is worth more than one in the body. Running BM25 per field and weighting the results is how relevance is actually shaped in practice.
5. **Remember IDF is corpus-dependent and shard-dependent** — In a distributed index, each shard computes IDF from its own documents unless global statistics are used, which can make identical documents rank differently depending on placement.
6. **Treat BM25 as the baseline, not the ceiling** — It ranks on lexical overlap only. It knows nothing about popularity, recency, user behaviour or meaning, all of which usually matter more in a product than pure textual relevance.

**Why the three corrections matter, concretely**

```text
QUERY  "distributed consensus"

WITHOUT IDF
  a document mentioning "distributed" 20 times and
  "consensus" once scores similarly to one mentioning
  each 10 times
  -> but "consensus" is far rarer and more informative
  WITH IDF: the consensus match dominates the score

WITHOUT SATURATION
  a glossary page listing "distributed" 200 times ranks
  first for every query containing it
  WITH SATURATION: its score plateaus; a focused article
  mentioning it 5 times in context competes

WITHOUT LENGTH NORMALISATION
  a 50,000-word book contains almost every term several
  times by chance, so it matches everything
  WITH NORMALISATION: a 500-word article with the same raw
  counts scores higher, because the terms are denser

EACH CORRECTION FIXES A SPECIFIC MISRANKING, and removing
any one reintroduces a recognisable bad-search behaviour.
```

> **BM25 ranks text, not usefulness**  
> It has no notion of whether a document is recent, popular, authoritative, in stock, or from a trustworthy source. A perfectly relevant but three-year-old answer outranks a current one if its text matches slightly better. Production ranking almost always combines BM25 with signals the formula cannot see, which is why relevance engineering is mostly about what to add to it rather than about tuning it.

**Worked example**

Ranking product search results, showing why BM25 alone is insufficient and what it is combined with.

**From text relevance to product relevance**

```text
QUERY  "wireless headphones"

BM25 ALONE ranks by textual match:
  1. "Wireless Headphones Wireless Audio Headphones"  (spam-ish)
  2. discontinued 2019 model, perfect title match
  3. out-of-stock item
  4. the actual best-selling current product

WHAT BM25 CANNOT SEE
  popularity, sales, ratings
  availability and price
  recency and product lifecycle
  user's past behaviour
  business priorities

PRODUCTION SCORING
  final = w1 x BM25(title)     x 3.0
        + w2 x BM25(description)
        + w3 x log(1 + sales_last_30d)
        + w4 x rating
        + w5 x recency_decay
        - penalty if out of stock
        (and increasingly: a learned model over these
         features rather than hand-set weights)

BM25'S ROLE
  it is the RETRIEVAL and baseline relevance signal:
  it decides which 1,000 of 10 million products are
  candidates. The other signals then reorder those 1,000.
  -> cheap, broad, text-based filtering first;
     expensive, signal-rich ranking second
```

| Metric | Value | Note |
|---|---|---|
| BM25 | retrieval + baseline | millions → 1,000 |
| Business signals | reranking | **1,000 → 10** |
| Title boost | 3× | position matters |
| Stock | penalty | BM25 cannot see it |

> **BM25's real job is candidate retrieval, not final ranking**  
> In a two-stage architecture, BM25 cheaply reduces millions of documents to a few hundred candidates using only the index, and a richer model — with behavioural, commercial and semantic signals — reorders those. That division matters because the rich model is far too expensive to run over the whole corpus. Understanding BM25 as the first stage explains both why it is still ubiquitous and why tuning it rarely fixes relevance complaints.

**When to use it**

- **Default text ranking** in any search system, as the baseline that later signals build on.
- **Candidate retrieval** in a two-stage ranking architecture, where cheap and broad matters more than perfect.
- **Corpora without behavioural data**, such as a new product or a private document search, where there are no clicks to learn from.
- **Alongside field boosts**, which is how most practical relevance shaping is done.
- **As a component of hybrid retrieval**, fused with vector similarity to cover both lexical and semantic matching.

**When to avoid it**

- **Do not use it as the final ranking** for a product or content system, where popularity, recency and availability dominate user satisfaction.
- **Do not expect it to handle synonyms or meaning**; it matches tokens, so `car` and `automobile` are unrelated to it.
- **Do not apply length normalisation to uniformly short fields**, where it penalises noise.
- **Do not compare scores across queries** — BM25 scores are not calibrated and have no absolute meaning.
- **Do not ignore shard-local IDF** in a distributed index, which can make ranking depend on document placement.

**Advantages**

- **Strong baseline with no training data**, which is why it remains the default after decades.
- **Cheap to compute** from statistics already stored in the index.
- **Robust to keyword stuffing** through frequency saturation.
- **Two interpretable parameters**, so tuning is comprehensible rather than opaque.
- **Competitive with far more complex methods** on pure text relevance, and hard to beat without behavioural data.

**Disadvantages**

- **Lexical only** — no synonyms, no semantics, no understanding of meaning.
- **Blind to everything outside the text**: popularity, recency, quality, availability, authority.
- **Scores are not comparable across queries**, so thresholds are meaningless.
- **IDF depends on the corpus**, so ranking shifts as the corpus changes and differs across shards.
- **Word order and proximity are ignored** unless phrase matching is added separately.

**Trade-offs**

**Parameter effects**

| Parameter | Low value | High value | Typical |
|---|---|---|---|
| k1 (saturation) | Repetition counts for almost nothing | Closer to linear term frequency | 1.2–2.0 |
| b (length norm) | Length ignored; long documents favoured | Strong penalty on long documents | 0.75 |
| Field boost | Field contributes little | Field dominates ranking | Title 3–10× body |

The most useful practical adjustment is not k1 or b but field boosting: a title match is worth many times a body match, and getting that ratio right typically improves perceived relevance far more than tuning the formula's internals.

**How it fails**

**Ranking failures and their causes**

| Symptom | Cause | Fix |
|---|---|---|
| Keyword-stuffed pages rank first | Saturation disabled or k1 too high | Default k1; add quality signals |
| Long documents dominate | Length normalisation too low | Raise b toward 0.75 |
| Short titles unfairly penalised | Length normalisation applied to a uniform field | Lower b near zero for that field |
| Stale results outrank current ones | No recency signal | Add recency decay outside BM25 |
| Identical documents rank differently | Shard-local IDF statistics | Use global term statistics, or accept the variance |
| Relevance does not improve with tuning | Missing signals rather than bad scoring | Add behavioural, commercial and quality features |
| Synonyms do not match | BM25 is lexical | Synonym expansion in analysis; or hybrid with vector retrieval |

**Limits**

> **Working values**
>
> - **k1 ≈ 1.2**, controlling how fast term frequency saturates.
> - **b ≈ 0.75**, controlling length normalisation; near 0 for uniform short fields.
> - **IDF dominates** in most queries — the rarest term drives the ranking.
> - **Scores are relative within a query only**; there is no meaningful absolute threshold.
> - **Two-stage ranking** is the standard architecture: BM25 retrieves hundreds of candidates, a richer model reorders them.

**Alternatives**

| Approach | Adds over BM25 | Cost |
|---|---|---|
| BM25 | Baseline lexical relevance | No semantics or behaviour |
| TF-IDF | Simpler, older | No saturation; worse |
| Field-boosted BM25 | Structural importance | Weights to tune |
| Learning to rank | Behavioural and commercial signals | Training data; a model to maintain |
| Vector / embedding retrieval | Semantic similarity | No exact matching; opaque |
| Hybrid (BM25 + vectors) | Both lexical and semantic | Fusion tuning; two indexes |

Learning to rank is the natural successor once click data exists: BM25 becomes one feature among many, alongside popularity, recency and personalisation, with weights learned rather than guessed. Without behavioural data there is nothing to learn from, which is why BM25 remains the right starting point.

**In real systems**

- **Elasticsearch and Lucene** made BM25 the default scoring function, replacing classic TF-IDF, because saturation and length normalisation measurably improve results.
- **Two-stage retrieval and reranking** is the standard architecture in web and product search, with a cheap lexical first stage feeding an expensive model.
- **Hybrid retrieval** fusing BM25 with dense vector search has become common, because each covers the other's blind spot — exact terms versus meaning.
- **BM25 remains a strong baseline** in information retrieval benchmarks, frequently competitive with neural methods on pure text relevance.
- **E-commerce ranking** treats BM25 as one feature among many, with availability, margin and popularity often outweighing textual match entirely.

**Common mistakes**

- **Tuning k1 and b** to fix problems caused by analysis or missing signals.
- **Using BM25 as the final ranking** in a product context where availability and popularity matter more.
- **Comparing scores across queries**, which have no common scale.
- **Applying length normalisation to uniform short fields**, penalising noise.
- **Ignoring shard-local IDF**, so ranking depends on document placement.
- **Expecting semantic matching** from a lexical scorer.
- **Changing ranking without measurement**, making relevance a matter of opinion.

**The staff-level view**

Relevance complaints are almost never fixed by tuning the scoring function, which is where teams instinctively start.

- **Diagnose before tuning.** Ask whether the right documents are being *retrieved* at all — an analysis problem — before assuming the problem is *ranking*. Tuning k1 cannot help a document the analyser made unmatchable.
- **Add signals rather than adjusting parameters.** Recency, popularity, quality and availability usually matter more to users than textual relevance, and BM25 is structurally blind to all of them.
- **Adopt two-stage ranking early.** Cheap retrieval plus expensive reranking is the architecture that allows rich signals at all, and retrofitting it is harder than starting with it.
- **Use field boosts as the primary tuning lever**, since a title-versus-body weighting typically improves perceived relevance more than the formula's internals.
- **Measure relevance rather than debating it.** Without click-through, conversion or judged relevance data, every ranking change is an opinion, and teams argue indefinitely.

**Go deeper**

BM25 corrects three failures of naive word counting. Rare terms contribute far more than common ones, so matching an unusual word dominates the score. Term repetition saturates, so the tenth occurrence adds much less than the second, which makes it robust against keyword stuffing and glossary pages. And document length is normalised, so the same raw counts in a short document score higher than in a long one, since long documents contain more of everything by chance.

Two parameters control this: k1 sets how fast repetition saturates, and b sets how strongly length is penalised — typically 1.2 and 0.75, with b near zero for uniformly short fields like titles where length variation is noise. In practice, field boosting matters more than either: a title match weighted several times a body match improves perceived relevance more than tuning the formula's internals.

Its limitation is structural: it ranks textual overlap and is blind to recency, popularity, quality, availability and meaning — all of which usually matter more to users. So production systems use it for retrieval and baseline relevance, cheaply reducing millions of documents to a few hundred candidates, then rerank those with a richer model. That also explains why relevance complaints are rarely fixed by tuning BM25 and are usually caused by analysis or missing signals.

BM25 has remained the default text ranking function for decades because it makes three specific corrections to word counting, each of which fixes a recognisable failure mode.

**Inverse document frequency does most of the work.** A term appearing in ten documents out of ten million is enormously more informative than one appearing in nine million, so IDF multiplies each term's contribution by its rarity. In a typical multi-word query, the rarest term dominates the ranking — which is why searching for “the Kubernetes operator” effectively ranks on the two distinctive words and why stop words contribute almost nothing even when retained.

**Saturation is the key improvement over TF-IDF.** Classic scoring multiplies term frequency linearly, so a page repeating a term two hundred times outranks everything. BM25 applies a saturating function where the score approaches a ceiling: the difference between one and three occurrences is substantial, between fifty and a hundred negligible. That makes ranking robust against keyword stuffing and against documents, like glossaries and indexes, whose repetition signals nothing about relevance. The parameter k1 controls the rate, and its right value is domain-dependent — technical documentation where repetition genuinely indicates focus wants a higher value than marketing copy where it does not.

**Length normalisation prevents long documents dominating.** A fifty-thousand-word book contains most terms several times by chance, so raw counts would make it match everything. Dividing by length relative to the corpus average corrects this, controlled by b. The important nuance is that this correction is only meaningful where lengths vary: applied to a title field where every value is a few words, it penalises documents for noise, so b near zero is usually better there. This is a common misconfiguration precisely because the global default is applied to every field.

**What it cannot see is why it is rarely the final ranking.** BM25 scores lexical overlap. It has no notion of recency, popularity, ratings, availability, authority or meaning, and in most products those matter more to user satisfaction than which document has marginally better term overlap. The standard architecture is therefore two-stage: BM25 retrieves a few hundred candidates from millions cheaply, using only statistics already in the index, and a richer model — hand-weighted or learned — reorders those candidates using signals far too expensive to compute corpus-wide. Recognising BM25 as the first stage explains both its durability and why tuning it seldom resolves relevance complaints.

**Scores are relative, not absolute.** A BM25 score depends on the query's terms, their IDF, the number of terms and the document lengths involved, so scores from different queries are not comparable and thresholds based on them behave unpredictably. Similarly, IDF is computed from corpus statistics, which in a sharded index are often shard-local — meaning two identical documents can rank differently depending on which shard holds them, unless global statistics are configured.

**Moving beyond it requires data.** Learning to rank makes BM25 one feature among many with weights learned from clicks and conversions, which is strictly better once behavioural data exists and impossible before. Hybrid retrieval fusing BM25 with dense vector search addresses the orthogonal gap: lexical matching cannot find a document that expresses the same idea in different words, while vector similarity cannot reliably match an exact model number. Each covers the other's blind spot, which is why wholesale replacement of lexical retrieval tends to produce search that feels approximately right and is precisely wrong for the queries users are most confident about.

**Prove it — interview questions**

1. **[Basic] What are the three ideas in BM25?**

   <details><summary>Model answer</summary>

   Rare terms matter more than common ones, so matching an unusual word contributes far more to the score than matching a frequent one. Term repetition saturates, so the tenth occurrence adds much less than the second, which prevents keyword stuffing from dominating. And document length is normalised, so a term appearing three times in a short document counts for more than three times in a long one, since long documents contain more of everything by chance. Each correction fixes a specific way that naive word counting misranks.

   </details>

2. **[Basic] Why does term frequency saturate?**

   <details><summary>Model answer</summary>

   Because relevance does not grow linearly with repetition. A document mentioning a term twice is meaningfully more relevant than one mentioning it once; a document mentioning it a hundred times is not meaningfully more relevant than one mentioning it fifty times — it is more likely to be a glossary or spam. BM25's saturating function lets early occurrences count substantially while later ones approach a ceiling, which is the main improvement over classic TF-IDF and makes the ranking robust against keyword stuffing.

   </details>

3. **[Senior] Why is BM25 rarely the final ranking in production?**

   <details><summary>Model answer</summary>

   Because it scores textual match and nothing else. It cannot see whether a product is in stock, whether an article is three years out of date, whether a result is popular or highly rated, or whether the source is authoritative — and in most products those factors matter more to user satisfaction than which document has slightly better term overlap. The standard architecture uses BM25 for retrieval and baseline relevance, reducing millions of documents to a few hundred candidates cheaply, and then reorders those candidates with a model incorporating behavioural, commercial and quality signals that would be far too expensive to compute over the whole corpus.

   </details>

4. **[Senior] Why can't you compare BM25 scores across queries?**

   <details><summary>Model answer</summary>

   Because the scores have no absolute scale. They depend on the IDF of the query terms, the number of terms, the corpus statistics and the document lengths involved, so a score of twelve for one query and eight for another says nothing about which result is better. That means score thresholds — “only show results above X” — are meaningless and will behave unpredictably as queries and the corpus change. Relative ordering within a single query result set is the only thing the score is valid for.

   </details>

5. **[Staff] A team reports poor search relevance. How do you approach it?**

   <details><summary>Model answer</summary>

   By establishing which stage is failing before touching any parameters. The first question is whether the right documents are being retrieved at all, because if the analyser made a document unmatchable — a model number split by tokenisation, a stemming mismatch — no amount of ranking work will surface it, and this is by far the most common cause. Second, whether the relevant documents are retrieved but ranked below irrelevant ones, in which case field boosts usually matter more than the formula's parameters: a title match should outweigh a body match substantially. Third, whether ranking is textually correct but unsatisfying, which means missing signals — recency, popularity, availability — and the answer is adding features rather than tuning. And underneath all of it, I would want measurement: click-through, conversion or judged relevance, because without it every change is an opinion and the team will argue indefinitely without converging.

   </details>

6. **[Principal] When does it make sense to move beyond BM25?**

   <details><summary>Model answer</summary>

   When behavioural data exists to learn from, and when the corpus has meaningful signals beyond text. Learning to rank turns BM25 into one feature alongside popularity, recency, personalisation and commercial factors, with weights learned from clicks and conversions rather than guessed — which is strictly better once you have the data, and impossible before. Separately, hybrid retrieval fusing BM25 with dense vector search addresses a different gap: BM25 cannot match meaning, so a user describing a problem in their own words finds nothing, while vectors cannot match exact identifiers, so a user typing a model number finds the wrong thing. Fusing both covers each other's blind spot. What I would resist is replacing BM25 with a neural retriever wholesale, because exact lexical matching remains genuinely important and losing it produces a search that feels vaguely right and is precisely wrong for the queries users are most confident about.

   </details>

---

### Autocomplete and prefix search

*Suggest completions as the user types, within tens of milliseconds, ranked by what they are likely to want rather than by what matches.*

**Flow:** `Typed prefix` → `Prefix index` → `Context filter` → `Popularity rank` → `Suggestions`

> **The 30-second version**  
> Predict the query from a fragment in tens of milliseconds, ranked by likelihood rather than match quality — from a small curated corpus of queries, not the document corpus.

**The problem**

A user types three characters and expects suggestions before they type the fourth. That is a budget of perhaps fifty milliseconds end to end including the network, for a query that must match against millions of possible completions — and it happens on every keystroke, so the query rate is several times the search rate.

Standard search machinery is the wrong shape for this. A full-text query with analysis, scoring and ranking takes tens of milliseconds server-side and matches whole terms, not prefixes. Typing “dis” should suggest “distributed systems”, but a term index has no entry for “dis”.

> **Autocomplete is a different problem from search**  
> Search retrieves documents matching a completed query. Autocomplete predicts the query itself, from a fragment, under a far tighter latency budget and a much higher request rate. It is ranked by what users are likely to want — popularity, personal history, context — rather than by textual relevance to the fragment. Treating it as a search query with a wildcard is the usual mistake.

**Mental model**

Build a structure that maps prefixes to candidate completions, keep it small enough to stay in memory, and rank the candidates by likelihood rather than by match quality.

1. **Edge n-grams** — Index each prefix as a separate token: `dis`, `dist`, `distr`. Simple, works with an ordinary inverted index, multiplies index size.
2. **Trie / FST** — A tree keyed by character, where a prefix walk yields all completions beneath it. Compact and extremely fast, and the basis of dedicated suggester implementations.
3. **Completion suggester** — A purpose-built in-memory structure that trades flexibility for latency, typically returning in single-digit milliseconds.
4. **Ranking signals** — Query frequency, click-through, recency, personal history and context. This is what determines quality, not the matching.
5. **Context filtering** — Restricting suggestions by category, location or user segment, so the same prefix yields different completions in different contexts.

> **The request rate is the hidden constraint**  
> Autocomplete fires on every keystroke, so a user typing an eight-character query generates six to eight requests where search generates one. Traffic is therefore roughly an order of magnitude higher than search traffic, with a far tighter latency budget — which means the design must be cheap per request, not merely fast. Client-side debouncing and result caching are not optimisations here but requirements.

**How it works**

**Prefix structures compared**

```text
EDGE N-GRAMS (in a normal inverted index)
  "distributed" indexes as:
    d, di, dis, dist, distr, distri, ... distributed
  query "dis" is an ordinary term lookup
  + works with existing search infrastructure
  + composes with filters and normal scoring
  - index size grows by roughly the average term length
  - memory cost is significant

TRIE / FST (dedicated structure)
  characters form a tree; a prefix walk finds the subtree
  + very compact when shared prefixes are common
  + microsecond lookups, entirely in memory
  - a separate structure to build and keep in sync
  - filtering and custom scoring are limited

COMPLETION SUGGESTER (purpose-built)
  FST with weights baked in at index time
  + fastest option; single-digit milliseconds
  - weights are fixed at index time, so changing ranking
    requires reindexing
  - limited filtering

PRACTICAL CHOICE
  edge n-grams when suggestions need filtering, scoring
  or combination with other query logic
  FST-based suggesters when raw latency dominates and
  ranking is mostly static popularity
```

1. **Suggest queries, not documents** — Users want to complete their query, not see search results. Indexing a curated set of popular queries is smaller, faster and more relevant than prefix-matching every document title.
2. **Rank by popularity, not by match quality** — Every candidate matches the prefix equally well. What distinguishes them is how likely the user is to want each one, which comes from query logs, click-through and personal history.
3. **Debounce and cache on the client** — Firing on every keystroke is wasteful; a short debounce plus caching prefix results locally removes most of the traffic before it reaches the server.
4. **Handle typos deliberately** — A user typing “distrubuted” gets nothing from a strict prefix match. Fuzzy prefix matching or a spell-corrected fallback is needed, but it is expensive — apply it only when the strict match returns too few results.
5. **Use context to filter, not just to rank** — The same prefix should suggest differently on a product page than on a support page, and a location-aware query should prefer nearby results.
6. **Keep the suggestion corpus curated** — Auto-generating suggestions from all user queries surfaces typos, offensive text and zero-result queries. Filter by minimum frequency and by result count.

**Why suggesting queries beats suggesting documents**

```text
SUGGESTING DOCUMENTS
  prefix "sony wh" matches every product whose title
  starts with or contains those tokens
  -> 40 nearly identical product variants
  -> the user sees a wall of near-duplicates
  -> the index must cover millions of titles

SUGGESTING QUERIES
  a curated list of popular searches:
    "sony wh-1000xm5"      (12,000 searches/month)
    "sony wh-1000xm4"      (3,400)
    "sony wh headphones"   (900)
  -> 3 distinct, useful suggestions
  -> the index holds ~100k queries, not 10M documents
  -> it is 100x smaller and fits comfortably in memory

BUILDING THE QUERY CORPUS
  take search logs from the last 90 days
  keep queries with >= N occurrences
  DROP queries that returned zero results
    -> otherwise you suggest searches that fail
  drop offensive and personally identifying text
  recompute weekly so new products appear

THE ZERO-RESULT FILTER IS THE ONE PEOPLE FORGET,
and suggesting a query that returns nothing is worse
than suggesting nothing at all.
```

> **Auto-generated suggestions need moderation**  
> A suggestion corpus built from raw query logs will contain misspellings, offensive phrases, competitor names, personal information typed into the search box, and queries that return no results. Each of these is embarrassing when surfaced as a suggestion by your product. Filtering by minimum frequency, by result count, and against a blocklist is not optional polish — it is a precondition for shipping.

**Worked example**

An e-commerce autocomplete, designed around the latency and traffic constraints.

**Architecture and budget**

```text
CONSTRAINTS
  p99 latency budget: 50ms end to end
  traffic: ~8x search traffic (one request per keystroke)
  corpus: 10M products, but suggestions come from queries

DESIGN
  suggestion corpus: top 200k queries from 90 days of logs
    filtered: >= 5 occurrences, > 0 results, moderated
    weight: log(frequency) x recency_decay
  structure: FST in memory, ~50 MB
  server latency: 2-5ms

  CLIENT
    debounce 150ms  -> 8 keystrokes become ~3 requests
    cache by prefix -> typing then deleting hits the cache
    abort in-flight requests when a new keystroke arrives

  CONTEXT
    if the user is in a category, filter suggestions to it
    if signed in, blend in their recent searches at the top

  FALLBACK
    if the strict prefix returns < 3 suggestions, run a
    fuzzy prefix match (edit distance 1) as a second pass
    -> handles typos without paying for fuzziness always

RESULTING TRAFFIC
  without debounce: 8 requests per search
  with debounce + cache: ~2 requests per search
  -> a 4x reduction before any server work
```

| Metric | Value | Note |
|---|---|---|
| Corpus | 200k queries | not 10M products |
| Structure | 50 MB FST | **in memory** |
| Server | 2–5 ms | within budget |
| Client | debounce + cache | 4× fewer requests |

> **The client-side work matters as much as the server design**  
> Debouncing and caching reduce request volume several-fold before anything reaches the server, which is the cheapest capacity you will ever find. Aborting in-flight requests when a new keystroke arrives additionally prevents out-of-order responses overwriting newer suggestions with older ones — a subtle bug that makes autocomplete feel erratic and is invisible in server metrics.

**When to use it**

- **Search boxes** in any product with a non-trivial corpus, where typing the full query is friction.
- **Reducing zero-result searches**, by steering users toward queries that are known to work.
- **Guiding discovery**, since suggestions shape what users search for at all.
- **Address, place and name entry**, where prefix completion prevents typos in structured data.
- **Command palettes and internal tools**, where the corpus is small and latency expectations are extreme.

**When to avoid it**

- **Do not implement it as a wildcard search query**, which is orders of magnitude too slow and matches the wrong things.
- **Do not suggest documents when users want queries**, which produces walls of near-duplicates.
- **Do not suggest queries that return no results**, which is worse than suggesting nothing.
- **Do not apply fuzzy matching to every request**, which is expensive and unnecessary when the strict prefix succeeds.
- **Do not ship an unmoderated corpus** built from raw query logs.

**Advantages**

- **Dramatically reduces typing and typos**, which directly reduces zero-result searches.
- **Steers users toward queries that work**, improving conversion and reducing failed sessions.
- **Very fast when built on the right structure**, with single-digit millisecond server latency.
- **Small corpus** when suggesting queries rather than documents, so it fits in memory easily.
- **Shapes demand**, since suggestions influence what users search for — a commercial lever as well as a usability one.

**Disadvantages**

- **High request volume**, several times search traffic, on a tighter latency budget.
- **Ranking weights are often fixed at index time** in the fastest structures, so changes need a rebuild.
- **Requires moderation**, since auto-generated suggestions surface typos and offensive content.
- **Typo tolerance is expensive**, and applying it universally defeats the latency budget.
- **A separate index to build and keep fresh**, which is another pipeline to operate.
- **Cold-start problem**: without query logs there is no popularity signal to rank by.

**Trade-offs**

**Implementation choices**

| Approach | Latency | Flexibility | Index cost |
|---|---|---|---|
| Edge n-grams in the search index | 10–30 ms | Full filtering and scoring | Large — multiplies term count |
| Dedicated FST suggester | 1–5 ms | Limited filtering; static weights | Small, in memory |
| Trie in application memory | <1 ms | Full control | Build and sync it yourself |
| Wildcard search query | 100 ms+ | Full | None — but unusable |
| External suggestion service | Varies | Full | Another system |

> **Framing the design**  
> “Suggest queries rather than documents, from a curated corpus of the top couple of hundred thousand searches with zero-result queries filtered out, held in an in-memory FST for single-digit millisecond lookups. Then client-side debouncing and caching, which cuts request volume four-fold before anything reaches the server — and fuzzy matching only as a second pass when the strict prefix returns too few results.”

**How it fails**

**Autocomplete failures**

| Symptom | Cause | Fix |
|---|---|---|
| Suggestions arrive after the user finished typing | Latency budget exceeded; wrong structure | In-memory prefix structure; debounce; reduce corpus |
| Suggestions flicker or show stale text | Out-of-order responses overwriting newer ones | Abort in-flight requests; discard responses for stale prefixes |
| Suggested query returns no results | Corpus not filtered by result count | Exclude zero-result queries when building the corpus |
| Offensive or personal text suggested | Unmoderated corpus from raw logs | Frequency threshold, blocklist, manual review |
| Wall of near-identical suggestions | Suggesting documents rather than queries | Suggest curated queries; deduplicate |
| Typos return nothing | Strict prefix matching only | Fuzzy fallback when strict results are insufficient |
| Autocomplete traffic overwhelms search infrastructure | No debouncing; shared index with search | Client debounce and cache; separate suggestion service |

**Limits**

> **Design parameters**
>
> - **Latency budget**: 50 ms end to end, implying single-digit milliseconds server-side.
> - **Traffic multiplier**: roughly 5–10× search volume without debouncing; 2–3× with it.
> - **Debounce**: around 100–200 ms, which typically halves or better the request count.
> - **Corpus size**: hundreds of thousands of curated queries, not millions of documents.
> - **Fuzzy fallback** only when strict matching returns fewer than a handful of suggestions.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Query suggestion from logs | Established products with traffic | Cold start; needs moderation |
| Document title prefix | New products with no query data | Near-duplicate suggestions |
| Curated editorial suggestions | Small corpora; commercial control | Manual maintenance |
| Personalised from history | Signed-in users | Privacy; cold start for new users |
| Semantic suggestion (embeddings) | Intent-based completion | Latency; less predictable |
| No autocomplete | Tiny corpora | More typing; more zero-result searches |

The cold-start problem is real and often overlooked: a new product has no query logs, so popularity ranking is impossible. The usual bridge is document titles and curated editorial suggestions until sufficient query volume accumulates, then a switch to log-derived suggestions.

**In real systems**

- **Search engines** suggest queries rather than results, ranked by frequency and personalised by history, which is the canonical form of this design.
- **Elasticsearch's completion suggester** uses an in-memory FST with weights fixed at index time, trading ranking flexibility for single-digit millisecond latency.
- **E-commerce autocomplete** blends popular queries, category context and personal history, and is treated as a commercial surface because suggestions shape demand.
- **Command palettes** in developer tools use in-process tries over small corpora, achieving sub-millisecond response with no network at all.
- **Address autocomplete services** exist as dedicated products precisely because prefix matching over structured geographic data is a specialised problem.

**Common mistakes**

- **Implementing it as a wildcard or regex search query**, which cannot meet the latency budget.
- **Suggesting documents**, producing near-duplicate walls instead of distinct queries.
- **Including zero-result queries** in the suggestion corpus.
- **No moderation**, surfacing typos, offensive text and personal data.
- **No client debouncing**, multiplying traffic unnecessarily.
- **No request cancellation**, so out-of-order responses make suggestions flicker.
- **Always applying fuzzy matching**, spending the latency budget on a rare case.

**The staff-level view**

Autocomplete is usually built as a feature of search and should be designed as a separate system with different constraints.

- **Separate it from search infrastructure.** The traffic profile is an order of magnitude different and the latency budget is far tighter; sharing an index couples their failure modes and capacity.
- **Suggest queries, not documents.** The corpus is a hundred times smaller, the suggestions are more useful, and the whole thing fits in memory.
- **Filter zero-result queries from the corpus**, since suggesting a search that fails is worse than suggesting nothing and actively damages trust.
- **Invest in the client side**, where debouncing, caching and request cancellation reduce load several-fold and fix the flicker bug that server metrics cannot see.
- **Treat the suggestion corpus as a moderated, owned asset**, because it is a published surface of your product assembled from whatever users typed into a box.

**Go deeper**

Autocomplete predicts the query rather than retrieving documents, under a fifty-millisecond end-to-end budget and at several times search traffic since it fires on every keystroke. That rules out ordinary search machinery: a wildcard query is orders of magnitude too slow, and term indexes have no entry for a partial word. The structures that work are edge n-grams in an inverted index, or an in-memory trie or FST giving single-digit millisecond lookups.

Suggest queries, not documents. A curated corpus of the most popular searches is perhaps a hundred times smaller than the document corpus, fits comfortably in memory, and produces a handful of distinct suggestions rather than a wall of near-identical product variants. Build it from search logs filtered by minimum frequency, moderated for offensive and personal text, and — critically — with zero-result queries excluded, since suggesting a search that fails is worse than suggesting nothing.

Ranking is by likelihood rather than match quality, because every candidate matches the prefix equally: popularity, recency, personal history and context. And the client-side work matters as much as the server design — debouncing and caching cut request volume several-fold before anything reaches the server, while cancelling in-flight requests prevents slower earlier responses from overwriting newer suggestions, a flicker bug invisible in server metrics.

Autocomplete is routinely built as a feature of search and is better understood as a separate system, because its constraints differ in kind rather than in degree.

**The constraints.** A fifty-millisecond end-to-end budget including network means single-digit milliseconds server-side. Firing on every keystroke makes traffic five to ten times search volume before any client-side mitigation. And the ranking question is different: every candidate matches the prefix equally well, so ordering is by how likely the user is to want each one — popularity from query logs, personal history, contextual signals — rather than by textual relevance to a fragment.

**Structures.** Edge n-grams index every prefix as a separate token so an ordinary inverted index can serve prefix lookups, which composes well with filters and scoring but multiplies index size by roughly the average term length. A trie or finite state transducer keyed by character allows a prefix walk to yield all completions beneath it, is extremely compact where prefixes are shared, and gives microsecond lookups entirely in memory — at the cost of being a separate structure with limited filtering and, in the fastest implementations, weights fixed at index time so changing ranking requires a rebuild.

**Suggest queries rather than documents.** This is the decision that makes everything else work. Prefix-matching document titles surfaces forty near-identical variants and requires indexing millions of records; a curated list of the most popular searches yields a few distinct suggestions from a corpus perhaps a hundred times smaller, which is what allows it to sit in memory. Building that corpus means taking recent query logs, applying a minimum frequency threshold to exclude one-off typos, moderating against offensive and personally identifying text, and excluding queries that returned no results — because suggesting a search that fails is worse than offering nothing and is the filter teams most often omit.

**The client is half the system.** Debouncing by a hundred to two hundred milliseconds converts eight keystrokes into two or three requests, a several-fold traffic reduction achieved before any server capacity is consumed. Caching by prefix means backtracking hits local state. And cancelling in-flight requests when a new keystroke arrives prevents an earlier slower response from replacing a newer one — a bug that makes suggestions flicker and display stale text, which is entirely invisible in server-side metrics and immediately obvious to users.

**Typo tolerance is a fallback, not a default.** Fuzzy prefix matching is expensive and the budget is tight, so the strict match runs first and a second pass with edit distance one runs only when the strict result set is too small. Fuzzy results should rank below exact ones, and very short prefixes should be excluded from fuzziness entirely, since an edit distance of one matches almost everything at two or three characters and produces noise.

**It is a product surface, not just a feature.** Suggestions shape what users search for, which makes the corpus a commercial lever and an editorial responsibility — it is assembled from whatever people typed into a text box, and publishing that unfiltered carries reputational risk unrelated to engineering. Combined with the divergent traffic profile and latency budget, that argues for treating autocomplete as its own system with its own corpus pipeline, ownership, moderation process and service-level objective, rather than as an endpoint on the search cluster whose failures and capacity are entangled with it.

**Prove it — interview questions**

1. **[Basic] Why is autocomplete a different problem from search?**

   <details><summary>Model answer</summary>

   Because it predicts the query rather than retrieving documents for one, under a much tighter latency budget and a much higher request rate. It fires on every keystroke, so traffic is several times search volume, and it must respond within tens of milliseconds including the network. Ranking is also different: every candidate matches the prefix equally, so ordering is by likelihood — popularity, recency, personal history — rather than by textual relevance.

   </details>

2. **[Basic] Why suggest queries rather than documents?**

   <details><summary>Model answer</summary>

   Because users are trying to complete their query, and because the corpus is dramatically smaller. Prefix-matching document titles produces walls of near-identical variants — forty similar products all starting with the same words — while a curated list of popular queries gives a handful of distinct, useful suggestions. It is also perhaps a hundred times smaller, so it fits comfortably in memory, which is what makes single-digit millisecond latency achievable.

   </details>

3. **[Senior] How do you build the suggestion corpus?**

   <details><summary>Model answer</summary>

   From search logs, with filtering. Take queries from a recent window, keep those above a minimum frequency so one-off typos are excluded, and critically drop any query that returned zero results — suggesting a search that fails is worse than suggesting nothing and directly damages trust. Then apply moderation against offensive terms and personally identifying text, since the corpus is assembled from whatever users typed into a box and will be published as a surface of your product. Recompute weekly so new products and trends appear, with weights combining frequency and recency decay.

   </details>

4. **[Senior] What client-side work matters for autocomplete?**

   <details><summary>Model answer</summary>

   Three things that collectively matter as much as the server design. Debouncing by a hundred or two hundred milliseconds turns eight keystrokes into two or three requests, which is a several-fold traffic reduction before anything reaches the server. Caching results by prefix means typing and then deleting characters hits the cache rather than the network. And cancelling in-flight requests when a new keystroke arrives prevents a slower earlier response from overwriting a newer one — a bug that makes suggestions flicker and show stale text, which is invisible in server metrics and very visible to users.

   </details>

5. **[Staff] How would you handle typos in autocomplete?**

   <details><summary>Model answer</summary>

   As a fallback rather than as the default, because fuzzy matching is expensive and the latency budget is tight. The strict prefix match runs first; only if it returns fewer than a handful of suggestions does a second pass with edit distance one run. That way the common case — correctly typed prefixes — pays nothing, while typos still produce something useful. I would also weight the fuzzy results below exact ones so that a correct prefix match always ranks first, and be careful with very short prefixes, where edit distance one matches almost everything and produces noise rather than help.

   </details>

6. **[Principal] Why should autocomplete be treated as a separate system from search?**

   <details><summary>Model answer</summary>

   Because its constraints differ in kind rather than degree. Traffic is roughly an order of magnitude higher, the latency budget is several times tighter, the corpus is a hundred times smaller, and the ranking signals are entirely different — popularity and personal history rather than textual relevance. Sharing an index couples their capacity and failure modes, so an autocomplete traffic spike degrades search and a search index rebuild degrades autocomplete, neither of which is acceptable. There is also a product dimension: suggestions shape what users search for, which makes the corpus a commercial surface with editorial and moderation requirements that search results do not have — it is assembled from whatever people typed into a box, and publishing that unfiltered is a reputational risk rather than a technical one. Treating it as its own system with its own ownership, corpus pipeline and latency SLO reflects what it actually is.

   </details>

---

### Faceted search

*Show counts of matching documents per attribute value alongside results, so users can narrow a large result set by properties they can see.*

**Flow:** `Text matches` → `Active filters` → `Facet aggregation` → `Counts` → `Refined results`

> **The 30-second version**  
> Show how many results each attribute value would yield, given the current query and other filters — which requires aggregating over the matching set and demands bounded cardinality.

**The problem**

A search for “laptop” returns four thousand results. The user cannot evaluate four thousand items, and they do not know what refinement is available — is there a filter for RAM? For screen size? How many results would remain if they chose sixteen gigabytes?

Facets answer both questions at once. They enumerate the available attributes and, crucially, show the count of matching documents for each value. That converts an overwhelming result set into a navigable space, and it prevents the worst refinement outcome: clicking a filter and landing on zero results.

> **The counts are the feature, not the filters**  
> Filters alone are just query parameters. What makes faceting valuable is showing how many results each value would yield *given the current query and other filters*. That tells the user which refinements are worth making and which lead nowhere — and it means every facet count must be computed against the live result set, not from static metadata.

**Mental model**

A facet is an aggregation over the documents matching the current query. Computing it means grouping the result set by an attribute and counting each group — which is why faceting is an aggregation problem rather than a search problem.

1. **Facet field** — An attribute with a bounded set of values: brand, category, colour, price range. Must be indexed for aggregation, not just for matching.
2. **Facet count** — The number of documents in the current result set having each value. Recomputed for every query and every filter change.
3. **Filter** — A constraint that narrows the result set. Applied as a set intersection, cheaply, using the same posting-list machinery.
4. **Multi-select behaviour** — Whether selecting two values within one facet means AND or OR, and how that affects the other facets' counts.
5. **Cardinality** — How many distinct values a field has. This is what determines whether faceting on it is cheap or catastrophic.

> **High-cardinality facets are the failure mode**  
> Faceting on brand with two hundred values is cheap. Faceting on a field with a million distinct values — a product identifier, a free-text tag, a user id — means aggregating a million buckets per query, which consumes enormous memory and time. Facet fields must have bounded, low cardinality by design, and the ones that cause outages are usually added without anyone checking.

**How it works**

**Multi-select semantics: the subtle part**

```text
QUERY "laptop", user has selected Brand = Dell

NAIVE: apply all filters to all facet counts
  Brand facet now shows:  Dell (340)
                          Apple (0)
                          Lenovo (0)
  -> useless. The user cannot see what switching would give.

CORRECT: exclude a facet's own filter from its own counts
  Brand facet shows:      Dell (340)     <- selected
                          Apple (280)
                          Lenovo (410)
  -> the user can see the effect of switching brands
  RAM facet shows counts WITH the Dell filter applied:
                          8GB (120)
                          16GB (180)
                          32GB (40)
  -> the user sees what is available within Dell

THE RULE
  each facet's counts are computed against the result set
  filtered by all OTHER active filters, but not its own.
  Within a facet, multiple selections are usually OR;
  across facets, AND.
```

1. **Exclude a facet's own filter when computing its counts** — Otherwise the unselected values all show zero and the user cannot see what switching would give — the single most common faceting bug.
2. **Use OR within a facet and AND across facets** — Selecting Dell and Lenovo should show both brands; selecting Dell and 16GB should show only Dell machines with 16GB. This matches user expectation and almost nothing else does.
3. **Keep facet fields low cardinality** — Bounded value sets only. A field with unbounded values should be a filter or a search field, never a facet.
4. **Bucket continuous values** — Price and date are continuous, so facet them as ranges — under £500, £500–£1000 — rather than as distinct values.
5. **Cap the values returned per facet** — Show the top ten by count with a “show more” option, rather than returning eight hundred brands the user will never scroll through.
6. **Consider precomputation for stable facets** — Where the corpus and facets change rarely, counts for common query-plus-filter combinations can be cached, though the combinatorial space limits how far this goes.

**Why faceting is expensive, and what bounds it**

```text
A FACET COUNT is an aggregation over the matching set

QUERY matches 4,000 documents
facet on brand (200 distinct values)
  -> read the brand value of 4,000 documents
  -> increment 200 counters
  -> cheap

QUERY matches 2,000,000 documents
facet on brand
  -> read 2,000,000 values
  -> still only 200 counters, but 2M reads
  -> expensive; scales with RESULT SET SIZE

FACET ON A HIGH-CARDINALITY FIELD (1M values)
  -> 1M counter buckets held in memory PER QUERY
  -> memory explodes; queries time out
  -> this is how faceting takes a cluster down

WHAT MAKES IT AFFORDABLE
  column-oriented per-field storage (doc values), so
    reading one field for many documents is sequential
  bounded cardinality
  capping returned values
  approximate counting for very large result sets
```

> **Facet counts and result counts can disagree in a distributed index**  
> With sharded indexes, each shard returns its top N facet values and the coordinator merges them. A value ranked eleventh on every shard but present everywhere may be omitted, and merged counts can be slightly wrong. Exact counts require each shard to return all values, which is expensive. Most systems accept approximation and it is usually invisible — but it surprises people when a facet count does not match the number of results after selecting it.

**Worked example**

A product catalogue facet design, including the field that must not be a facet.

**Facet configuration and its costs**

```text
CATALOGUE  10M products

GOOD FACETS (bounded cardinality)
  brand            ~2,000 values   cap display at 10
  category         ~500            hierarchical
  colour           ~50
  rating           5 buckets
  price            8 range buckets (precomputed ranges)
  availability     2 values
  -> aggregation is cheap; memory is bounded

BAD FACET (proposed, rejected)
  "tags" - free-text, user-generated, ~4M distinct values
  -> 4M buckets per query
  -> tens of gigabytes of memory for a single request
  -> INSTEAD: make tags searchable, not facetable;
     or curate a bounded set of ~200 canonical tags

PRICE AS RANGES, NOT VALUES
  faceting on exact price -> ~50,000 distinct values
  faceting on 8 ranges    -> 8 buckets
  -> ranges are the only workable approach for continuous
     fields, and the range boundaries are a product decision

HIERARCHY
  category is a tree: Electronics > Computers > Laptops
  facet on the path so selecting a parent includes children
  -> index the full path as separate terms, or use a
     dedicated hierarchical facet type

LARGE RESULT SETS
  query "laptop" -> 4,000 results  -> exact counts, cheap
  query ""       -> 10M results    -> approximate counts,
                                      or precomputed for
                                      the unfiltered case
```

| Metric | Value | Note |
|---|---|---|
| Good facets | 6 fields | bounded cardinality |
| Rejected | free-text tags | **4M buckets** |
| Price | 8 ranges | not 50k values |
| Large sets | approximate | or precomputed |

> **The rejected facet is the important design decision**  
> Adding a facet is trivially easy and its cost is invisible until a query matches a large result set. A free-text tag field with millions of distinct values will work perfectly in testing against a small dataset and exhaust memory in production. Reviewing every proposed facet for cardinality — and having an alternative, such as making the field searchable rather than facetable — is the single most valuable discipline here.

**When to use it**

- **Large result sets** that users need to narrow, which is most catalogue and listing search.
- **Structured attributes** with bounded values — brand, category, colour, size, status.
- **Guiding discovery**, where showing available refinements teaches users what the corpus contains.
- **Preventing zero-result refinements**, since counts show which filters lead nowhere before they are clicked.
- **Analytics-style exploration**, where the counts themselves are the information users want.

**When to avoid it**

- **Do not facet high-cardinality fields**, which consumes memory proportional to distinct values per query.
- **Do not facet continuous values directly**; bucket them into ranges.
- **Do not apply a facet's own filter to its own counts**, which makes every alternative show zero.
- **Do not return every facet value**; cap and offer expansion.
- **Do not add facets without reviewing cardinality**, since the cost is invisible until a large query arrives.

**Advantages**

- **Turns an overwhelming result set into a navigable space**, which is the primary usability win.
- **Counts prevent dead-end refinements**, since the user sees which filters return nothing before clicking.
- **Reveals the shape of the corpus**, teaching users what attributes and values exist.
- **Filters are cheap**, being set intersections over posting lists already in the index.
- **Composes naturally with text search**, since both operate on the same matching set.

**Disadvantages**

- **Aggregation cost scales with result set size**, so broad queries are expensive.
- **Memory scales with cardinality**, making high-cardinality fields dangerous.
- **Counts may be approximate in a distributed index**, occasionally disagreeing with actual result counts.
- **Multi-select semantics are subtle**, and the naive implementation is confusingly wrong.
- **Every facet adds per-query work**, so the facet set is a latency budget to manage.
- **Requires a separate field representation** — doc values or similar — costing index space.

**Trade-offs**

**Facet field suitability**

| Field type | Cardinality | Facetable? | Alternative |
|---|---|---|---|
| Brand, category, status | Tens to thousands | Yes | — |
| Colour, size, rating | Tens | Ideal | — |
| Price, date | Continuous | As ranges only | Range buckets |
| Free-text tags | Millions | No | Searchable field; or a curated subset |
| Product id, user id | Unbounded | Never | Filter by exact value |
| Location | Continuous 2D | As geo buckets | Distance ranges or geohash prefixes |

> **Framing the decision**  
> “Facet on brand, category, colour, rating and price ranges — all bounded cardinality, so aggregation memory is fixed regardless of result set size. The proposed tags facet has four million distinct values and would allocate a bucket per value per query, so tags become a searchable field instead, with a curated set of two hundred canonical tags if we need them as a facet.”

**How it fails**

**Faceting failures**

| Symptom | Cause | Fix |
|---|---|---|
| All alternative values show zero | A facet's own filter applied to its own counts | Exclude the facet's own filter when aggregating it |
| Cluster out of memory on a broad query | High-cardinality facet field | Remove it; bucket it; or make it searchable instead |
| Facet count disagrees with result count | Per-shard top-N merging in a distributed index | Accept approximation, or increase the per-shard limit |
| Selecting a filter gives zero results | Counts computed against the wrong filter set | Recompute counts with all other filters applied |
| Query latency dominated by faceting | Too many facets, or very large result sets | Reduce facet count; approximate; precompute common cases |
| Price facet unusable | Faceting on exact values | Range buckets chosen as a product decision |
| Hierarchy filters do not include children | Faceting on leaf category only | Index the full path; use hierarchical facets |

**Limits**

> **Design guidance**
>
> - **Facet cardinality** should be bounded and ideally under a few thousand distinct values.
> - **Aggregation cost scales with result set size**, so unfiltered queries are the expensive case.
> - **Cap returned values** at around ten per facet with expansion, regardless of cardinality.
> - **Continuous fields must be bucketed** into ranges chosen as a product decision.
> - **Distributed facet counts are approximate** unless each shard returns all values.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Facets with live counts | Catalogue and listing search | Aggregation cost per query |
| Static filters without counts | Very large or high-cardinality attributes | Users hit zero-result refinements |
| Precomputed facet counts | Stable corpus and common query patterns | Combinatorial space; staleness |
| Approximate counts | Very large result sets | Counts may be slightly wrong |
| Guided navigation / curated paths | Small, editorially managed catalogues | Manual maintenance |
| Post-filtering in the client | Small result sets | Does not scale |

Static filters without counts are a legitimate fallback for attributes that cannot be faceted affordably: the user can still narrow, they simply cannot see in advance how many results remain. Mixing both — counted facets for bounded fields, uncounted filters for the rest — is common and honest.

**In real systems**

- **E-commerce catalogues** are the canonical case, where facet counts are a primary navigation mechanism and a commercial surface.
- **Elasticsearch aggregations** implement faceting, with doc values providing the column-oriented per-field storage that makes it affordable.
- **Log and observability platforms** use faceting over fields like service, level and host — and are also where high-cardinality facet mistakes most commonly cause outages.
- **Solr's faceting module** pioneered much of this, including the exclusion semantics needed for correct multi-select behaviour.
- **Distributed facet approximation** is standard in sharded search engines, with per-shard top-N merging accepted as a reasonable trade.

**Common mistakes**

- **Faceting a high-cardinality field**, exhausting memory on broad queries.
- **Applying a facet's own filter to its own counts**, so alternatives all show zero.
- **Faceting continuous values** rather than bucketing them into ranges.
- **Returning every facet value**, producing unusable lists and wasted work.
- **AND within a facet** rather than OR, which contradicts user expectation.
- **Assuming distributed counts are exact**, and building logic that breaks when they are not.
- **Adding facets without measuring their latency cost**, which accumulates silently.

**The staff-level view**

Faceting is a feature request that arrives as a small UI change and lands as an aggregation workload on the search cluster.

- **Review every proposed facet for cardinality.** The cost is invisible in testing against small data and catastrophic in production against a broad query, and this single check prevents most faceting incidents.
- **Get the multi-select semantics right once**, in shared code. The naive implementation — applying a facet's own filter to its own counts — is subtly wrong and confusing, and every team reimplements it.
- **Treat the facet set as a latency budget.** Each facet adds per-query aggregation work, so adding facets has a cost that accumulates invisibly across feature requests.
- **Offer uncounted filters as an alternative** for attributes that cannot be faceted affordably, rather than either refusing the feature or accepting the cost.
- **Be explicit that distributed counts are approximate**, so nobody builds logic that depends on them matching result counts exactly.

**Go deeper**

Faceting shows counts per attribute value alongside results, turning an overwhelming result set into a navigable space and preventing users from clicking a filter that leads to zero results. The counts are the feature: filters alone are just query parameters, while counts tell the user which refinements are worth making.

Two rules govern correctness. Each facet's counts must be computed with all *other* active filters applied but not its own, or every unselected value shows zero and the user cannot see what switching would give. And multi-select should be OR within a facet and AND across facets, which matches user expectation and means each facet aggregates against a slightly different filter set — subtle enough to be worth implementing once and sharing.

The dominant operational risk is cardinality. Aggregation allocates a counter per distinct value per query, so a bounded field like brand is trivial while a free-text tag field with millions of values can allocate tens of gigabytes for a single request — working perfectly in testing and exhausting memory in production on a broad query. Continuous fields like price must be bucketed into ranges, and fields that cannot be faceted affordably should become searchable fields or uncounted filters instead.

Faceted search looks like a filtering feature and is an aggregation workload, which is why it arrives as a small UI request and lands as a capacity problem on the search cluster.

**The counts are what matter.** Filters narrow a result set using ordinary posting-list intersection, which is cheap. Facets additionally report how many documents match each value of an attribute *within the current result set*, which requires reading that field for every matching document and tallying. That is what makes the feature valuable — the user can see which refinements are productive and which lead nowhere — and it is also what makes it expensive, because the work scales with result set size.

**Two semantic rules that are easy to get wrong.** A facet's counts must be computed against the result set filtered by all other active filters but not its own; applying its own filter makes every alternative value show zero, leaving the user unable to evaluate switching. And within a single facet, multiple selections should combine with OR — selecting two brands broadens — while across facets they combine with AND. Together these mean each facet aggregates against a slightly different filter set, which is subtle enough that it should be implemented once in shared code rather than rediscovered per team.

**Cardinality is the operational constraint.** Aggregation allocates a counter per distinct value for every query. A field with a few thousand values is trivial; one with millions allocates millions of buckets per request, consuming tens of gigabytes and timing out. The dangerous property is that this is invisible during development — a test catalogue of a thousand products with a few hundred tags behaves perfectly — and appears in production when a broad query matches a large result set. Reviewing cardinality on every proposed facet is the single most effective preventive measure.

**Continuous and hierarchical fields need special treatment.** Price and date have effectively unbounded distinct values, so they must be faceted as ranges, with the boundaries chosen as a product decision rather than derived automatically. Categories are usually trees, so faceting on the leaf alone means selecting a parent excludes its children; indexing the full path as separate terms, or using a hierarchical facet type, is required for the expected behaviour.

**Distributed counts are approximate.** In a sharded index each shard returns its top values and the coordinator merges them, so a value ranked just below the cutoff on every shard can be omitted and merged counts can be slightly off. Exact counts require every shard to return every value, which is expensive. Most systems accept the approximation, which is almost always invisible, but it surprises people when a facet count does not exactly match the number of results after selecting it — so nothing downstream should depend on the two agreeing.

**Manage the facet set as a budget.** Every facet adds per-query aggregation work, and because each individual addition looks negligible the cost accumulates invisibly across feature requests, producing gradual degradation that nobody attributes to any particular change. Requiring a cardinality review and a latency measurement per facet, publishing the per-query cost, and measuring the unfiltered browse query specifically — since that is the worst case rather than a typical filtered search — keeps the trade visible. And where an attribute genuinely cannot be faceted affordably, an uncounted filter remains a legitimate option that teams rarely consider because they assume counts are mandatory.

**Prove it — interview questions**

1. **[Basic] What makes faceting different from filtering?**

   <details><summary>Model answer</summary>

   The counts. A filter is just a constraint narrowing the result set, which is cheap and requires no aggregation. A facet additionally shows how many documents match each value given the current query and other active filters, which requires aggregating over the whole matching set. Those counts are the actual feature — they tell the user which refinements are worthwhile and which lead to zero results, converting an overwhelming list into a navigable space.

   </details>

2. **[Basic] Why must facet counts exclude their own filter?**

   <details><summary>Model answer</summary>

   Because otherwise every unselected value shows zero. If the user has selected Dell and the brand facet applies that filter to its own counts, then Apple and Lenovo both show zero matching documents, which tells the user nothing about what switching would give. The correct rule is that each facet's counts are computed against the result set filtered by all *other* active filters but not its own, so the user can always see the effect of changing that facet's selection.

   </details>

3. **[Senior] Why are high-cardinality facets dangerous?**

   <details><summary>Model answer</summary>

   Because aggregation allocates a counter per distinct value, per query. Faceting on brand with two thousand values means two thousand counters — trivial. Faceting on a free-text tag field with four million distinct values means four million buckets held in memory for a single request, which can consume tens of gigabytes and take the cluster down. The cost is invisible during development against a small dataset and appears in production when a broad query matches a large result set, which is why cardinality review on every proposed facet is the most valuable discipline in this area.

   </details>

4. **[Senior] How should multi-select behave across facets?**

   <details><summary>Model answer</summary>

   OR within a facet and AND across facets. Selecting Dell and Lenovo should show products from either brand, since the user is broadening within that dimension. Selecting Dell and sixteen gigabytes should show only Dell machines with sixteen gigabytes, since those are different dimensions being narrowed. This matches how users expect it to work and almost nothing else does, and combined with the own-filter exclusion rule it means each facet's counts are computed against a slightly different filter set — which is why the implementation is subtle enough to be worth writing once and sharing.

   </details>

5. **[Staff] A team wants to facet on user-generated tags with millions of distinct values. How do you respond?**

   <details><summary>Model answer</summary>

   By explaining the cost and offering alternatives rather than simply refusing. Faceting allocates a bucket per distinct value per query, so millions of values means the request may allocate tens of gigabytes and will work perfectly in their test environment with a thousand products. Then the alternatives: make tags a searchable field so users can find them without faceting; curate a bounded set of a couple of hundred canonical tags that can be faceted affordably; or offer tags as an uncounted filter, where the user can narrow but does not see counts in advance. The third option is often acceptable and rarely considered, because teams assume facets must have counts. What I would not do is allow it with a limit on returned values, since the memory cost is in the counters allocated during aggregation rather than in the values returned.

   </details>

6. **[Principal] How do you keep faceting from degrading search performance over time?**

   <details><summary>Model answer</summary>

   By treating the facet set as a governed latency budget rather than a list that grows with feature requests. Each facet adds aggregation work to every query, so the cost accumulates invisibly as individual additions each look negligible — and the degradation appears gradually rather than as an incident anyone can attribute. Concretely, I would require a cardinality review and a latency measurement for any new facet, publish the per-query facet cost so the trade is visible, and set a maximum facet count per search surface that requires a deliberate decision to exceed. I would also make the expensive case explicit: aggregation scales with result set size, so the unfiltered browse query is the worst case and should be measured specifically rather than assuming a typical filtered search is representative. Where the budget is genuinely exceeded, precomputing counts for the most common query-and-filter combinations is the usual escape, bounded by the combinatorial space of possible filter states.

   </details>

---

### Vector similarity search

*Represent content as points in a high-dimensional space so that semantically similar items are near each other, and search by proximity rather than by shared words.*

**Flow:** `Content` → `Embedding model` → `Vector index` → `Query vector` → `Nearest results`

> **The 30-second version**  
> Embed content as points in a high-dimensional space so similar meanings are near each other, then search by proximity — excellent for paraphrase, unreliable for exact identifiers.

**The problem**

A user searches for “how do I stop my application crashing when memory runs out” and the relevant document is titled “Handling OOM errors in production”. They share almost no words. Lexical search matches tokens, so it finds nothing — and the user concludes the search is broken while the answer sits in the index.

The gap is between words and meaning. Synonyms, paraphrases, related concepts and different vocabularies all describe the same thing without sharing terms, and no amount of stemming or synonym lists closes it in general, because the space of ways to express an idea is unbounded.

> **Embeddings turn meaning into geometry**  
> A model maps text into a vector such that semantically similar text produces nearby vectors. Similarity then becomes distance, and search becomes a nearest-neighbour query. The strength is that it generalises — it handles paraphrases nobody wrote a synonym rule for. The weakness is the same property: it has no notion of exactness, so a specific model number is just another point in space.

**Mental model**

Every document becomes a point in a space of several hundred dimensions. The query becomes a point too. The answer is whichever document points lie closest, measured by cosine similarity or Euclidean distance.

1. **Embedding model** — Maps content to a vector. The quality of the entire system is bounded by how well it captures meaning for your domain.
2. **Dimensionality** — Typically a few hundred to a few thousand. Higher captures more nuance and costs proportionally more memory and computation.
3. **Distance metric** — Cosine similarity for direction-based meaning, Euclidean for magnitude-sensitive cases. The model determines which is appropriate.
4. **Index** — A structure enabling approximate nearest-neighbour search, because exact search over millions of vectors is too slow.
5. **Chunking** — How documents are split before embedding, since a model has a bounded input length and a whole document averages into an unhelpful point.

> **Embeddings inherit everything about the model that produced them**  
> Vectors from different models are incomparable, so changing models means re-embedding the entire corpus. The model's training data determines what it understands — a general model may not distinguish two technical terms your users treat as opposites. And the model's biases become the search's biases, silently.

**How it works**

**From content to results**

```text
INDEXING
  document -> chunk into passages (~200-500 tokens)
           -> embed each chunk -> vector[768]
           -> store vector + chunk + document reference

QUERYING
  query text -> embed with the SAME model -> vector[768]
             -> find nearest vectors by cosine similarity
             -> return the associated chunks/documents

WHY CHUNKING MATTERS
  embedding a 50-page document produces ONE vector that
  averages every topic in it -> close to nothing specific
  embedding 100 passages produces 100 specific vectors
  -> a query about one paragraph matches that paragraph
  -> chunk size is a real quality lever: too small loses
     context, too large dilutes meaning

COSINE SIMILARITY
  cos(a,b) = (a . b) / (|a| |b|)
  measures ANGLE, ignoring magnitude
  -> 1.0 identical direction, 0 unrelated, -1 opposite
  most text embedding models are trained for this metric
```

1. **Use the same model for indexing and querying** — Vectors from different models occupy different spaces and are meaningless to compare. This is the vector equivalent of an analyser mismatch.
2. **Chunk deliberately** — Chunk size and overlap are among the largest quality levers available, and they are usually set once arbitrarily and never revisited.
3. **Store the source text alongside the vector** — A vector is not reversible into its content, so retrieval must return the original chunk. The vector index is an index, not a store.
4. **Expect to re-embed on model changes** — Upgrading the embedding model invalidates every stored vector, requiring a full re-embedding run — which for a large corpus is a significant compute cost.
5. **Pair it with lexical search** — Vectors cannot match exact identifiers, product codes or rare proper nouns reliably. Hybrid retrieval covers the gap in both directions.
6. **Filter before or during search, carefully** — Combining metadata filters with vector search is genuinely awkward: filtering after retrieval may leave too few results, while filtering during traversal can degrade the index's guarantees.

**What vectors are good and bad at**

```text
VECTORS EXCEL AT
  paraphrase      "car won't start" ~ "vehicle fails to ignite"
  synonymy        "laptop" ~ "notebook computer"
  conceptual      "memory leak" ~ "heap growth over time"
  cross-lingual   with a multilingual model
  typo tolerance  similar spellings embed similarly

VECTORS FAIL AT
  exact codes     "WH-1000XM5" - just a point in space;
                  "WH-1000XM4" is extremely close, which is
                  WRONG for a user who wants a specific model
  rare terms      a proper noun the model never saw
  negation        "not waterproof" often embeds close to
                  "waterproof" - models handle it poorly
  numeric ranges  "under 500 grams" is not a geometric concept
  freshness       no notion of recency at all

CONSEQUENCE
  a pure vector search feels intelligent on descriptive
  queries and unreliable on precise ones - which are exactly
  the queries where the user is most certain what they want.
```

> **Vector search has no notion of exactness**  
> Two model numbers differing in one character embed to nearly identical vectors, so a user searching for a specific product may get its predecessor. Negation is frequently ignored, so “not waterproof” retrieves waterproof items. Numeric constraints have no geometric meaning at all. These are not tuning problems; they are consequences of representing meaning as position, and they are why pure vector search is rarely the whole answer.

**Worked example**

Adding semantic search to a documentation site, including the decisions that determine quality.

**Pipeline and the levers that matter**

```text
CORPUS  20,000 documentation pages

CHUNKING (the biggest quality lever)
  split by heading, then by paragraph, targeting ~300 tokens
  with ~50 tokens of overlap so context is not lost at
  boundaries
  -> ~180,000 chunks
  too large: a chunk covering three topics matches none well
  too small: "see the previous section" loses its referent

EMBEDDING
  model: 768 dimensions
  180,000 x 768 x 4 bytes = ~550 MB of raw vectors
  -> fits in memory; an ANN index adds overhead

INDEX
  HNSW graph, ~1.5x memory overhead
  query latency ~5ms for approximate top-20

RETRIEVAL
  embed the query, find top 50 by cosine similarity
  -> rerank with a cross-encoder for the top 10
     (far more accurate, far too slow for 180k chunks)

HYBRID
  run BM25 in parallel; fuse the rankings
  -> BM25 catches exact API names and error codes
  -> vectors catch conceptual queries
  -> fusion covers both, which neither does alone

METADATA
  store product version, section and date with each chunk
  -> filter to the user's version before or during search
```

| Metric | Value | Note |
|---|---|---|
| Chunks | 180,000 | from 20k pages |
| Vectors | ~550 MB | 768 dimensions |
| Query | ~5 ms | approximate top-20 |
| Hybrid | BM25 + vectors | **covers both gaps** |

> **Chunking quality dominates model quality**  
> Teams spend considerable effort choosing between embedding models whose benchmark scores differ by a few percent, and almost none on chunk size and boundaries — which routinely change retrieval quality far more. A chunk spanning three unrelated topics embeds to an average that matches none of them; a chunk cut mid-explanation loses the context that made it meaningful. Getting the chunking right is usually the highest-return work available.

**When to use it**

- **Semantic and conceptual search**, where users describe what they want in their own words.
- **Question answering and retrieval-augmented generation**, where relevant passages must be found without keyword overlap.
- **Recommendation by similarity**, where content or behaviour is embedded and nearby items are suggested.
- **Multilingual search**, where a multilingual model matches across languages without translation.
- **As one half of hybrid retrieval**, complementing lexical search rather than replacing it.

**When to avoid it**

- **Do not use it alone for exact matching** — identifiers, model numbers, error codes and rare proper nouns need lexical search.
- **Do not rely on it for negation or numeric constraints**, which have no geometric representation.
- **Do not compare vectors from different models**, which occupy different spaces.
- **Do not treat the vector index as a store**, since vectors cannot be reversed into content.
- **Do not ignore chunking**, which affects quality more than the model choice usually does.

**Advantages**

- **Matches meaning rather than words**, handling paraphrases nobody anticipated.
- **Generalises without maintenance**, unlike synonym lists that must be curated forever.
- **Works across languages** with a multilingual model.
- **Tolerant of typos and phrasing variation**, since similar spellings embed similarly.
- **Enables similarity recommendation** from the same index, without a separate system.

**Disadvantages**

- **No exactness** — precise identifiers and codes are unreliable.
- **Negation and numeric constraints are not represented**, and are handled poorly or not at all.
- **Model changes require re-embedding the whole corpus**, a significant compute cost.
- **Memory-hungry**, since vectors and the index must be resident for acceptable latency.
- **Opaque**, so a wrong result cannot be explained the way a term match can.
- **Filtering interacts awkwardly** with approximate nearest-neighbour indexes.

**Trade-offs**

**Vector versus lexical retrieval**

|  | Vector | Lexical (BM25) |
|---|---|---|
| Paraphrase and synonymy | Excellent | Only via curated synonyms |
| Exact identifiers | Unreliable | Excellent |
| Rare proper nouns | Poor if unseen in training | Excellent |
| Negation | Poor | Poor, but explicitly expressible |
| Explainability | Opaque | Which terms matched |
| Index cost | High memory | Compressed postings |
| Maintenance | Re-embed on model change | Reindex on analyser change |

> **Framing the choice**  
> “Hybrid, not either. Vectors handle the conceptual queries where users describe a problem in their own words, BM25 handles exact API names, error codes and version numbers — and those are exactly the queries where users are most certain what they want and least tolerant of being given something similar instead. Fusing both rankings covers each other's blind spot.”

**How it fails**

**Vector search failures**

| Symptom | Cause | Fix |
|---|---|---|
| Wrong model variant returned | Near-identical identifiers embed to near-identical vectors | Hybrid with lexical matching; boost exact matches |
| Query about a specific term finds nothing relevant | Term unseen by the model | Lexical fallback; domain-tuned model |
| Negated query returns the opposite | Models represent negation poorly | Extract negation into a metadata filter |
| Results vaguely related but never precise | Chunks too large, averaging multiple topics | Smaller chunks with overlap |
| Answers cut off mid-explanation | Chunks too small or boundaries poorly chosen | Larger chunks; split on semantic boundaries |
| Quality collapsed after a model upgrade | Mixed vectors from two models in one index | Re-embed everything; never mix |
| Filtered queries return too few results | Post-filtering after approximate retrieval | Filter during traversal, or retrieve more candidates |

**Limits**

> **Sizing and behaviour**
>
> - **Dimensionality**: typically 384–1,536; memory is dimensions × 4 bytes × vector count, plus index overhead.
> - **Chunk size**: commonly 200–500 tokens with 10–20% overlap, and this matters more than the model choice.
> - **Approximate search** is necessary above roughly a hundred thousand vectors; exact search does not scale.
> - **Re-embedding on model change** is a full corpus pass — plan it as a migration.
> - **Hybrid retrieval** is the practical default; neither method alone covers the query distribution.

**Alternatives**

| Approach | Strength | Weakness |
|---|---|---|
| Vector similarity | Meaning, paraphrase, cross-lingual | Exactness, negation, numbers |
| BM25 lexical | Exact terms, explainable | No semantics |
| Hybrid fusion | Covers both | Two indexes; fusion tuning |
| Curated synonyms | Precise, explainable domain knowledge | Manual; does not generalise |
| Query rewriting with an LLM | Handles intent and negation | Latency; cost; nondeterminism |
| Cross-encoder reranking | Much higher accuracy | Far too slow for retrieval; use on candidates only |

Cross-encoder reranking is the standard third stage: retrieve candidates cheaply with vectors or BM25, then score the top few dozen with a model that reads the query and document together. It is substantially more accurate and far too slow to run over a corpus, which is exactly why the pipeline is staged.

**In real systems**

- **Retrieval-augmented generation** depends on this pipeline, and its quality is usually limited by chunking and retrieval rather than by the language model.
- **Hybrid search** combining BM25 with dense vectors is now standard in major search engines, because each covers the other's failure mode.
- **Recommendation systems** embed items and users in a shared space, making similarity search the core retrieval mechanism.
- **Multilingual embedding models** enable cross-language retrieval without translating the corpus, which is otherwise very expensive.
- **Cross-encoder rerankers** are the standard accuracy layer over cheap retrieval, reflecting the two-stage pattern seen throughout search.

**Common mistakes**

- **Replacing lexical search entirely**, breaking exact-match queries.
- **Arbitrary chunk sizes**, which dominate quality and are rarely revisited.
- **Mixing vectors from different models** in one index.
- **Expecting negation and numeric constraints** to work.
- **Treating the vector index as a document store**, when vectors cannot be reversed.
- **Post-filtering after approximate retrieval**, leaving too few results.
- **No plan for re-embedding**, freezing the system on its initial model.

**The staff-level view**

Vector search is frequently adopted as a replacement for lexical search and almost always ends up alongside it.

- **Insist on hybrid from the start.** A pure vector system fails precisely on the queries where users are most certain — exact identifiers, error codes, version numbers — and that failure reads as the search being unreliable rather than merely imperfect.
- **Spend effort on chunking before model selection.** Chunk size and boundaries routinely affect retrieval quality more than the difference between competing models, and they are usually set arbitrarily.
- **Plan re-embedding as a routine migration.** Model upgrades invalidate the entire corpus, and a team that cannot re-embed confidently is frozen on whichever model they started with.
- **Be explicit about what vectors cannot do.** Negation, numeric ranges and exactness have no geometric representation, so those constraints must be extracted into filters rather than left to the model.
- **Budget the memory honestly.** Vectors plus index overhead must be resident for acceptable latency, and this is a substantially larger footprint than an inverted index over the same corpus.

**Go deeper**

An embedding model maps content into a vector such that semantically similar text produces nearby vectors, so search becomes a nearest-neighbour query rather than a term match. That closes the gap between words and meaning: a query about an application crashing when memory runs out can match a document about handling out-of-memory errors despite sharing no vocabulary, and it generalises to paraphrases nobody wrote a synonym rule for.

The same property creates the weaknesses. Vectors have no notion of exactness, so two model numbers differing by one character embed almost identically and a user wanting a specific product may get its predecessor. Negation is represented poorly, numeric constraints have no geometric meaning, and rare proper nouns the model never saw are effectively invisible. These are consequences of representing meaning as position rather than tuning problems.

Consequently hybrid retrieval — fusing vector similarity with BM25 — is the practical default, since each covers the other's blind spot and the lexical failures matter most for the queries users are most certain about. Two operational facts dominate: chunking quality usually affects results more than model choice, and upgrading the embedding model invalidates the entire corpus, so re-embedding must be a routine, planned migration rather than an obstacle.

Vector search represents meaning as position in a high-dimensional space, which is simultaneously why it generalises so well and why it fails so specifically.

**The mechanism.** An embedding model maps content into a vector — typically a few hundred to a couple of thousand dimensions — trained so that semantically similar inputs land near each other. Documents and queries pass through the same model, and retrieval finds the nearest vectors, usually by cosine similarity since most text models are trained for angular distance. Above roughly a hundred thousand vectors, exact search becomes too slow and approximate nearest-neighbour indexes are required, trading a small recall loss for order-of-magnitude speed.

**Chunking dominates quality more than model choice.** A single vector represents whatever was embedded, so a whole document becomes an average of all its topics — close to nothing specific. Splitting into passages of a few hundred tokens with modest overlap produces vectors that can actually match a question about one paragraph. Chunks that are too large dilute meaning; chunks that are too small lose the context that made them intelligible. Teams routinely spend effort comparing models whose benchmarks differ by a few percent while leaving chunk size at an arbitrary default that costs far more.

**What geometry cannot express.** Exactness has no representation: two identifiers differing by a character are adjacent points, so a precise model number returns its neighbour — which reads as unreliability rather than imperfection, because the user knew exactly what they wanted. Negation is handled poorly by most models, so “not waterproof” frequently retrieves waterproof items. Numeric ranges are not geometric concepts at all. Recency, popularity and authority are absent entirely. The correct response is not to tune but to extract these constraints elsewhere — metadata filters for numbers and categories, lexical matching for exact terms.

**Hybrid retrieval is the practical default.** Lexical search fails on paraphrase; vector search fails on exactness. Fusing both rankings covers each blind spot, and the asymmetry matters: a user describing a problem vaguely is tolerant of approximate results, while a user typing an error code or a version number is not. That is why major systems added vectors alongside BM25 rather than replacing it, and why a pure vector deployment tends to feel impressive in demonstrations and frustrating in use.

**Operational costs are real and often underestimated.** Vectors plus approximate-index overhead must be memory-resident for acceptable latency, which for a large corpus substantially exceeds an inverted index over the same content. Upgrading the embedding model invalidates every stored vector, since vectors from different models occupy different spaces and cannot be compared — so a full re-embedding pass is required, and a team that has not made that routine is effectively frozen on whichever model it started with. And combining metadata filters with approximate search is genuinely awkward: filtering after retrieval can leave too few results, while filtering during graph traversal degrades the index's recall guarantees.

**Stage the pipeline and measure it.** The mature shape is cheap retrieval — vectors and BM25 in parallel — feeding a fusion step, feeding a cross-encoder reranker over the top few dozen candidates, which reads query and document together and is far more accurate and far too slow to run corpus-wide. Underneath, judged relevance or click data is what makes iteration meaningful: without it, every change to chunk size, fusion weight or model is an opinion, and teams iterate indefinitely without knowing whether anything improved.

**Prove it — interview questions**

1. **[Basic] How does vector search find semantically similar documents?**

   <details><summary>Model answer</summary>

   An embedding model maps content into a vector of several hundred dimensions such that semantically similar content produces nearby vectors. Both documents and queries are embedded with the same model, and search becomes finding the nearest vectors to the query point, typically by cosine similarity. Because similarity is geometric rather than lexical, a query and a document can match strongly while sharing no words at all.

   </details>

2. **[Basic] Why does chunking matter?**

   <details><summary>Model answer</summary>

   Because a single vector represents the whole of whatever was embedded. Embedding a fifty-page document produces one point that averages every topic in it, which is close to nothing specific and matches poorly. Splitting it into passages produces many specific vectors, so a query about one paragraph can match that paragraph. But chunks that are too small lose context — a passage beginning “as described above” has no referent — so chunk size and overlap are genuine quality levers, and in practice they affect results more than the choice of model.

   </details>

3. **[Senior] What can vector search not do?**

   <details><summary>Model answer</summary>

   Exactness, negation and numbers. Two model numbers differing by one character embed to nearly identical vectors, so a user searching for a specific product may receive its predecessor — which is wrong in a way that feels unreliable rather than merely imperfect. Negation is represented poorly, so “not waterproof” often retrieves waterproof items. Numeric constraints like “under five hundred grams” have no geometric meaning at all. None of these are tuning problems; they follow from representing meaning as position, which is why numeric and categorical constraints should be extracted into metadata filters and exact matching should come from a lexical index.

   </details>

4. **[Senior] Why is hybrid retrieval the practical default?**

   <details><summary>Model answer</summary>

   Because each method fails exactly where the other succeeds. Lexical search cannot match a paraphrase, so a user describing a problem in their own words finds nothing. Vector search cannot match an exact identifier reliably, so a user typing a precise model number or error code gets something similar instead. The second failure is worse in practice, because those are the queries where the user knows exactly what they want and is least tolerant of approximation. Running both and fusing the rankings covers both blind spots, which is why major search systems converged on it rather than replacing lexical retrieval.

   </details>

5. **[Staff] What is the operational cost of adopting vector search?**

   <details><summary>Model answer</summary>

   Three things beyond the obvious. Memory: vectors plus approximate-index overhead must be resident for acceptable latency, and for a large corpus that is substantially more than an inverted index over the same content. Re-embedding: upgrading the embedding model invalidates every stored vector, so the entire corpus must be re-embedded — a full compute pass that has to be planned as a migration, and a team that cannot do it confidently is frozen on whichever model they happened to start with. And filtering: combining metadata constraints with approximate nearest-neighbour search is genuinely awkward, since filtering after retrieval can leave too few results while filtering during graph traversal degrades the index's guarantees, so this needs to be designed rather than assumed.

   </details>

6. **[Principal] How would you approach retrieval quality for a documentation or knowledge system?**

   <details><summary>Model answer</summary>

   By treating retrieval as a staged pipeline and being clear about where quality actually comes from. Chunking first, because it dominates — chunk boundaries aligned to semantic structure with modest overlap routinely matter more than the difference between competing embedding models, and it is the step teams skip. Then hybrid retrieval, since documentation queries split cleanly into conceptual questions where vectors win and exact API names, error codes and version identifiers where lexical wins, and a system that fails the second category reads as unreliable. Then metadata filtering for constraints that have no semantic representation — product version especially, since returning correct information for the wrong version is worse than returning nothing. Then cross-encoder reranking over the top candidates, which is substantially more accurate and far too slow to run over a corpus. And underneath all of it, measurement: without judged relevance or click data, every change to chunking, fusion weights or models is an opinion, and the team will iterate indefinitely without knowing whether anything improved.

   </details>

---

### Approximate nearest neighbors

*Give up a small amount of recall to make similarity search fast, because exact nearest-neighbour search over millions of high-dimensional vectors is not tractable.*

**Flow:** `Query vector` → `Entry candidates` → `Approximate exploration` → `Candidate filter` → `Top-k`

> **The 30-second version**  
> Trade a few percent of recall for orders of magnitude of speed, because exact nearest-neighbour search in high dimensions is no better than a scan.

**The problem**

Finding the closest vector to a query among ten million vectors of 768 dimensions means computing ten million distance calculations, each over 768 numbers — roughly eight billion floating-point operations per query. At any meaningful query rate this is not a latency problem, it is an impossibility.

Worse, the usual spatial data structures do not rescue you. Trees that partition space work well in two or three dimensions and degenerate badly as dimensionality rises, until they examine nearly every point and perform worse than a brute-force scan.

> **The curse of dimensionality is why this is hard**  
> In high dimensions, distances between random points concentrate: the nearest and farthest neighbours become nearly equidistant, and the volume of space grows so fast that any partitioning scheme must examine an enormous fraction of it to be certain. Exact nearest-neighbour search in high dimensions provably cannot be done much faster than a scan — which is why every practical system accepts approximation.

**Mental model**

Instead of guaranteeing the true nearest neighbours, explore a structure that usually finds them. Recall — the fraction of true neighbours returned — becomes a tunable parameter traded against latency and memory.

1. **Recall** — The proportion of the true top-k that the approximate search actually returns. Typically tuned to 0.9–0.99.
2. **HNSW** — A layered navigable graph. Search starts at a sparse top layer and descends, greedily moving toward the query. Fast and accurate; memory-hungry.
3. **IVF** — Partition vectors into clusters; search only the nearest few clusters. Cheap to build, and recall depends on how many clusters you probe.
4. **Product quantisation** — Compress vectors by splitting them into subvectors and replacing each with a codebook entry. Dramatically reduces memory, at some accuracy cost.
5. **Search parameters** — Every index exposes a knob — how many neighbours to explore, how many clusters to probe — that trades recall against latency at query time.

> **Recall is a dial, not a property**  
> The same index can return 85% recall in one millisecond or 99% in ten, purely by adjusting how much of the structure is explored at query time. That means the question is never “is it accurate?” but “what recall do we need, and what does it cost?” — and for most applications 95% recall is indistinguishable from exact, because the missing neighbours were marginal anyway.

**How it works**

**How the main approaches work**

```text
HNSW  (hierarchical navigable small world)
  a multi-layer graph; upper layers are sparse long-range
  links, lower layers dense local ones
  search: start at the top, greedily hop toward the query,
          descend a layer, repeat
  -> logarithmic-ish hops instead of a linear scan
  + excellent recall/latency; no training step
  - high memory (graph edges per vector)
  - deletions are awkward (tombstones and rebuilds)

IVF  (inverted file / cluster-based)
  cluster all vectors (k-means) into, say, 4,096 lists
  index stores which list each vector belongs to
  search: find the nearest few centroids, scan only those
          lists
  + low memory; simple; fast build
  - recall depends on nprobe (how many lists scanned)
  - requires training on a sample; poor if the distribution
    shifts

PRODUCT QUANTISATION  (compression, combined with the above)
  split a 768-dim vector into 96 subvectors of 8 dims
  replace each with the nearest of 256 codebook entries
  -> 96 bytes instead of 3,072 bytes: 32x smaller
  + makes billion-scale corpora fit in memory
  - distances become approximate, lowering recall
  - usually paired with a re-rank on exact vectors
```

1. **Decide the recall target from the application, not from the default** — A recommendation feed tolerates 90% recall invisibly; a legal search where a missing document matters may not. The target determines the index and its parameters.
2. **Expect to rebuild rather than update** — Graph-based indexes handle insertion reasonably and deletion poorly. High-churn corpora usually need periodic rebuilds, which must be planned for.
3. **Combine compression with re-ranking** — Retrieve a larger candidate set using compressed vectors, then compute exact distances on those few — recovering most of the lost accuracy for a fraction of the cost.
4. **Measure recall against a brute-force baseline** — Compute exact neighbours for a sample of queries once, then measure what the index returns. Without this, recall is an assumption.
5. **Treat filtering as a first-class requirement** — Combining metadata filters with graph traversal is genuinely hard, and the choice between pre-filtering, post-filtering and filtered traversal has large consequences.
6. **Budget memory honestly** — Raw vectors, graph edges or cluster assignments, and any re-ranking copies all consume memory, and latency collapses once the index no longer fits.

**The filtering problem, which is underestimated**

```text
QUERY  "similar products, but only in stock and under £500"

POST-FILTER (search then filter)
  retrieve top 100 by similarity
  apply filters -> 3 survive
  -> the user asked for 20; you have 3
  -> retrieving top 1,000 instead may still not be enough
     if the filter is selective

PRE-FILTER (filter then exact search)
  find all in-stock items under £500 -> 50,000 vectors
  brute-force search those 50,000
  -> correct, and possibly fast enough at this size
  -> hopeless if the filtered set is millions

FILTERED TRAVERSAL (filter during graph search)
  traverse the graph but only accept matching nodes
  -> the graph's connectivity assumes all nodes are
     reachable; filtering can disconnect regions and
     recall collapses
  -> works well for weak filters, badly for selective ones

THERE IS NO UNIVERSALLY CORRECT ANSWER.
The right choice depends on filter selectivity, which
varies per query - so mature systems choose dynamically.
```

> **Recall degrades silently**  
> An approximate index that returns slightly worse results produces no error, no warning and no visible symptom — the results simply become marginally less relevant. Index parameters drift, the data distribution shifts, compression is added for memory reasons, and recall falls from 98% to 80% with nobody noticing. Continuous recall measurement against an exact baseline is the only defence, and it is routinely absent.

**Worked example**

Sizing an index for ten million vectors and choosing between approaches.

**Memory, latency and recall compared**

```text
CORPUS  10M vectors, 768 dimensions, float32

RAW VECTORS
  10M x 768 x 4 B = 30 GB
  -> already too large for a comfortable single node

OPTION A  HNSW on raw vectors
  vectors 30 GB + graph edges ~8 GB = ~38 GB
  recall 0.98 at ~3ms
  -> needs a large-memory machine or sharding
  -> best quality; highest cost

OPTION B  IVF, nprobe = 32 of 4,096 clusters
  vectors 30 GB + small centroid overhead
  scans ~0.8% of vectors per query = 80,000 distances
  recall ~0.92 at ~5ms
  -> similar memory, lower recall, simpler

OPTION C  IVF + product quantisation (96 bytes/vector)
  10M x 96 B = ~1 GB
  recall ~0.85 at ~2ms
  + re-rank top 200 using exact vectors from disk
  -> recall recovers to ~0.95
  -> 30x less memory; fits on a modest machine

THE DECISION
  if 30 GB of RAM is affordable: HNSW, best quality
  if not: quantisation plus re-ranking recovers most of
  the quality at a fraction of the memory
  -> memory budget, not accuracy, is usually what decides
```

| Metric | Value | Note |
|---|---|---|
| Raw | 30 GB | 10M × 768 dims |
| HNSW | 38 GB, 0.98 | best quality |
| PQ + rerank | ~1 GB, 0.95 | **30× less memory** |
| Decided by | memory budget | not accuracy |

> **Quantisation plus re-ranking is usually the right shape**  
> Compressed vectors are cheap enough to hold an enormous corpus in memory but lose accuracy. Retrieving a larger candidate set from the compressed index and then computing exact distances on just those few hundred recovers nearly all the lost recall at negligible cost. That two-stage structure — cheap approximate retrieval, expensive exact scoring on candidates — mirrors the same pattern found throughout search.

**When to use it**

- **Any vector search beyond roughly a hundred thousand vectors**, where exact search stops being viable.
- **Recommendation and similarity systems**, where approximate neighbours are indistinguishable from exact ones.
- **Retrieval-augmented generation**, where candidates are reranked afterwards anyway.
- **Real-time serving**, where a few milliseconds of latency budget rules out exhaustive search.
- **Large corpora with a memory constraint**, where quantisation is the only way to fit.

**When to avoid it**

- **Do not use approximation for small corpora**, where brute force is exact, simple and fast enough.
- **Do not assume default parameters give acceptable recall**; measure against an exact baseline.
- **Do not post-filter with selective filters**, which leaves too few results.
- **Do not ignore the churn profile** — graph indexes handle deletion badly and may need rebuilds.
- **Do not deploy without recall monitoring**, since degradation is entirely silent.

**Advantages**

- **Makes similarity search tractable** at scales where exact search is impossible.
- **Recall is tunable at query time**, so the same index serves different latency and quality requirements.
- **Quantisation reduces memory by an order of magnitude or more**, which often decides feasibility.
- **Graph-based methods achieve excellent recall/latency trade-offs** without training.
- **Composes with re-ranking**, recovering accuracy lost to compression cheaply.

**Disadvantages**

- **Recall is below one hundred per cent**, and the missing results are invisible.
- **Memory-hungry**, particularly graph indexes, whose performance collapses if they do not fit.
- **Deletions are awkward**, typically handled by tombstones with periodic rebuilds.
- **Filtering interacts badly** with approximate structures, and no approach is universally correct.
- **Parameters are numerous and interacting**, so tuning is empirical rather than principled.
- **Cluster-based methods need training** and degrade when the data distribution shifts.

**Trade-offs**

**Index types compared**

| Index | Memory | Recall | Build cost | Deletions |
|---|---|---|---|---|
| Brute force | Raw vectors | 1.0 exact | None | Trivial |
| HNSW | Vectors + graph | 0.95–0.99 | Moderate | Poor — tombstones |
| IVF | Vectors + centroids | 0.85–0.95 | Training required | Reasonable |
| IVF + PQ | Highly compressed | 0.80–0.90, 0.95 with rerank | Training required | Reasonable |
| Disk-based (DiskANN-style) | Small in memory | 0.90–0.95 | High | Rebuild-oriented |

The final row matters at very large scale: keeping most of the index on SSD with a compressed in-memory portion makes billion-vector corpora affordable, accepting higher latency per query. That is the point at which the design stops being about algorithms and becomes about storage hierarchy.

**How it fails**

**ANN failure modes**

| Symptom | Cause | Fix |
|---|---|---|
| Results quietly worse than before | Recall degraded by parameter or data drift | Continuous recall measurement against an exact baseline |
| Filtered queries return too few results | Post-filtering after approximate retrieval | Pre-filter when selective; filtered traversal when not |
| Latency collapses as the corpus grows | Index no longer fits in memory | Quantisation; sharding; disk-based index |
| Recall falls after adding compression | Quantisation error | Re-rank candidates on exact vectors |
| Deleted items still returned | Tombstones not yet compacted | Filter deleted ids at query time; schedule rebuilds |
| Recall degrades over months | Data distribution drifted from the training sample | Retrain clusters; rebuild the index |
| Query latency highly variable | Graph traversal depth varies with the query | Cap exploration; measure p99 rather than mean |

**Limits**

> **Sizing and tuning**
>
> - **Exact search stops scaling** around a hundred thousand vectors for interactive latency.
> - **Raw memory** = vectors × dimensions × 4 bytes; a graph index adds roughly 20–50% on top.
> - **Product quantisation** commonly gives 16–32× compression at a recall cost recoverable by re-ranking.
> - **Recall targets**: 0.95 is usually indistinguishable from exact for recommendation and retrieval.
> - **Measure recall continuously**, since degradation produces no error and no symptom.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Brute force | Under ~100k vectors | Linear cost |
| HNSW | Best recall/latency, memory available | Memory; poor deletion |
| IVF | Large corpora, simpler operation | Lower recall; needs training |
| IVF + PQ | Very large corpora, memory constrained | Compression error; needs re-ranking |
| Disk-based ANN | Billion-scale on modest hardware | Higher latency |
| Sharded exact search | Moderate corpora, exactness required | Cost scales linearly |

Sharded brute force deserves mention: splitting a corpus across many nodes and scanning each in parallel gives exact results with predictable latency, and for corpora of a few million vectors it can be simpler and more reliable than an approximate index — the cost is linear in corpus size, which is precisely the cost ANN exists to avoid.

**In real systems**

- **HNSW** is the default in most vector databases and search engines because its recall-versus-latency curve is hard to beat and it needs no training step.
- **FAISS** popularised IVF and product quantisation, and its combinations remain the reference for memory-constrained billion-scale search.
- **DiskANN-style approaches** keep most of the index on SSD with a compressed in-memory portion, making very large corpora affordable.
- **Vector databases** expose recall-tuning parameters directly, which makes the approximation explicit rather than hidden.
- **Filtered vector search** remains an active area precisely because no existing approach handles selective filters and high recall simultaneously.

**Common mistakes**

- **Deploying without measuring recall**, so silent degradation goes unnoticed for months.
- **Accepting default parameters** without knowing what recall they produce.
- **Post-filtering with selective filters**, returning far fewer results than requested.
- **Ignoring memory headroom**, so latency collapses when the index no longer fits.
- **Using an approximate index for a small corpus**, adding complexity for no benefit.
- **Assuming deletions are cheap** in graph-based indexes.
- **Never retraining cluster-based indexes**, so recall drifts as the data distribution changes.

**The staff-level view**

Approximate search introduces a quality dimension that silently degrades, which makes measurement the central discipline rather than algorithm choice.

- **Measure recall continuously against an exact baseline.** Compute true neighbours for a fixed query sample periodically and compare — without this, recall drifts with parameters, data and compression changes and nobody notices.
- **Make the recall target an explicit requirement**, since it determines index choice, memory budget and cost. Defaults encode somebody else's answer to a question you have not asked.
- **Plan for filtering before choosing an index.** Selective metadata filters break approximate traversal, and retrofitting a solution is much harder than choosing an index that supports it.
- **Budget memory as the primary constraint.** In practice the decision between approaches is made by what fits, not by which has the best benchmark.
- **Ask whether brute force suffices.** Below a few hundred thousand vectors, exact search is simpler, has no recall question, and removes an entire operational surface.

**Go deeper**

Exact nearest-neighbour search over millions of high-dimensional vectors is intractable: space-partitioning structures degenerate as dimensionality rises until they examine nearly everything, so nothing beats a brute-force scan — and a scan is billions of operations per query. Approximate methods accept returning most rather than all of the true neighbours, with recall as a tunable query-time parameter.

Two families dominate. Graph indexes such as HNSW navigate from sparse long-range links toward the query through progressively denser layers, giving the best recall-versus-latency curve with no training, at the cost of substantial memory and awkward deletions. Cluster-based indexes such as IVF partition with k-means and probe only the nearest clusters, using less memory and building faster but needing training and degrading if the data distribution shifts. Product quantisation compresses vectors sixteen- to thirty-two-fold, making very large corpora fit in memory, with the accuracy loss recovered by re-ranking a candidate set on exact vectors.

Two things dominate practice. Memory budget usually decides the approach rather than benchmark accuracy, since latency collapses once the index no longer fits. And recall degrades silently — no error, no symptom, just marginally worse results — so continuous measurement against an exact baseline is the essential discipline, and it is almost always absent.

Approximate nearest-neighbour search exists because exact search in high dimensions provably cannot be made much faster than a scan, and a scan is unaffordable at any interesting scale.

**Why the problem is hard.** Space-partitioning trees that work well in two or three dimensions degenerate as dimensionality rises: distances between random points concentrate, so nearest and farthest neighbours become nearly equidistant, and the volume of space grows so rapidly that pruning becomes ineffective. Past a few dozen dimensions such structures examine most of the data and perform worse than brute force. Since text embeddings are typically several hundred to a couple of thousand dimensions, exact search is a non-starter above roughly a hundred thousand vectors.

**Recall is a dial rather than a property.** Approximate methods return most of the true top-k, and how much of the structure is explored at query time determines both recall and latency. The same index can give eighty-five per cent recall in one millisecond or ninety-nine per cent in ten. The correct framing is therefore not whether the index is accurate but what recall the application needs and what it costs — and for recommendation, retrieval-augmented generation and most similarity use cases, ninety-five per cent is indistinguishable from exact, because the missing neighbours were marginal.

**The main families and their trade-offs.** HNSW builds a layered navigable graph, searching from sparse long-range links down into dense local ones, and achieves the best recall-versus-latency curve available with no training step — paying in memory for the graph edges and handling deletions poorly, usually via tombstones and periodic rebuilds. IVF clusters the vectors and probes only the nearest few lists, using less memory and building quickly, but requiring training on a representative sample and degrading as the data distribution drifts. Product quantisation is orthogonal compression: splitting vectors into subvectors and replacing each with a codebook entry gives sixteen- to thirty-twofold reduction, which is what makes billion-scale corpora fit in memory at the cost of approximate distances.

**Two-stage retrieval recovers what compression loses.** Retrieving a larger candidate set from a compressed index and then computing exact distances on those few hundred vectors recovers most of the lost recall for negligible additional cost. This is the same cheap-retrieval-then-expensive-scoring shape found throughout search, and at large scale it is usually the right structure — memory budget, rather than benchmark accuracy, is what actually decides the design, because latency collapses once the index no longer fits in RAM.

**Filtering is the unsolved corner.** Combining metadata constraints with approximate search has no universally correct answer. Post-filtering returns far too few results when filters are selective; pre-filtering is exact but only feasible when the filtered set is small; filtering during graph traversal preserves efficiency but can disconnect regions and collapse recall. Since selectivity varies per query, mature systems choose dynamically — and because retrofitting this is much harder than accounting for it up front, the filtering requirement should inform the index choice rather than being discovered afterwards.

**Silent degradation is the operational risk.** An approximate index returning worse results produces no error and no visible symptom; results simply become marginally less relevant. Recall drifts as parameters are tuned, compression is added for memory reasons, or data moves away from a cluster index's training distribution, and a system that launched at ninety-eight per cent can sit at eighty months later with nobody aware. The only defence is computing exact neighbours for a fixed query sample periodically and comparing — an automated check against an explicitly agreed recall target, which should have been a stated requirement rather than whatever the defaults produced.

**Prove it — interview questions**

1. **[Basic] Why is exact nearest-neighbour search impractical in high dimensions?**

   <details><summary>Model answer</summary>

   Because the structures that make low-dimensional spatial search fast degenerate as dimensionality rises. Space-partitioning trees end up examining nearly every point, so they perform no better than a brute-force scan — and a scan over ten million vectors of several hundred dimensions is billions of operations per query. This is the curse of dimensionality: distances concentrate so that nearest and farthest points become similar, and no partitioning scheme can prune much of the space.

   </details>

2. **[Basic] What does approximate mean in this context?**

   <details><summary>Model answer</summary>

   That the search returns most of the true nearest neighbours rather than all of them. Recall — the fraction of the true top-k actually returned — is typically tuned between ninety and ninety-nine per cent, and it is a query-time parameter rather than a fixed property: exploring more of the index gives higher recall at higher latency. For most applications the missing neighbours were marginal, so ninety-five per cent recall is indistinguishable from exact.

   </details>

3. **[Senior] How do graph-based and cluster-based indexes differ?**

   <details><summary>Model answer</summary>

   A graph index like HNSW builds a navigable structure where search starts at sparse long-range links and greedily hops toward the query, descending into denser local layers — giving excellent recall at low latency with no training step, at the cost of substantial memory for the edges and poor handling of deletions. A cluster-based index like IVF partitions vectors with k-means and searches only the nearest few clusters, which uses less memory and builds quickly but requires training on a representative sample, gives lower recall for the same latency, and degrades if the data distribution shifts away from what it was trained on.

   </details>

4. **[Senior] Why is filtering hard with approximate indexes?**

   <details><summary>Model answer</summary>

   Because the index's efficiency depends on structure that filtering breaks. Post-filtering retrieves the top candidates by similarity and then applies the filter, which works for weak filters but returns far too few results when the filter is selective — asking for twenty results and getting three. Pre-filtering restricts to matching vectors first and searches those exactly, which is correct but only feasible when the filtered set is small. Filtering during graph traversal keeps the search efficient but can disconnect regions of the graph, collapsing recall. None is universally right, and the correct choice depends on filter selectivity, which varies per query — so mature systems decide dynamically.

   </details>

5. **[Staff] How would you size and choose an index for ten million 768-dimensional vectors?**

   <details><summary>Model answer</summary>

   Raw vectors alone are about thirty gigabytes, which already pushes past a comfortable single node, so memory budget rather than accuracy is what decides. HNSW on raw vectors gives the best quality — around ninety-eight per cent recall at a few milliseconds — but needs roughly thirty-eight gigabytes including graph edges, so it requires a large-memory machine or sharding. If that is not affordable, product quantisation compressing each vector to around a hundred bytes brings the index to about a gigabyte, with recall dropping to the mid-eighties — which is then recovered to around ninety-five per cent by retrieving a larger candidate set and re-ranking those few hundred on exact vectors. That two-stage shape is usually the right answer at this scale, and it mirrors the cheap-retrieval-then-expensive-scoring pattern found throughout search.

   </details>

6. **[Principal] What is the most important operational discipline with approximate search?**

   <details><summary>Model answer</summary>

   Continuous recall measurement against an exact baseline, because degradation is completely silent. There is no error, no warning and no visible symptom — results simply become marginally less relevant, and a system that started at ninety-eight per cent recall can drift to eighty over months through parameter changes, compression added for memory reasons, or the data distribution moving away from what a cluster index was trained on. The only way to know is to compute true nearest neighbours for a fixed sample of queries periodically and compare what the index returns. Almost nobody does this, which is why vector search quality tends to decline invisibly after launch while attention moves to model choice and chunking. I would treat it the same way as any other silent-failure surface: an automated check, a tracked metric, and an alert on a threshold agreed when the recall target was set — because the target should have been an explicit requirement rather than whatever the defaults happened to produce.

   </details>

---

### Hybrid retrieval and reranking

*Retrieve candidates cheaply from several complementary sources, fuse their rankings, then reorder the survivors with an expensive model that could never run corpus-wide.*

**Flow:** `Lexical candidates` → `Vector candidates` → `Rank fusion` → `Reranker` → `Final results`

> **The 30-second version**  
> Retrieve candidates cheaply from complementary sources, fuse by rank since their scores are incomparable, then reorder the few survivors with a model too expensive to run corpus-wide.

**The problem**

Lexical search cannot match a paraphrase, so a user describing a problem in their own words finds nothing. Vector search cannot reliably match an exact identifier, so a user typing a model number gets its near neighbour. Each is excellent at what the other fails at, and neither alone produces a search that feels reliable.

Meanwhile the models that rank best — cross-encoders that read the query and document together — are hundreds of times too slow to score a corpus. Running one over ten million documents per query is impossible; running one over fifty candidates is trivial.

> **The architecture follows from a cost asymmetry**  
> Retrieval must be cheap because it touches everything; ranking can be expensive because it touches almost nothing. That single asymmetry produces the standard shape: broad cheap retrieval from complementary sources, fusion into one candidate list, then precise expensive scoring over a few dozen items. Every stage exists because the next one cannot afford to see the whole corpus.

**Mental model**

Think of it as a funnel with widening cost per item. Millions of documents are reduced to hundreds by cheap methods, then to tens by an expensive one, with each stage spending more per candidate than the last.

1. **Retrieval** — Several independent methods each propose candidates: lexical, vector, and sometimes popularity or category-based recall.
2. **Fusion** — Combining rankings from sources whose scores are not comparable, producing one ordered candidate list.
3. **Reranking** — A model that scores query and document jointly, far more accurate and far too slow for retrieval.
4. **Business layer** — Availability, recency, margin, personalisation — signals no text model sees, applied after relevance.
5. **Candidate depth** — How many items each stage passes on. The single most consequential tuning parameter in the pipeline.

> **Scores from different retrievers are not comparable**  
> A BM25 score of fourteen and a cosine similarity of 0.83 have no common scale, and normalising them is unreliable because BM25 scores vary unboundedly with query terms and corpus statistics. Fusing by rank rather than by score sidesteps this entirely, which is why reciprocal rank fusion is the default despite being almost embarrassingly simple.

**How it works**

**Reciprocal rank fusion, and why it works**

```text
RRF score(d) = sum over retrievers r of  1 / (k + rank_r(d))
with k typically 60

EXAMPLE  query "oom crash"
  BM25 ranks:    [A, B, C, D, ...]
  vector ranks:  [C, E, A, F, ...]

  A: 1/(60+1) + 1/(60+3) = 0.0164 + 0.0159 = 0.0323
  C: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323
  B: 1/(60+2) + 0                          = 0.0161
  E: 0        + 1/(60+2)                   = 0.0161

PROPERTIES
  uses only RANK, so incomparable scores never meet
  documents ranked well by BOTH retrievers rise to the top
  a document ranked first by one retriever still scores
    reasonably even if the other missed it entirely
  k dampens the influence of the very top ranks, so one
    retriever cannot dominate

IT IS HARD TO BEAT WITHOUT TRAINING DATA, which is why it
remains the default despite its simplicity.
```

1. **Fuse by rank, not by score** — Score normalisation across retrievers is fragile because the scales are unrelated and unbounded. Rank fusion is robust and needs no tuning.
2. **Set candidate depth deliberately** — Too shallow and the reranker never sees the right document; too deep and reranking cost explodes. Fifty to two hundred is typical, and it should be measured rather than guessed.
3. **Put the expensive model last** — A cross-encoder is orders of magnitude more accurate and orders of magnitude slower. It belongs over candidates, never over a corpus.
4. **Keep business signals after relevance** — Availability, recency and margin should adjust a relevance-ordered list rather than being mixed into retrieval, so the two can be reasoned about and tuned separately.
5. **Measure each stage independently** — Retrieval recall — did the right document reach the candidate set — is a different question from ranking quality. Conflating them makes diagnosis impossible.
6. **Add retrievers for coverage, not for accuracy** — A third retriever earns its place if it finds documents the others miss, not if it ranks the same documents slightly better.

**Why measuring stages separately matters**

```text
COMPLAINT  "the right document is not in the results"

TWO VERY DIFFERENT CAUSES
  RETRIEVAL FAILURE
    the document was never in the candidate set
    -> reranking cannot fix it; it never saw it
    -> fix: analysis, another retriever, deeper candidates
  RANKING FAILURE
    the document was retrieved but ranked 47th
    -> retrieval is fine; the reranker or signals are wrong
    -> fix: reranker, boosts, business signals

MEASUREMENT
  retrieval recall@k = of the known-relevant documents,
    what fraction appear in the top k candidates?
    -> if this is 0.6, no amount of ranking work helps
       the missing 40%
  ranking quality (NDCG, MRR) measured ON the retrieved set
    -> isolates ranking from retrieval

TEAMS ROUTINELY TUNE THE RERANKER TO FIX A RETRIEVAL
PROBLEM, and get nowhere, because the document was never
a candidate.
```

> **Candidate depth is the cheapest quality lever**  
> Increasing retrieval depth from fifty to two hundred candidates often improves final quality more than swapping the reranker, because it raises the ceiling on what ranking can achieve. The cost is linear in reranker latency, which is usually affordable — and it should be measured as a curve, since the benefit typically plateaus well before the cost does.

**Worked example**

A documentation search pipeline, showing what each stage contributes.

**Four stages, each with a distinct job**

```text
QUERY  "app keeps dying when memory fills up"

STAGE 1  RETRIEVAL (parallel, ~10ms)
  BM25 over titles and body
    -> matches "memory", "fills"
    -> misses the relevant doc titled "Handling OOM errors"
  vector search over chunks
    -> matches the OOM document semantically  <- the win
  each returns top 100

STAGE 2  FUSION (~0ms)
  reciprocal rank fusion -> 150 unique candidates
  documents found by both rise; documents found by one
  still survive

STAGE 3  RERANKING (~80ms for 150 candidates)
  cross-encoder scores (query, chunk) pairs jointly
  -> understands that "app keeps dying" and "process
     terminated by the OOM killer" describe the same thing
  -> reorders substantially; keeps top 20

STAGE 4  BUSINESS SIGNALS (~0ms)
  boost documents matching the user's product version
  boost recently updated pages
  demote deprecated content
  -> final top 10

TOTAL ~90ms, dominated by the reranker

WHAT EACH STAGE CONTRIBUTED
  vector retrieval: found the document at all
  fusion: kept it despite BM25 missing it
  reranker: moved it from rank 40 to rank 2
  business signals: ensured the right version
```

| Metric | Value | Note |
|---|---|---|
| Retrieval | ~10 ms | 2 sources, 200 candidates |
| Fusion | ~0 ms | 150 unique |
| Rerank | ~80 ms | **dominates latency** |
| Total | ~90 ms | within budget |

> **Each stage fixes a failure the others cannot**  
> The vector retriever is what made the document reachable, fusion is what kept it despite the lexical retriever missing it, the reranker is what moved it from the fortieth position to the second, and the business layer is what ensured the version was right. Remove any one and the result degrades in a distinct, identifiable way — which is exactly why the pipeline has four stages rather than one clever model.

**When to use it**

- **Any search where queries vary between precise and descriptive**, which is most user-facing search.
- **Retrieval-augmented generation**, where the quality of retrieved passages bounds everything downstream.
- **Product and content search**, where relevance must be combined with commercial and freshness signals.
- **When one retriever demonstrably misses a class of query**, which is the signal that a second is warranted.
- **Where a reranker is affordable over candidates** but not over the corpus, which is essentially always.

**When to avoid it**

- **Do not add retrievers that find the same documents**; a retriever earns its place through coverage, not marginal ranking.
- **Do not normalise and add scores across retrievers**, which is fragile because the scales are unrelated.
- **Do not run a cross-encoder over the corpus**, which is orders of magnitude too slow.
- **Do not mix business signals into retrieval**, which makes relevance and commerce impossible to tune separately.
- **Do not tune the reranker to fix a retrieval gap**, which cannot work because the document was never a candidate.

**Advantages**

- **Covers complementary failure modes**, so precise and descriptive queries both work.
- **Enables an expensive, accurate model** by restricting it to a small candidate set.
- **Rank fusion needs no training data**, so the pipeline works from day one.
- **Stages are independently measurable and tunable**, which makes diagnosis tractable.
- **Business signals apply cleanly** after relevance is established, keeping the two concerns separable.

**Disadvantages**

- **More moving parts**: two or more indexes, a fusion step, a reranking model, a signals layer.
- **Latency is dominated by reranking**, and scales with candidate depth.
- **Several indexes to keep in sync**, each with its own freshness and rebuild characteristics.
- **Tuning surface is large** — depth, fusion parameters, reranker choice, signal weights.
- **Harder to explain** why a particular result appeared, since four stages contributed.
- **Cost per query is substantially higher** than single-retriever search.

**Trade-offs**

**Where quality comes from, and what it costs**

| Stage | Quality contribution | Latency | Tuning difficulty |
|---|---|---|---|
| Lexical retrieval | Exact terms, identifiers | Low | Analysis and boosts |
| Vector retrieval | Paraphrase, concepts | Low | Chunking and model |
| Rank fusion | Coverage of both | Negligible | Almost none |
| Cross-encoder rerank | Largest single improvement | High | Model choice; depth |
| Business signals | User satisfaction beyond relevance | Negligible | Weights; needs measurement |

> **Framing the architecture**  
> “Two retrievers in parallel because they fail on different queries, fused by rank since their scores are incomparable, then a cross-encoder over the top hundred and fifty — which is far more accurate and far too slow to run corpus-wide. Business signals last, so relevance and commercial priorities stay separately tunable. Latency is dominated by reranking and scales with candidate depth, which is the main dial.”

**How it fails**

**Pipeline failures**

| Symptom | Likely stage | Fix |
|---|---|---|
| Relevant document never appears | Retrieval — it was not a candidate | Deeper candidates; add a retriever; fix analysis or chunking |
| Relevant document retrieved but ranked low | Reranking or signals | Better reranker; adjust boosts; check depth |
| Latency exceeds budget | Reranking over too many candidates | Reduce depth; smaller model; batch scoring |
| One retriever dominates results | Score-based fusion with mismatched scales | Rank-based fusion |
| Quality improved offline, not in production | Evaluation set unrepresentative of real queries | Sample from real query logs; measure per query class |
| Adding a retriever changed nothing | It finds the same documents as an existing one | Measure unique contribution before adding |
| Cannot diagnose complaints | Stages measured only end to end | Measure retrieval recall and ranking quality separately |

**Limits**

> **Operating parameters**
>
> - **Candidate depth**: typically 50–200 per retriever; the benefit plateaus before the cost does, so measure the curve.
> - **RRF constant k ≈ 60**, which dampens the influence of the very top ranks and is rarely worth tuning.
> - **Cross-encoder latency** scales linearly with candidates — this dominates the pipeline budget.
> - **Retrieval recall@k** is the ceiling on final quality: nothing downstream can recover a document that was never a candidate.
> - **Add retrievers for coverage**, measured as the fraction of relevant documents only that retriever finds.

**Alternatives**

| Architecture | Strength | Weakness |
|---|---|---|
| Single lexical retriever | Simple, fast, explainable | Fails on paraphrase |
| Single vector retriever | Semantic | Fails on exact terms |
| Hybrid retrieval only | Covers both | No precise final ranking |
| Hybrid + cross-encoder rerank | Best quality | Latency; complexity |
| Learning to rank over features | Learns from behaviour | Needs training data |
| End-to-end neural retrieval | Single model | Expensive; loses exactness |

Learning to rank is the natural evolution of the final stage once click and conversion data exist: rather than hand-weighting business signals over a reranker's output, a model learns the combination from behaviour. It does not replace the retrieval or reranking stages, which exist for cost reasons that no amount of training data changes.

**In real systems**

- **Web search** has used multi-stage retrieval and ranking for decades, for exactly the cost reasons described — cheap candidate generation, expensive final ranking.
- **Reciprocal rank fusion** is the default hybrid combination in modern search engines because it is robust, needs no training, and handles incomparable scores.
- **Retrieval-augmented generation pipelines** converged on hybrid retrieval plus cross-encoder reranking, since retrieval quality bounds everything the language model can do.
- **Cross-encoder rerankers** are widely deployed over the top few dozen candidates, reflecting the same asymmetry that drives the whole architecture.
- **E-commerce search** applies commercial signals after relevance ranking, keeping merchandising and relevance separately owned and tunable.

**Common mistakes**

- **Tuning the reranker** to fix a retrieval failure.
- **Normalising and adding incomparable scores** rather than fusing by rank.
- **Candidate depth set arbitrarily**, capping quality without anyone noticing.
- **Adding retrievers that duplicate coverage**, increasing cost for no gain.
- **Mixing business signals into retrieval**, making both untunable.
- **Measuring only end-to-end quality**, so failures cannot be attributed to a stage.
- **Evaluating on a curated query set** unrepresentative of real traffic.

**The staff-level view**

The value of the staged architecture is as much diagnostic as it is qualitative: it makes it possible to tell which part of search is failing.

- **Measure retrieval recall separately from ranking quality.** Teams routinely tune rerankers to fix retrieval gaps and make no progress, because the document was never a candidate — and only stage-separated measurement reveals that.
- **Treat candidate depth as a primary dial.** Increasing it often improves results more than changing the reranker, and its cost is predictable and linear.
- **Justify each retriever by unique coverage.** A second source earns its place by finding documents the first misses, not by ranking the same documents slightly differently.
- **Keep business signals in their own stage**, so relevance engineering and commercial tuning do not entangle — they are usually owned by different people with different objectives.
- **Evaluate on real query distributions.** Offline improvements on a curated set routinely fail to reproduce because the evaluation queries were not representative of what users actually type.

**Go deeper**

Lexical and vector retrieval fail on opposite query types — one cannot match paraphrase, the other cannot match exact identifiers — so running both and combining covers each blind spot. Their scores are not comparable, so fusion works on rank rather than score: reciprocal rank fusion sums one over a constant plus each retriever's rank, which needs no tuning, requires no training data and is hard to beat.

A cross-encoder that reads query and document together ranks far better than any retrieval method and is orders of magnitude slower, so it runs over the fused candidate set rather than the corpus. That cost asymmetry — retrieval touches everything so it must be cheap, ranking touches almost nothing so it can be expensive — is what produces the staged architecture. Business signals such as availability, recency and version come last, so relevance and commercial priorities remain separately tunable.

The diagnostic value is as important as the quality. Measure retrieval recall — did the relevant document reach the candidate set — separately from ranking quality on the retrieved set, because they are different failures with different fixes and teams routinely tune rerankers to solve retrieval gaps. Candidate depth is usually the highest-return dial, since it raises the ceiling on everything downstream at a predictable linear cost.

The multi-stage retrieval and ranking architecture is not an accumulation of components but a direct consequence of a cost asymmetry: retrieval must examine everything and therefore must be cheap, while ranking examines almost nothing and can therefore be expensive.

**Complementary retrieval.** Lexical matching handles exact identifiers, error codes, rare proper nouns and anything the user typed precisely; it cannot match a paraphrase. Vector similarity handles conceptual and reworded queries; it cannot reliably distinguish two model numbers differing by one character. Running both in parallel covers each blind spot, and the asymmetry of consequences matters — a user describing a problem vaguely tolerates approximate results, while a user typing an exact code does not. A third retriever is justified only by unique coverage, measured as the proportion of relevant documents it alone finds, never by marginally better ordering of documents the others already return.

**Rank fusion rather than score fusion.** BM25 scores are unbounded and depend on query terms, corpus statistics and document lengths; cosine similarities are bounded and mean something different. Normalising across them is fragile and query-dependent. Reciprocal rank fusion uses only positions — summing one over a constant plus each rank, with the constant around sixty — so incomparable scales never interact. Documents ranked well by both retrievers rise, documents found by only one still survive, and no retriever can dominate. It requires no training data, which is why it remains the default despite its simplicity.

**Reranking is where most of the quality comes from.** A cross-encoder processes query and document jointly rather than embedding them separately, which is substantially more accurate and hundreds of times more expensive per item. Over ten million documents it is impossible; over a hundred and fifty candidates it is tens of milliseconds. That single fact determines the architecture's shape, and it means the reranker's effectiveness is bounded by what retrieval delivered — it can reorder, but it cannot introduce.

**Stage separation is a diagnostic asset.** A complaint that the right document is missing has two entirely different causes with different fixes: it was never retrieved, in which case reranking is irrelevant and the problem lies in analysis, chunking, retriever coverage or candidate depth; or it was retrieved and ranked poorly, in which case retrieval is fine. Measuring retrieval recall at depth separately from ranking quality on the retrieved set is what distinguishes them, and without that separation teams spend weeks tuning rerankers against retrieval failures and make no progress.

**Candidate depth is the most underrated dial.** Increasing it from fifty to two hundred raises the ceiling on everything downstream and frequently improves final quality more than changing the reranker, at a cost that is linear and predictable in reranking latency. It should be measured as a curve, since the benefit typically plateaus well before the cost does, and it is routinely set arbitrarily at the start and never revisited.

**Keep business signals in their own stage.** Availability, recency, margin, version compatibility and personalisation are invisible to any text model and frequently matter more to user satisfaction than relevance. Applying them after relevance ranking keeps two concerns separable that are usually owned by different people with different objectives — merchandising and relevance engineering — and once click and conversion data exist, this stage is the natural place for a learned model, which does not replace retrieval or reranking because those exist for cost reasons that training data does not change.

**Prove it — interview questions**

1. **[Basic] Why use more than one retriever?**

   <details><summary>Model answer</summary>

   Because lexical and vector retrieval fail on different queries. Lexical search cannot match a paraphrase, so a user describing a problem in their own words finds nothing; vector search cannot reliably match an exact identifier, so a user typing a model number gets its near neighbour. Running both and combining the results covers each blind spot, and the second failure matters especially because those are the queries where users are most certain what they want.

   </details>

2. **[Basic] Why fuse by rank rather than by score?**

   <details><summary>Model answer</summary>

   Because scores from different retrievers have no common scale. A BM25 score depends on query terms, corpus statistics and document lengths and is unbounded; a cosine similarity is bounded between minus one and one. Normalising them is fragile and query-dependent. Reciprocal rank fusion uses only each document's position in each retriever's list, so incomparable scales never meet — documents ranked well by both rise to the top, while a document found by only one still survives. It needs no tuning and is hard to beat without training data.

   </details>

3. **[Senior] Why is reranking done over candidates rather than the corpus?**

   <details><summary>Model answer</summary>

   Because of a cost asymmetry. A cross-encoder reads the query and document together and is far more accurate than any retrieval method, but it costs orders of magnitude more per item — scoring ten million documents per query is impossible while scoring a hundred and fifty is a few tens of milliseconds. So retrieval must be cheap because it touches everything, and ranking can be expensive because it touches almost nothing. That asymmetry is what produces the multi-stage architecture rather than a single model.

   </details>

4. **[Senior] Why measure retrieval recall separately from ranking quality?**

   <details><summary>Model answer</summary>

   Because they are different failures with different fixes, and end-to-end measurement cannot distinguish them. If the relevant document was never in the candidate set, no amount of reranking helps — the model never saw it — and the fix is deeper candidates, another retriever, or better analysis and chunking. If it was retrieved but ranked fortieth, retrieval is fine and the reranker or the signals are wrong. Teams routinely spend weeks tuning a reranker to fix a retrieval gap and make no progress, and only stage-separated measurement reveals why.

   </details>

5. **[Staff] Which tuning lever would you reach for first to improve a hybrid pipeline?**

   <details><summary>Model answer</summary>

   Candidate depth, because it raises the ceiling on everything downstream and its cost is predictable. Increasing retrieval depth from fifty to two hundred often improves final quality more than swapping the reranker, since the reranker can only reorder what it receives — and the cost is linear in reranking latency, which is usually affordable. I would measure it as a curve rather than picking a number, because the benefit typically plateaus well before the cost does. After that, I would look at whether a retriever is contributing unique coverage — measured as the fraction of relevant documents only it finds — because adding a source that returns the same documents increases cost with no gain, and that is a common way pipelines become expensive without becoming better.

   </details>

6. **[Principal] How would you structure ownership and evaluation of a search pipeline?**

   <details><summary>Model answer</summary>

   By making the stages separately owned and separately measured, because they fail differently and are tuned by different skills. Retrieval quality — recall at depth — is an engineering concern involving analysis, chunking, embedding models and index configuration. Ranking quality is a modelling concern measured on the retrieved set. Business signals are usually a commercial concern, owned by merchandising or content teams who understand priorities engineering does not, which is exactly why they belong in their own stage rather than mixed into retrieval where they cannot be tuned independently. On evaluation, the discipline that matters most is sampling queries from real traffic rather than curating a set: offline improvements on a hand-picked evaluation set routinely fail to reproduce, because the curated queries are not representative of the distribution users actually produce — and per-query-class measurement, separating precise identifier lookups from descriptive questions, is what reveals that a change helped one class while quietly harming another.

   </details>

---
