# Drill Gym — Small reps, serious progress

> 150 timed scenarios with constraints. Write your answer first, take the hints one at a time, then open the reference reasoning, trade-offs, staff-level view and follow-ups.

## Contents

- **Foundational** (15): [A single viral alias](#a-single-viral-alias) · [Negative cache hides a new alias](#negative-cache-hides-a-new-alias) · [Two writers claim one custom alias](#two-writers-claim-one-custom-alias) · [The enterprise tenant hot key](#the-enterprise-tenant-hot-key) · [Limiter store is unreachable](#limiter-store-is-unreachable) · [Double spending across regions](#double-spending-across-regions) · [Rebalancing a warm cache](#rebalancing-a-warm-cache) · [Mass expiry stampede](#mass-expiry-stampede) · [Stale writer resurrects a value](#stale-writer-resurrects-a-value) · [Sequence space is exhausted](#sequence-space-is-exhausted) · [Clock moves backwards](#clock-moves-backwards) · [A worker ID is reused too soon](#a-worker-id-is-reused-too-soon) · [Midnight job cliff](#midnight-job-cliff) · [Worker dies after sending a report](#worker-dies-after-sending-a-report) · [Cancel races with execution](#cancel-races-with-execution)
- **Social** (15): [Celebrity fanout explosion](#celebrity-fanout-explosion) · [Feed contains a deleted post](#feed-contains-a-deleted-post) · [Pagination repeats ranked items](#pagination-repeats-ranked-items) · [A new thumbnail size goes viral](#a-new-thumbnail-size-goes-viral) · [Incomplete image marked ready](#incomplete-image-marked-ready) · [Private image survives album removal](#private-image-survives-album-removal) · [A celebrity has a huge follower partition](#a-celebrity-has-a-huge-follower-partition) · [Follower counts become negative](#follower-counts-become-negative) · [Blocked user remains in a stale projection](#blocked-user-remains-in-a-stale-projection) · [One event dominates replies](#one-event-dominates-replies) · [Moderation disappears after an edit retry](#moderation-disappears-after-an-edit-retry) · [Deleting a parent orphans discussion](#deleting-a-parent-orphans-discussion) · [Viewer list overwhelms one story](#viewer-list-overwhelms-one-story) · [Expired story remains playable](#expired-story-remains-playable) · [Audience changes during viewing](#audience-changes-during-viewing)
- **Messaging** (15): [One busy group overloads its owner](#one-busy-group-overloads-its-owner) · [Messages vanish after reconnect](#messages-vanish-after-reconnect) · [A removed member receives delayed messages](#a-removed-member-receives-delayed-messages) · [Marketing consumes transactional capacity](#marketing-consumes-transactional-capacity) · [SMS provider accepted an ambiguous request](#sms-provider-accepted-an-ambiguous-request) · [Opt-out races with a queued campaign](#opt-out-races-with-a-queued-campaign) · [Large attachments dominate ingress](#large-attachments-dominate-ingress) · [Accepted mail disappears during a crash](#accepted-mail-disappears-during-a-crash) · [Folder moves race with flag updates](#folder-moves-race-with-flag-updates) · [One slow endpoint consumes workers](#one-slow-endpoint-consumes-workers) · [Receiver processed the event but returned 500](#receiver-processed-the-event-but-returned-500) · [A newer update arrives before an older retry](#a-newer-update-arrives-before-an-older-retry) · [Consumer fanout multiplies bandwidth](#consumer-fanout-multiplies-bandwidth) · [Consumer is behind retention](#consumer-is-behind-retention) · [A database sink duplicates increments](#a-database-sink-duplicates-increments)
- **Media** (15): [A viral upload is cold at every edge](#a-viral-upload-is-cold-at-every-edge) · [A manifest advertises missing segments](#a-manifest-advertises-missing-segments) · [A transcode worker retries after lease loss](#a-transcode-worker-retries-after-lease-loss) · [Final-minute sports traffic surge](#final-minute-sports-traffic-surge) · [Encoder restart resets timestamps](#encoder-restart-resets-timestamps) · [Two ingests believe they own a stream](#two-ingests-believe-they-own-a-stream) · [A new album creates a cache cliff](#a-new-album-creates-a-cache-cliff) · [Rights change but cached playback remains authorized](#rights-change-but-cached-playback-remains-authorized) · [Two devices overwrite playlist edits](#two-devices-overwrite-playlist-edits) · [Large town hall exhausts SFU egress](#large-town-hall-exhausts-sfu-egress) · [UDP is blocked on a corporate network](#udp-is-blocked-on-a-corporate-network) · [A removed participant reconnects with an old token](#a-removed-participant-reconnects-with-an-old-token) · [Polling every feed wastes most requests](#polling-every-feed-wastes-most-requests) · [A publisher reuses an episode GUID](#a-publisher-reuses-an-episode-guid) · [Progress regresses after offline sync](#progress-regresses-after-offline-sync)
- **Storage** (15): [A shared folder has a million small files](#a-shared-folder-has-a-million-small-files) · [A partial upload becomes the current version](#a-partial-upload-becomes-the-current-version) · [Two offline devices modify the same file](#two-offline-devices-modify-the-same-file) · [Repair competes with serving traffic](#repair-competes-with-serving-traffic) · [Silent corruption passes through replication](#silent-corruption-passes-through-replication) · [Concurrent overwrites return mixed fragments](#concurrent-overwrites-return-mixed-fragments) · [Restore bandwidth misses the promised RTO](#restore-bandwidth-misses-the-promised-rto) · [Backup is green but missing a log segment](#backup-is-green-but-missing-a-log-segment) · [Retention cleanup removes an active restore dependency](#retention-cleanup-removes-an-active-restore-dependency) · [One shared blob has a hot reference counter](#one-shared-blob-has-a-hot-reference-counter) · [Garbage collector races with a new reference](#garbage-collector-races-with-a-new-reference) · [A guessed digest reveals private content](#a-guessed-digest-reveals-private-content) · [Small files overwhelm namespace memory](#small-files-overwhelm-namespace-memory) · [A stale client writes after lease transfer](#a-stale-client-writes-after-lease-transfer) · [Rename crosses metadata shards](#rename-crosses-metadata-shards)
- **Commerce** (15): [A sale overwhelms inventory reservation](#a-sale-overwhelms-inventory-reservation) · [Payment succeeded but the order timed out](#payment-succeeded-but-the-order-timed-out) · [A reservation expires as payment completes](#a-reservation-expires-as-payment-completes) · [Hot SKU saturates one row lock](#hot-sku-saturates-one-row-lock) · [Expiry and checkout both decrement reservations](#expiry-and-checkout-both-decrement-reservations) · [Warehouse event is delivered twice](#warehouse-event-is-delivered-twice) · [A settlement account becomes a hot row](#a-settlement-account-becomes-a-hot-row) · [Retry posts the same transfer twice](#retry-posts-the-same-transfer-twice) · [Currency rounding breaks ledger balance](#currency-rounding-breaks-ledger-balance) · [Seat-map refresh melts the backend](#seat-map-refresh-melts-the-backend) · [A timer releases a newly confirmed seat](#a-timer-releases-a-newly-confirmed-seat) · [Two groups partially acquire adjacent seats](#two-groups-partially-acquire-adjacent-seats) · [A slow seller blocks every order item](#a-slow-seller-blocks-every-order-item) · [Shipment callback moves an item backwards](#shipment-callback-moves-an-item-backwards) · [Refund and payout race](#refund-and-payout-race)
- **Real Time** (15): [Airport arrivals overload one map cell](#airport-arrivals-overload-one-map-cell) · [Stale locations produce impossible pickups](#stale-locations-produce-impossible-pickups) · [Two riders get the same driver](#two-riders-get-the-same-driver) · [A giant document has years of operations](#a-giant-document-has-years-of-operations) · [Concurrent insertions diverge across clients](#concurrent-insertions-diverge-across-clients) · [Revoked editor uploads offline operations](#revoked-editor-uploads-offline-operations) · [Reconnect storm floods heartbeats](#reconnect-storm-floods-heartbeats) · [User remains online after a silent disconnect](#user-remains-online-after-a-silent-disconnect) · [Old disconnect clears a new session](#old-disconnect-clears-a-new-session) · [One global sorted set runs out of headroom](#one-global-sorted-set-runs-out-of-headroom) · [Replayed score event doubles a win](#replayed-score-event-doubles-a-win) · [Late scores cross a season boundary](#late-scores-cross-a-season-boundary) · [A map zoom returns every device](#a-map-zoom-returns-every-device) · [GPS jitter creates repeated enter-exit alerts](#gps-jitter-creates-repeated-enter-exit-alerts) · [Offline backlog overwrites the latest position](#offline-backlog-overwrites-the-latest-position)
- **Search** (15): [Common query fans out to every shard](#common-query-fans-out-to-every-shard) · [Crawler falls into an infinite calendar](#crawler-falls-into-an-infinite-calendar) · [Deleted pages remain in search results](#deleted-pages-remain-in-search-results) · [Every keystroke becomes a server request](#every-keystroke-becomes-a-server-request) · [A blocked suggestion persists in cached lists](#a-blocked-suggestion-persists-in-cached-lists) · [Personalized cache leaks another user’s history](#personalized-cache-leaks-another-users-history) · [High-cardinality facets blow query memory](#high-cardinality-facets-blow-query-memory) · [Search advertises yesterday’s price](#search-advertises-yesterdays-price) · [Backfill overwrites a newer product update](#backfill-overwrites-a-newer-product-update) · [One incident causes a logging storm](#one-incident-causes-a-logging-storm) · [Mapping explosion takes down indexing](#mapping-explosion-takes-down-indexing) · [Late logs fall into an already-expired partition](#late-logs-fall-into-an-already-expired-partition) · [Restrictive filters return too few neighbors](#restrictive-filters-return-too-few-neighbors) · [Embedding upgrade mixes incompatible spaces](#embedding-upgrade-mixes-incompatible-spaces) · [Private document appears in retrieved context](#private-document-appears-in-retrieved-context)
- **Infrastructure** (15): [Failover overwhelms the healthy region](#failover-overwhelms-the-healthy-region) · [Health checks remove every backend](#health-checks-remove-every-backend) · [Draining drops long-lived connections](#draining-drops-long-lived-connections) · [A tiny zone receives a massive query flood](#a-tiny-zone-receives-a-massive-query-flood) · [Bad zone version reaches every region](#bad-zone-version-reaches-every-region) · [DNS rollback appears ineffective](#dns-rollback-appears-ineffective) · [Request IDs create millions of new series](#request-ids-create-millions-of-new-series) · [Alert evaluator fails silently](#alert-evaluator-fails-silently) · [Missing samples are treated as zero](#missing-samples-are-treated-as-zero) · [Remote evaluation adds a dependency to every request](#remote-evaluation-adds-a-dependency-to-every-request) · [Bad rollout config reaches all clients](#bad-rollout-config-reaches-all-clients) · [Users jump between rollout cohorts](#users-jump-between-rollout-cohorts) · [Deployment generates a watch storm](#deployment-generates-a-watch-storm) · [Watcher resumes from a compacted revision](#watcher-resumes-from-a-compacted-revision) · [Two leaders push conflicting configuration](#two-leaders-push-conflicting-configuration)
- **Advanced** (15): [A hot tenant dominates one partition](#a-hot-tenant-dominates-one-partition) · [Async failover loses acknowledged recent writes](#async-failover-loses-acknowledged-recent-writes) · [Concurrent regions overwrite each other](#concurrent-regions-overwrite-each-other) · [Long histories make every decision expensive](#long-histories-make-every-decision-expensive) · [Deployment makes replay nondeterministic](#deployment-makes-replay-nondeterministic) · [Compensation itself fails](#compensation-itself-fails) · [A global campaign hot-spots budget checks](#a-global-campaign-hot-spots-budget-checks) · [Delayed impressions overshoot the budget](#delayed-impressions-overshoot-the-budget) · [Duplicate impression events bill twice](#duplicate-impression-events-bill-twice) · [One slow feature source consumes the latency budget](#one-slow-feature-source-consumes-the-latency-budget) · [Training and serving use different feature definitions](#training-and-serving-use-different-feature-definitions) · [Experiment exposure is counted before delivery](#experiment-exposure-is-counted-before-delivery) · [One shared IP becomes a hot aggregation key](#one-shared-ip-becomes-a-hot-aggregation-key) · [Processor restarts and forgets window state](#processor-restarts-and-forgets-window-state) · [Late events change a decision after funds moved](#late-events-change-a-decision-after-funds-moved)

## Foundational

### A single viral alias

⏱ 12 min

**Scenario:** URL shortener: One alias suddenly receives 200,000 redirects per second.

**Constraints:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms. Preserve: Custom aliases and expiry; Redirect without synchronous analytics.

<details><summary>Hints</summary>

1. Count requests per key, not just total QPS.
2. A redirect target rarely changes.

</details>

<details><summary>Reference answer</summary>

**Answer:** Cache the alias at edge and process level; coalesce misses and retain a stale value during refresh. Apply versioned invalidation for edits and keep click ingestion asynchronous.

**Trade-offs:** Long edge TTLs improve redirect latency but delay target changes; analytics can be eventually consistent while alias ownership cannot.

**Staff-level view:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

**Follow-ups:**

- Now combine this scenario with: An unavailable alias was cached as missing before its owner created it. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two create requests both observe that /launch is available.

</details>

---

### Negative cache hides a new alias

⏱ 10 min

**Scenario:** URL shortener: An unavailable alias was cached as missing before its owner created it.

**Constraints:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms. Preserve: Custom aliases and expiry; Redirect without synchronous analytics.

<details><summary>Hints</summary>

1. Negative answers need their own TTL.
2. Creation and invalidation race.

</details>

<details><summary>Reference answer</summary>

**Answer:** Invalidate the negative entry after the durable insert, use a short negative TTL, and optionally bypass cache for the owner after creation. Track negative-hit rate and creation-to-visibility delay.

**Trade-offs:** Long edge TTLs improve redirect latency but delay target changes; analytics can be eventually consistent while alias ownership cannot.

**Staff-level view:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

**Follow-ups:**

- Now combine this scenario with: One alias suddenly receives 200,000 redirects per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two create requests both observe that /launch is available.

</details>

---

### Two writers claim one custom alias

⏱ 15 min

**Scenario:** URL shortener: Two create requests both observe that /launch is available.

**Constraints:** 10M daily users; 20 redirects each; 100:1 read/write ratio; redirect p99 target 80 ms. Preserve: Custom aliases and expiry; Redirect without synchronous analytics.

<details><summary>Hints</summary>

1. A read-before-write check is insufficient.
2. Find the storage uniqueness boundary.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a unique primary key or conditional insert for the alias. Return conflict for the losing request; bind a client idempotency key to the winning request payload and result.

**Trade-offs:** Long edge TTLs improve redirect latency but delay target changes; analytics can be eventually consistent while alias ownership cannot.

**Staff-level view:** Model abuse takedown propagation separately from ordinary edits; establish a maximum revocation delay and test cache invalidation under partial regional failure.

**Follow-ups:**

- Now combine this scenario with: One alias suddenly receives 200,000 redirects per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: One alias suddenly receives 200,000 redirects per second.

</details>

---

### The enterprise tenant hot key

⏱ 12 min

**Scenario:** Distributed rate limiter: A single tenant sends half of all requests to one token bucket.

**Constraints:** 100M decisions/day; 1M tenants; 2 ms regional decision budget. Preserve: Per-tenant limits; Weighted expensive operations.

<details><summary>Hints</summary>

1. Strict shared counters serialize work.
2. Can budgets be leased to workers?

</details>

<details><summary>Reference answer</summary>

**Answer:** Allocate bounded token leases from a regional owner to gateways. A lease bounds overshoot and reduces central operations; reclaim only with an expiry rule that prevents double spending.

**Trade-offs:** A globally exact limit increases coordination latency; distributed leases buy availability with a calculable overshoot bound.

**Staff-level view:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

**Follow-ups:**

- Now combine this scenario with: The bucket datastore times out while the payment API remains healthy. How must the solution change?
- Which part of your solution protects the invariant in this related case: A tenant uses the full global allowance simultaneously in three regions.

</details>

---

### Limiter store is unreachable

⏱ 10 min

**Scenario:** Distributed rate limiter: The bucket datastore times out while the payment API remains healthy.

**Constraints:** 100M decisions/day; 1M tenants; 2 ms regional decision budget. Preserve: Per-tenant limits; Weighted expensive operations.

<details><summary>Hints</summary>

1. Failure policy is a product decision.
2. Different routes have different cost and risk.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use conservative local emergency budgets for essential reads; reject expensive writes if their budget cannot be established. Avoid unlimited fail-open traffic and record every fallback decision.

**Trade-offs:** A globally exact limit increases coordination latency; distributed leases buy availability with a calculable overshoot bound.

**Staff-level view:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

**Follow-ups:**

- Now combine this scenario with: A single tenant sends half of all requests to one token bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: A tenant uses the full global allowance simultaneously in three regions.

</details>

---

### Double spending across regions

⏱ 15 min

**Scenario:** Distributed rate limiter: A tenant uses the full global allowance simultaneously in three regions.

**Constraints:** 100M decisions/day; 1M tenants; 2 ms regional decision budget. Preserve: Per-tenant limits; Weighted expensive operations.

<details><summary>Hints</summary>

1. Local atomicity is not global atomicity.
2. Choose a bounded error or coordination.

</details>

<details><summary>Reference answer</summary>

**Answer:** Partition the allowance into regional allocations with a single transfer authority, or route strict decisions to one owner. Quantify the maximum excess caused by outstanding leases before promising a global cap.

**Trade-offs:** A globally exact limit increases coordination latency; distributed leases buy availability with a calculable overshoot bound.

**Staff-level view:** Define fairness during regional evacuation and prevent one tenant from monopolizing the emergency budget. Roll out policy changes with explicit version semantics.

**Follow-ups:**

- Now combine this scenario with: A single tenant sends half of all requests to one token bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: A single tenant sends half of all requests to one token bucket.

</details>

---

### Rebalancing a warm cache

⏱ 12 min

**Scenario:** Distributed cache: Adding four nodes remaps keys and doubles database reads.

**Constraints:** 2B GETs/day; 200GB hot data; 1KB median values. Preserve: TTL and eviction; Horizontal redistribution.

<details><summary>Hints</summary>

1. Measure moved keys and miss traffic.
2. Warmth is operational state.

</details>

<details><summary>Reference answer</summary>

**Answer:** Move small hash ranges gradually, read old owners during handoff, and throttle warmup below database headroom. Use virtual nodes and request coalescing; adding nodes all at once can worsen overload.

**Trade-offs:** Aggressive eviction lowers memory cost but creates source traffic; serving stale data is acceptable only for explicitly chosen fields.

**Staff-level view:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

**Follow-ups:**

- Now combine this scenario with: A batch import gives ten million popular keys the same expiry. How must the solution change?
- Which part of your solution protects the invariant in this related case: A delayed loader writes version 7 after an update invalidated version 8.

</details>

---

### Mass expiry stampede

⏱ 10 min

**Scenario:** Distributed cache: A batch import gives ten million popular keys the same expiry.

**Constraints:** 2B GETs/day; 200GB hot data; 1KB median values. Preserve: TTL and eviction; Horizontal redistribution.

<details><summary>Hints</summary>

1. TTL correlation creates a burst.
2. Only one caller needs to refill a key.

</details>

<details><summary>Reference answer</summary>

**Answer:** Add TTL jitter, single-flight refresh, and bounded stale-while-revalidate for permitted data. Gate refill concurrency against source capacity and shed cacheable traffic before overwhelming the database.

**Trade-offs:** Aggressive eviction lowers memory cost but creates source traffic; serving stale data is acceptable only for explicitly chosen fields.

**Staff-level view:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

**Follow-ups:**

- Now combine this scenario with: Adding four nodes remaps keys and doubles database reads. How must the solution change?
- Which part of your solution protects the invariant in this related case: A delayed loader writes version 7 after an update invalidated version 8.

</details>

---

### Stale writer resurrects a value

⏱ 15 min

**Scenario:** Distributed cache: A delayed loader writes version 7 after an update invalidated version 8.

**Constraints:** 2B GETs/day; 200GB hot data; 1KB median values. Preserve: TTL and eviction; Horizontal redistribution.

<details><summary>Hints</summary>

1. Deletion alone carries no version.
2. What rejects an old refill?

</details>

<details><summary>Reference answer</summary>

**Answer:** Store versioned values and use compare-and-set against an invalidation watermark or versioned namespace. If that complexity is excessive, bound staleness explicitly and route read-after-write requests to the source.

**Trade-offs:** Aggressive eviction lowers memory cost but creates source traffic; serving stale data is acceptable only for explicitly chosen fields.

**Staff-level view:** Size for failover with one shard unavailable, including connection storms, allocator overhead, replication buffers, and the source load from a cold replacement.

**Follow-ups:**

- Now combine this scenario with: Adding four nodes remaps keys and doubles database reads. How must the solution change?
- Which part of your solution protects the invariant in this related case: Adding four nodes remaps keys and doubles database reads.

</details>

---

### Sequence space is exhausted

⏱ 12 min

**Scenario:** Unique ID service: A worker needs more IDs in one clock tick than its sequence bits allow.

**Constraints:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage. Preserve: No duplicate IDs; Predictable bit budget.

<details><summary>Hints</summary>

1. Derive the per-tick maximum.
2. Do not silently wrap the sequence.

</details>

<details><summary>Reference answer</summary>

**Answer:** Block until the next safe tick, distribute allocation across leased worker IDs, or revise the bit split. Benchmark peak per worker, and account for a reduced timestamp lifetime when increasing sequence bits.

**Trade-offs:** Sortable IDs expose approximate creation time; larger worker or sequence fields shorten the timestamp horizon.

**Staff-level view:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

**Follow-ups:**

- Now combine this scenario with: A host clock steps backwards by three seconds after synchronization. How must the solution change?
- Which part of your solution protects the invariant in this related case: A partitioned old worker continues issuing IDs after its lease was reassigned.

</details>

---

### Clock moves backwards

⏱ 10 min

**Scenario:** Unique ID service: A host clock steps backwards by three seconds after synchronization.

**Constraints:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage. Preserve: No duplicate IDs; Predictable bit budget.

<details><summary>Hints</summary>

1. Wall time is not monotonic.
2. The last issued timestamp is safety state.

</details>

<details><summary>Reference answer</summary>

**Answer:** Refuse or pause generation until time catches up, or use a persisted logical timestamp strategy with a documented skew budget. Alert on clock regressions; never reuse a timestamp-sequence pair.

**Trade-offs:** Sortable IDs expose approximate creation time; larger worker or sequence fields shorten the timestamp horizon.

**Staff-level view:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

**Follow-ups:**

- Now combine this scenario with: A worker needs more IDs in one clock tick than its sequence bits allow. How must the solution change?
- Which part of your solution protects the invariant in this related case: A partitioned old worker continues issuing IDs after its lease was reassigned.

</details>

---

### A worker ID is reused too soon

⏱ 15 min

**Scenario:** Unique ID service: A partitioned old worker continues issuing IDs after its lease was reassigned.

**Constraints:** 500M IDs/day; bursts of 200K IDs/s; IDs fit in signed 64-bit storage. Preserve: No duplicate IDs; Predictable bit budget.

<details><summary>Hints</summary>

1. Lease expiry does not stop a paused process.
2. Can epochs be encoded in IDs?

</details>

<details><summary>Reference answer</summary>

**Answer:** Ensure the identity-generation epoch is encoded in the uniqueness space or prevent reuse for the full uncertainty interval. Gate generation on conservative lease validity and prove behavior for a paused process that resumes late.

**Trade-offs:** Sortable IDs expose approximate creation time; larger worker or sequence fields shorten the timestamp horizon.

**Staff-level view:** Prove uniqueness across clock rollback, restored VM snapshots, identity reuse, and disaster recovery. Include a documented migration before the timestamp field expires.

**Follow-ups:**

- Now combine this scenario with: A worker needs more IDs in one clock tick than its sequence bits allow. How must the solution change?
- Which part of your solution protects the invariant in this related case: A worker needs more IDs in one clock tick than its sequence bits allow.

</details>

---

### Midnight job cliff

⏱ 12 min

**Scenario:** Durable job scheduler: Three million reports become due at midnight for the same timezone.

**Constraints:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds. Preserve: Schedule and cancel jobs; Retry transient failures.

<details><summary>Hints</summary>

1. Due time and worker start time differ.
2. Can work be spread without breaking promises?

</details>

<details><summary>Reference answer</summary>

**Answer:** Bucket due jobs by time and tenant, introduce user-approved scheduling windows, and reserve urgent capacity. Dispatch with fair queues and backlog-age autoscaling; calculate drain time rather than scaling only on CPU.

**Trade-offs:** At-least-once execution is recoverable but requires idempotent effects; at-most-once execution avoids repeats by accepting lost work.

**Staff-level view:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

**Follow-ups:**

- Now combine this scenario with: The email side effect succeeded but the attempt acknowledgement was lost. How must the solution change?
- Which part of your solution protects the invariant in this related case: A user cancels a job while a worker has acquired its lease.

</details>

---

### Worker dies after sending a report

⏱ 10 min

**Scenario:** Durable job scheduler: The email side effect succeeded but the attempt acknowledgement was lost.

**Constraints:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds. Preserve: Schedule and cancel jobs; Retry transient failures.

<details><summary>Hints</summary>

1. Retries can repeat external effects.
2. Find an idempotency boundary at the destination.

</details>

<details><summary>Reference answer</summary>

**Answer:** Record a stable delivery key per job occurrence and pass it to an idempotent destination when available. Reconcile ambiguous outcomes before resending; represent unknown status if the provider cannot deduplicate.

**Trade-offs:** At-least-once execution is recoverable but requires idempotent effects; at-most-once execution avoids repeats by accepting lost work.

**Staff-level view:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

**Follow-ups:**

- Now combine this scenario with: Three million reports become due at midnight for the same timezone. How must the solution change?
- Which part of your solution protects the invariant in this related case: A user cancels a job while a worker has acquired its lease.

</details>

---

### Cancel races with execution

⏱ 15 min

**Scenario:** Durable job scheduler: A user cancels a job while a worker has acquired its lease.

**Constraints:** 50M jobs/day; 30-day scheduling horizon; execution delay p99 under 10 seconds. Preserve: Schedule and cancel jobs; Retry transient failures.

<details><summary>Hints</summary>

1. Cancellation needs a declared cutoff.
2. A flag cannot undo an external side effect.

</details>

<details><summary>Reference answer</summary>

**Answer:** Define cancel as successful only before a compare-and-set transition to running. Return already-running otherwise, and offer cooperative cancellation for interruptible work. Track occurrence IDs separately for recurring jobs.

**Trade-offs:** At-least-once execution is recoverable but requires idempotent effects; at-most-once execution avoids repeats by accepting lost work.

**Staff-level view:** Specify misfire behavior for missed recurring schedules, timezones, daylight-saving transitions, and prolonged outages; avoid replaying years of missed jobs on recovery.

**Follow-ups:**

- Now combine this scenario with: Three million reports become due at midnight for the same timezone. How must the solution change?
- Which part of your solution protects the invariant in this related case: Three million reports become due at midnight for the same timezone.

</details>

---

## Social

### Celebrity fanout explosion

⏱ 12 min

**Scenario:** Social news feed: An account with 80M followers posts during peak traffic.

**Constraints:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail. Preserve: Publish posts; Cursor pagination.

<details><summary>Hints</summary>

1. Fanout cost follows follower count.
2. Read and write amplification can be mixed.

</details>

<details><summary>Reference answer</summary>

**Answer:** Push posts for ordinary authors into inboxes, pull celebrity posts at read time, and merge by stable rank keys. Bound fanout work per author and measure active-follower coverage instead of writing dormant inboxes.

**Trade-offs:** Fanout-on-write gives fast reads but amplifies celebrity writes; read-time merging saves work at the cost of more feed computation.

**Staff-level view:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

**Follow-ups:**

- Now combine this scenario with: A cached inbox still references content removed by moderation. How must the solution change?
- Which part of your solution protects the invariant in this related case: Scores change between page one and page two, causing duplicates and skips.

</details>

---

### Feed contains a deleted post

⏱ 10 min

**Scenario:** Social news feed: A cached inbox still references content removed by moderation.

**Constraints:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail. Preserve: Publish posts; Cursor pagination.

<details><summary>Hints</summary>

1. Inbox entries are candidates, not authorization.
2. Content visibility needs a final check.

</details>

<details><summary>Reference answer</summary>

**Answer:** Filter candidates against current deletion and access state before rendering, propagate tombstones to caches, and backfill if filtering leaves a short page. Purge materialized entries asynchronously.

**Trade-offs:** Fanout-on-write gives fast reads but amplifies celebrity writes; read-time merging saves work at the cost of more feed computation.

**Staff-level view:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

**Follow-ups:**

- Now combine this scenario with: An account with 80M followers posts during peak traffic. How must the solution change?
- Which part of your solution protects the invariant in this related case: Scores change between page one and page two, causing duplicates and skips.

</details>

---

### Pagination repeats ranked items

⏱ 15 min

**Scenario:** Social news feed: Scores change between page one and page two, causing duplicates and skips.

**Constraints:** 50M daily users; 20 feed reads/day; follower counts have a heavy tail. Preserve: Publish posts; Cursor pagination.

<details><summary>Hints</summary>

1. Offset pagination moves under updates.
2. A session needs a stable ordering contract.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use an opaque cursor containing a ranking snapshot or stable boundary plus a deduplication window. Freeze candidate ordering for a bounded browsing session and refresh explicitly for newer posts.

**Trade-offs:** Fanout-on-write gives fast reads but amplifies celebrity writes; read-time merging saves work at the cost of more feed computation.

**Staff-level view:** Set separate SLOs for post acceptance, follower visibility, privacy revocation, and ranking freshness; a single availability number hides the important failures.

**Follow-ups:**

- Now combine this scenario with: An account with 80M followers posts during peak traffic. How must the solution change?
- Which part of your solution protects the invariant in this related case: An account with 80M followers posts during peak traffic.

</details>

---

### A new thumbnail size goes viral

⏱ 12 min

**Scenario:** Photo sharing: A layout release asks for a previously uncached transform for every photo.

**Constraints:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload. Preserve: Resumable upload; Thumbnail generation.

<details><summary>Hints</summary>

1. Transform diversity multiplies compute.
2. Untrusted sizes must not become arbitrary work.

</details>

<details><summary>Reference answer</summary>

**Answer:** Allowlist transform recipes, key variants by recipe version, precompute popular sizes, and single-flight lazy generation. Rate-limit per original and serve a nearby existing size during backlog.

**Trade-offs:** Precomputing variants increases storage but reduces user-visible transform latency; signed URL lifetime sets part of the access-revocation delay.

**Staff-level view:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

**Follow-ups:**

- Now combine this scenario with: An object event arrives before image validation and the gallery publishes it. How must the solution change?
- Which part of your solution protects the invariant in this related case: A long-lived CDN URL continues to expose a photo after access is revoked.

</details>

---

### Incomplete image marked ready

⏱ 10 min

**Scenario:** Photo sharing: An object event arrives before image validation and the gallery publishes it.

**Constraints:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload. Preserve: Resumable upload; Thumbnail generation.

<details><summary>Hints</summary>

1. Storage completion is not application readiness.
2. Readiness is a state transition.

</details>

<details><summary>Reference answer</summary>

**Answer:** Validate format, dimensions, checksums, and required variants before atomically publishing ready metadata. Keep uploads staged and sweep abandoned objects; the gallery reads only ready records.

**Trade-offs:** Precomputing variants increases storage but reduces user-visible transform latency; signed URL lifetime sets part of the access-revocation delay.

**Staff-level view:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

**Follow-ups:**

- Now combine this scenario with: A layout release asks for a previously uncached transform for every photo. How must the solution change?
- Which part of your solution protects the invariant in this related case: A long-lived CDN URL continues to expose a photo after access is revoked.

</details>

---

### Private image survives album removal

⏱ 15 min

**Scenario:** Photo sharing: A long-lived CDN URL continues to expose a photo after access is revoked.

**Constraints:** 8M uploads/day; 4MB original photos; 10 thumbnail views per upload. Preserve: Resumable upload; Thumbnail generation.

<details><summary>Hints</summary>

1. Cacheability and authorization lifetime interact.
2. Unlisted is not private.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use authenticated delivery or short-lived signed access with a documented revocation bound; invalidate cached variants on urgent removal. Keep object origins private and check album permissions before issuing access.

**Trade-offs:** Precomputing variants increases storage but reduces user-visible transform latency; signed URL lifetime sets part of the access-revocation delay.

**Staff-level view:** Coordinate deletion of originals, variants, caches, and backup-retention records; expose deletion progress without promising that a metadata delete erases every byte instantly.

**Follow-ups:**

- Now combine this scenario with: A layout release asks for a previously uncached transform for every photo. How must the solution change?
- Which part of your solution protects the invariant in this related case: A layout release asks for a previously uncached transform for every photo.

</details>

---

### A celebrity has a huge follower partition

⏱ 12 min

**Scenario:** Follow graph: One account has 60M followers and its inbound row becomes unmanageable.

**Constraints:** 200M accounts; 20B edges; most users have fewer than 1,000 edges. Preserve: Follow and unfollow; List followers.

<details><summary>Hints</summary>

1. Incoming and outgoing access patterns differ.
2. Large adjacency lists need bounded slices.

</details>

<details><summary>Reference answer</summary>

**Answer:** Bucket inbound edges by follower hash and paginate across fixed buckets using an opaque cursor. Keep ordinary outgoing lists simple, and compute counts asynchronously from versioned edge events.

**Trade-offs:** Duplicating inbound and outbound edges accelerates both directions but requires repairable projections and explicit count staleness.

**Staff-level view:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

**Follow-ups:**

- Now combine this scenario with: Unfollow retries decrement counters even when no edge was removed. How must the solution change?
- Which part of your solution protects the invariant in this related case: A blocked account appears in a mutual-friends result produced from lagging indexes.

</details>

---

### Follower counts become negative

⏱ 10 min

**Scenario:** Follow graph: Unfollow retries decrement counters even when no edge was removed.

**Constraints:** 200M accounts; 20B edges; most users have fewer than 1,000 edges. Preserve: Follow and unfollow; List followers.

<details><summary>Hints</summary>

1. A command is not a state transition.
2. Counters should follow committed edge changes.

</details>

<details><summary>Reference answer</summary>

**Answer:** Make edge mutations conditional and emit a transition event only when state changes. Deduplicate projection events, reconcile counts from edge partitions, and clamp display only as a temporary presentation safeguard.

**Trade-offs:** Duplicating inbound and outbound edges accelerates both directions but requires repairable projections and explicit count staleness.

**Staff-level view:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

**Follow-ups:**

- Now combine this scenario with: One account has 60M followers and its inbound row becomes unmanageable. How must the solution change?
- Which part of your solution protects the invariant in this related case: A blocked account appears in a mutual-friends result produced from lagging indexes.

</details>

---

### Blocked user remains in a stale projection

⏱ 15 min

**Scenario:** Follow graph: A blocked account appears in a mutual-friends result produced from lagging indexes.

**Constraints:** 200M accounts; 20B edges; most users have fewer than 1,000 edges. Preserve: Follow and unfollow; List followers.

<details><summary>Hints</summary>

1. Derived graphs may lag.
2. Block checks protect a stronger invariant.

</details>

<details><summary>Reference answer</summary>

**Answer:** Apply current block and visibility rules when returning results, even if graph projections are stale. Prioritize block invalidations and keep the blocked relationship authoritative independently of follower counts.

**Trade-offs:** Duplicating inbound and outbound edges accelerates both directions but requires repairable projections and explicit count staleness.

**Staff-level view:** Design repair jobs that compare projection checkpoints and sample edge symmetry without loading the complete graph into memory.

**Follow-ups:**

- Now combine this scenario with: One account has 60M followers and its inbound row becomes unmanageable. How must the solution change?
- Which part of your solution protects the invariant in this related case: One account has 60M followers and its inbound row becomes unmanageable.

</details>

---

### One event dominates replies

⏱ 12 min

**Scenario:** Threaded comments: A match-final thread receives 20,000 comments per second.

**Constraints:** 30M comments/day; one live event may receive 20K replies/s. Preserve: Post and edit replies; Sort by score or time.

<details><summary>Hints</summary>

1. Partitioning solely by root creates a hot key.
2. Chronological ordering can merge multiple buckets.

</details>

<details><summary>Reference answer</summary>

**Answer:** Shard comment ingestion by root and time/hash bucket, assign stable IDs, and merge bounded slices for reads. Cache the top-level page while separating live append traffic from score recomputation.

**Trade-offs:** Materialized thread paths speed subtree reads but make moves expensive; restricting reparenting simplifies storage and moderation semantics.

**Staff-level view:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

**Follow-ups:**

- Now combine this scenario with: An old body-edit request overwrites a newer hidden state. How must the solution change?
- Which part of your solution protects the invariant in this related case: A parent comment is deleted but hundreds of valid replies remain.

</details>

---

### Moderation disappears after an edit retry

⏱ 10 min

**Scenario:** Threaded comments: An old body-edit request overwrites a newer hidden state.

**Constraints:** 30M comments/day; one live event may receive 20K replies/s. Preserve: Post and edit replies; Sort by score or time.

<details><summary>Hints</summary>

1. Body and moderation versions have different owners.
2. Blind row replacement erases unrelated fields.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use optimistic concurrency and field-specific updates; keep moderation state independently versioned and enforce it in rendering. Reject outdated edits and retain an audit trail of state transitions.

**Trade-offs:** Materialized thread paths speed subtree reads but make moves expensive; restricting reparenting simplifies storage and moderation semantics.

**Staff-level view:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

**Follow-ups:**

- Now combine this scenario with: A match-final thread receives 20,000 comments per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: A parent comment is deleted but hundreds of valid replies remain.

</details>

---

### Deleting a parent orphans discussion

⏱ 15 min

**Scenario:** Threaded comments: A parent comment is deleted but hundreds of valid replies remain.

**Constraints:** 30M comments/day; one live event may receive 20K replies/s. Preserve: Post and edit replies; Sort by score or time.

<details><summary>Hints</summary>

1. Content deletion need not remove structure.
2. What does the thread contract promise?

</details>

<details><summary>Reference answer</summary>

**Answer:** Replace the parent body with a tombstone while retaining structural identifiers. Apply separate policies for subtree removal and author-content deletion; ensure descendants remain paginatable without exposing deleted text.

**Trade-offs:** Materialized thread paths speed subtree reads but make moves expensive; restricting reparenting simplifies storage and moderation semantics.

**Staff-level view:** Distinguish author deletion, moderator hiding, legal removal, and thread locking, since each has different visibility and audit requirements.

**Follow-ups:**

- Now combine this scenario with: A match-final thread receives 20,000 comments per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: A match-final thread receives 20,000 comments per second.

</details>

---

### Viewer list overwhelms one story

⏱ 12 min

**Scenario:** Ephemeral stories: A public story accumulates 30M unique viewers in an hour.

**Constraints:** 15M stories/day; 150M daily story views; media expiry is strict at serving time. Preserve: 24-hour visibility; Audience rules.

<details><summary>Hints</summary>

1. Exact viewer lists are different from counts.
2. Writes should spread beyond the story ID.

</details>

<details><summary>Reference answer</summary>

**Answer:** Hash-bucket view receipts by viewer, maintain approximate or delayed public counts, and page exact lists only where the product requires them. Deduplicate with a stable story-viewer key.

**Trade-offs:** Exact receipts improve creator analytics but increase privacy and storage cost; delayed aggregates are cheaper for huge audiences.

**Staff-level view:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

**Follow-ups:**

- Now combine this scenario with: A cleanup worker is late, leaving an expired object in storage and cache. How must the solution change?
- Which part of your solution protects the invariant in this related case: A creator switches from public to close friends while existing sessions are active.

</details>

---

### Expired story remains playable

⏱ 10 min

**Scenario:** Ephemeral stories: A cleanup worker is late, leaving an expired object in storage and cache.

**Constraints:** 15M stories/day; 150M daily story views; media expiry is strict at serving time. Preserve: 24-hour visibility; Audience rules.

<details><summary>Hints</summary>

1. Deletion schedules are not visibility checks.
2. Authorization must compare current time.

</details>

<details><summary>Reference answer</summary>

**Answer:** Enforce expires_at when issuing playback access and use access lifetimes no longer than remaining story life. Purge asynchronously, but deny new serving authorization after expiry even when bytes still exist.

**Trade-offs:** Exact receipts improve creator analytics but increase privacy and storage cost; delayed aggregates are cheaper for huge audiences.

**Staff-level view:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

**Follow-ups:**

- Now combine this scenario with: A public story accumulates 30M unique viewers in an hour. How must the solution change?
- Which part of your solution protects the invariant in this related case: A creator switches from public to close friends while existing sessions are active.

</details>

---

### Audience changes during viewing

⏱ 15 min

**Scenario:** Ephemeral stories: A creator switches from public to close friends while existing sessions are active.

**Constraints:** 15M stories/day; 150M daily story views; media expiry is strict at serving time. Preserve: 24-hour visibility; Audience rules.

<details><summary>Hints</summary>

1. Authorization decisions have a lifetime.
2. Define the cutoff for existing viewers.

</details>

<details><summary>Reference answer</summary>

**Answer:** Version audience rules and revalidate when renewing access. Bound existing signed-token validity, invalidate where supported, and document whether already downloaded media can continue playing.

**Trade-offs:** Exact receipts improve creator analytics but increase privacy and storage cost; delayed aggregates are cheaper for huge audiences.

**Staff-level view:** Separate the promised visibility deadline from physical deletion and backup expiration, and measure each boundary independently.

**Follow-ups:**

- Now combine this scenario with: A public story accumulates 30M unique viewers in an hour. How must the solution change?
- Which part of your solution protects the invariant in this related case: A public story accumulates 30M unique viewers in an hour.

</details>

---

## Messaging

### One busy group overloads its owner

⏱ 12 min

**Scenario:** Direct and group chat: A 1,000-member trading room generates 10,000 messages per second.

**Constraints:** 50M daily users; 40 messages/user/day; groups up to 1,000 members. Preserve: Conversation ordering; Offline sync.

<details><summary>Hints</summary>

1. Ordering scope constrains parallelism.
2. Fanout costs more than one append.

</details>

<details><summary>Reference answer</summary>

**Answer:** Keep one sequencer per conversation but separate append from sharded recipient fanout. Batch deliveries, cap outstanding bytes per connection, and enforce a room traffic limit if one sequencer reaches its measured ceiling.

**Trade-offs:** Per-conversation order is useful and affordable; a global order across all chats introduces coordination with little user benefit.

**Staff-level view:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

**Follow-ups:**

- Now combine this scenario with: A gateway acknowledged receipt before durable storage and crashed. How must the solution change?
- Which part of your solution protects the invariant in this related case: Queued fanout includes a user whose membership was revoked.

</details>

---

### Messages vanish after reconnect

⏱ 10 min

**Scenario:** Direct and group chat: A gateway acknowledged receipt before durable storage and crashed.

**Constraints:** 50M daily users; 40 messages/user/day; groups up to 1,000 members. Preserve: Conversation ordering; Offline sync.

<details><summary>Hints</summary>

1. Received is not durably accepted.
2. The client needs a replay cursor.

</details>

<details><summary>Reference answer</summary>

**Answer:** Acknowledge acceptance only after durable append; use clientMessageId for retry deduplication. On reconnect, fetch from the last durable sequence and deduplicate live deliveries against replay.

**Trade-offs:** Per-conversation order is useful and affordable; a global order across all chats introduces coordination with little user benefit.

**Staff-level view:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

**Follow-ups:**

- Now combine this scenario with: A 1,000-member trading room generates 10,000 messages per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: Queued fanout includes a user whose membership was revoked.

</details>

---

### A removed member receives delayed messages

⏱ 15 min

**Scenario:** Direct and group chat: Queued fanout includes a user whose membership was revoked.

**Constraints:** 50M daily users; 40 messages/user/day; groups up to 1,000 members. Preserve: Conversation ordering; Offline sync.

<details><summary>Hints</summary>

1. Authorization must include message and membership versions.
2. Revocation policy determines historical access.

</details>

<details><summary>Reference answer</summary>

**Answer:** Define whether membership is evaluated at send time or delivery time, record the relevant membership epoch, and check current eligibility for future delivery. Stop live fanout immediately and enforce the same rule in history sync.

**Trade-offs:** Per-conversation order is useful and affordable; a global order across all chats introduces coordination with little user benefit.

**Staff-level view:** Specify device synchronization, end-to-end encryption metadata boundaries, abuse reporting, and multi-region ownership transfer without claiming that a WebSocket alone guarantees delivery.

**Follow-ups:**

- Now combine this scenario with: A 1,000-member trading room generates 10,000 messages per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: A 1,000-member trading room generates 10,000 messages per second.

</details>

---

### Marketing consumes transactional capacity

⏱ 12 min

**Scenario:** Notification platform: A campaign backlog delays password reset messages by twenty minutes.

**Constraints:** 500M notification intents/day; provider quotas differ by region and channel. Preserve: Preferences and quiet hours; Per-channel retries.

<details><summary>Hints</summary>

1. Queue isolation should follow urgency.
2. Provider quotas are shared resources.

</details>

<details><summary>Reference answer</summary>

**Answer:** Reserve provider throughput for transactional traffic, use separate priority queues with starvation protection, and cap campaign admission. Autoscale workers only up to provider quotas and alert on oldest critical intent age.

**Trade-offs:** Provider failover improves availability but can duplicate ambiguous deliveries; status should preserve uncertainty instead of inventing certainty.

**Staff-level view:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

**Follow-ups:**

- Now combine this scenario with: A request timed out after the provider may have sent the SMS. How must the solution change?
- Which part of your solution protects the invariant in this related case: The user disables promotional email after an intent was queued.

</details>

---

### SMS provider accepted an ambiguous request

⏱ 10 min

**Scenario:** Notification platform: A request timed out after the provider may have sent the SMS.

**Constraints:** 500M notification intents/day; provider quotas differ by region and channel. Preserve: Preferences and quiet hours; Per-channel retries.

<details><summary>Hints</summary>

1. A timeout does not prove failure.
2. The provider may support a stable request key.

</details>

<details><summary>Reference answer</summary>

**Answer:** Retry with the same provider idempotency key where supported and reconcile using delivery lookup or callbacks. Otherwise mark the attempt uncertain and apply a product-specific duplicate-versus-loss policy.

**Trade-offs:** Provider failover improves availability but can duplicate ambiguous deliveries; status should preserve uncertainty instead of inventing certainty.

**Staff-level view:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

**Follow-ups:**

- Now combine this scenario with: A campaign backlog delays password reset messages by twenty minutes. How must the solution change?
- Which part of your solution protects the invariant in this related case: The user disables promotional email after an intent was queued.

</details>

---

### Opt-out races with a queued campaign

⏱ 15 min

**Scenario:** Notification platform: The user disables promotional email after an intent was queued.

**Constraints:** 500M notification intents/day; provider quotas differ by region and channel. Preserve: Preferences and quiet hours; Per-channel retries.

<details><summary>Hints</summary>

1. Preference snapshots can become stale.
2. Delivery-time eligibility matters.

</details>

<details><summary>Reference answer</summary>

**Answer:** Recheck current marketing consent immediately before provider submission and record the evaluated preference version. Define a clear cutoff once submission has occurred and propagate suppression changes to all channels.

**Trade-offs:** Provider failover improves availability but can duplicate ambiguous deliveries; status should preserve uncertainty instead of inventing certainty.

**Staff-level view:** Set channel-specific SLOs and per-tenant cost budgets, and rehearse a provider brownout with callbacks arriving after failover.

**Follow-ups:**

- Now combine this scenario with: A campaign backlog delays password reset messages by twenty minutes. How must the solution change?
- Which part of your solution protects the invariant in this related case: A campaign backlog delays password reset messages by twenty minutes.

</details>

---

### Large attachments dominate ingress

⏱ 12 min

**Scenario:** Email service: A few senders upload 100MB attachments and exhaust spool disk.

**Constraints:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes. Preserve: Durable acceptance; Attachments.

<details><summary>Hints</summary>

1. Budget bytes as well as messages.
2. Streaming avoids whole-body buffering.

</details>

<details><summary>Reference answer</summary>

**Answer:** Enforce sender and mailbox byte quotas before acceptance, stream attachments to staged object storage, and apply bounded concurrency for scanners. Reject before durable acceptance when capacity is unavailable.

**Trade-offs:** Scanning before acceptance reduces accepted malicious mail but lengthens SMTP latency; scanning after durable spooling requires quarantine and status handling.

**Staff-level view:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

**Follow-ups:**

- Now combine this scenario with: The SMTP edge returns success before its local spool is replicated. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two devices move and mark a message read using stale mailbox state.

</details>

---

### Accepted mail disappears during a crash

⏱ 10 min

**Scenario:** Email service: The SMTP edge returns success before its local spool is replicated.

**Constraints:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes. Preserve: Durable acceptance; Attachments.

<details><summary>Hints</summary>

1. Acceptance is a durability promise.
2. The spool needs a tested failure boundary.

</details>

<details><summary>Reference answer</summary>

**Answer:** Acknowledge only after the configured durable replicated write, then perform routing asynchronously. Recover from spool checkpoints and deduplicate delivery by message and recipient identifiers.

**Trade-offs:** Scanning before acceptance reduces accepted malicious mail but lengthens SMTP latency; scanning after durable spooling requires quarantine and status handling.

**Staff-level view:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

**Follow-ups:**

- Now combine this scenario with: A few senders upload 100MB attachments and exhaust spool disk. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two devices move and mark a message read using stale mailbox state.

</details>

---

### Folder moves race with flag updates

⏱ 15 min

**Scenario:** Email service: Two devices move and mark a message read using stale mailbox state.

**Constraints:** 100M inbound messages/day; 80KB average body plus attachments; 1M active mailboxes. Preserve: Durable acceptance; Attachments.

<details><summary>Hints</summary>

1. Whole-record replacement loses independent edits.
2. Flags and membership need explicit semantics.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use version-checked or operation-based mailbox updates with stable UIDs. Separate independent flags from folder membership, return conflict for incompatible moves, and expose a change cursor for device reconciliation.

**Trade-offs:** Scanning before acceptance reduces accepted malicious mail but lengthens SMTP latency; scanning after durable spooling requires quarantine and status handling.

**Staff-level view:** Plan mailbox export, delivery-loop protection, per-recipient bounce state, and restore tests that verify both bodies and mailbox indexes.

**Follow-ups:**

- Now combine this scenario with: A few senders upload 100MB attachments and exhaust spool disk. How must the solution change?
- Which part of your solution protects the invariant in this related case: A few senders upload 100MB attachments and exhaust spool disk.

</details>

---

### One slow endpoint consumes workers

⏱ 12 min

**Scenario:** Webhook delivery: An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants.

**Constraints:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable. Preserve: Durable events; Endpoint-specific retries.

<details><summary>Hints</summary>

1. Concurrency limits need an endpoint scope.
2. Timeouts and queue age reveal saturation.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use per-endpoint concurrency caps, separate tenant queues, bounded request deadlines, and a circuit breaker. Keep global workers available for healthy endpoints and give the customer clear backlog and replay controls.

**Trade-offs:** Strict delivery order simplifies receivers but lets one poison event block a stream; versioned events permit parallel recovery.

**Staff-level view:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

**Follow-ups:**

- Now combine this scenario with: A receiver commits a database change and then its response path fails. How must the solution change?
- Which part of your solution protects the invariant in this related case: The delete event is delivered before a delayed create event.

</details>

---

### Receiver processed the event but returned 500

⏱ 10 min

**Scenario:** Webhook delivery: A receiver commits a database change and then its response path fails.

**Constraints:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable. Preserve: Durable events; Endpoint-specific retries.

<details><summary>Hints</summary>

1. Transport status and business effect can differ.
2. The event ID must remain stable across attempts.

</details>

<details><summary>Reference answer</summary>

**Answer:** Retry the same event ID with a new attempt ID, document at-least-once delivery, and provide signing over immutable event bytes plus timestamp. Receivers must deduplicate durable effects by event ID.

**Trade-offs:** Strict delivery order simplifies receivers but lets one poison event block a stream; versioned events permit parallel recovery.

**Staff-level view:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

**Follow-ups:**

- Now combine this scenario with: An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants. How must the solution change?
- Which part of your solution protects the invariant in this related case: The delete event is delivered before a delayed create event.

</details>

---

### A newer update arrives before an older retry

⏱ 15 min

**Scenario:** Webhook delivery: The delete event is delivered before a delayed create event.

**Constraints:** 200M deliveries/day; 100K endpoints; endpoints can be slow or unavailable. Preserve: Durable events; Endpoint-specific retries.

<details><summary>Hints</summary>

1. Retries break naive arrival-order assumptions.
2. A resource version can supersede older events.

</details>

<details><summary>Reference answer</summary>

**Answer:** Include aggregate ID and monotonic resource version; receivers ignore superseded updates or fetch current state. Offer ordered per-resource delivery only with an explicit head-of-line blocking tradeoff.

**Trade-offs:** Strict delivery order simplifies receivers but lets one poison event block a stream; versioned events permit parallel recovery.

**Staff-level view:** Make replay preserve original event identity while recording a distinct replay request, and bound customer endpoint access to prevent internal-network targeting.

**Follow-ups:**

- Now combine this scenario with: An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants. How must the solution change?
- Which part of your solution protects the invariant in this related case: An enterprise endpoint takes 25 seconds per request and blocks unrelated tenants.

</details>

---

### Consumer fanout multiplies bandwidth

⏱ 12 min

**Scenario:** Publish-subscribe event bus: Twenty consumer groups each read the full billion-event stream.

**Constraints:** 1B events/day; 1KB mean event; 20 consumer groups. Preserve: Partition ordering; Replay retention.

<details><summary>Hints</summary>

1. One stored byte may be read many times.
2. Partition count is only one dimension.

</details>

<details><summary>Reference answer</summary>

**Answer:** Model aggregate broker egress and catch-up reads, batch and compress events, isolate heavy replays, and tier old data. Add partitions only after checking key-order implications and consumer parallelism.

**Trade-offs:** Long retention enables recovery but increases storage and replay load; ordering by one hot key limits available parallelism.

**Staff-level view:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

**Follow-ups:**

- Now combine this scenario with: A broken analytics consumer resumes after its oldest required offset was deleted. How must the solution change?
- Which part of your solution protects the invariant in this related case: The consumer writes to a database then crashes before committing its offset.

</details>

---

### Consumer is behind retention

⏱ 10 min

**Scenario:** Publish-subscribe event bus: A broken analytics consumer resumes after its oldest required offset was deleted.

**Constraints:** 1B events/day; 1KB mean event; 20 consumer groups. Preserve: Partition ordering; Replay retention.

<details><summary>Hints</summary>

1. A checkpoint can outlive its data.
2. Recovery needs a new baseline.

</details>

<details><summary>Reference answer</summary>

**Answer:** Detect offset-out-of-range explicitly, restore a consistent snapshot, and replay from its associated checkpoint. Do not silently skip to newest; publish data-loss or rebuild status to downstream owners.

**Trade-offs:** Long retention enables recovery but increases storage and replay load; ordering by one hot key limits available parallelism.

**Staff-level view:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

**Follow-ups:**

- Now combine this scenario with: Twenty consumer groups each read the full billion-event stream. How must the solution change?
- Which part of your solution protects the invariant in this related case: The consumer writes to a database then crashes before committing its offset.

</details>

---

### A database sink duplicates increments

⏱ 15 min

**Scenario:** Publish-subscribe event bus: The consumer writes to a database then crashes before committing its offset.

**Constraints:** 1B events/day; 1KB mean event; 20 consumer groups. Preserve: Partition ordering; Replay retention.

<details><summary>Hints</summary>

1. Broker guarantees stop at an external side effect.
2. Store dedupe with the business update.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a durable event-ID uniqueness record in the same transaction as the sink mutation, then advance the broker checkpoint. Replays become no-ops; Kafka transactions alone do not make arbitrary database writes exactly once.

**Trade-offs:** Long retention enables recovery but increases storage and replay load; ordering by one hot key limits available parallelism.

**Staff-level view:** Budget replay traffic separately, govern schema compatibility, and provide tenant-level kill switches that preserve other streams during poison-event incidents.

**Follow-ups:**

- Now combine this scenario with: Twenty consumer groups each read the full billion-event stream. How must the solution change?
- Which part of your solution protects the invariant in this related case: Twenty consumer groups each read the full billion-event stream.

</details>

---

## Media

### A viral upload is cold at every edge

⏱ 12 min

**Scenario:** YouTube-style video platform: A newly published video draws 2M concurrent viewers while CDN segment caches are empty.

**Constraints:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions. Preserve: Resumable multipart uploads; HLS and DASH adaptive playback.

<details><summary>Hints</summary>

1. Compute edge egress separately from origin egress.
2. Many edges can request the same segment at once.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use immutable versioned segment URLs, an origin shield, request collapsing, and prewarm initial segments for predicted hot videos. Keep signed authorization from fragmenting cache keys. Measure origin bytes, startup time, rebuffer ratio, and per-rendition cache hit rate; reduce initial bitrate during overload.

**Trade-offs:** More codecs and renditions improve device coverage and bandwidth efficiency but increase encoding time and storage; eager encoding helps popular uploads while demand-driven renditions reduce waste.

**Staff-level view:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

**Follow-ups:**

- Now combine this scenario with: A packager updates the published manifest before all rendition segments are durable and validated. How must the solution change?
- Which part of your solution protects the invariant in this related case: A long-running encoder finishes after its lease expired and a replacement worker started the same rendition.

</details>

---

### A manifest advertises missing segments

⏱ 10 min

**Scenario:** YouTube-style video platform: A packager updates the published manifest before all rendition segments are durable and validated.

**Constraints:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions. Preserve: Resumable multipart uploads; HLS and DASH adaptive playback.

<details><summary>Hints</summary>

1. Playback manifests are a publication boundary.
2. Retries must not overwrite another recipe version.

</details>

<details><summary>Reference answer</summary>

**Answer:** Write outputs under immutable video/source/recipe/rendition keys. Validate checksums, codec metadata, aligned segment timelines, and required assets, then conditionally promote a publication pointer. Keep the previous complete version available. For live playback, publish only segments that meet the availability contract and distinguish manifest TTL from immutable segment TTL.

**Trade-offs:** More codecs and renditions improve device coverage and bandwidth efficiency but increase encoding time and storage; eager encoding helps popular uploads while demand-driven renditions reduce waste.

**Staff-level view:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

**Follow-ups:**

- Now combine this scenario with: A newly published video draws 2M concurrent viewers while CDN segment caches are empty. How must the solution change?
- Which part of your solution protects the invariant in this related case: A long-running encoder finishes after its lease expired and a replacement worker started the same rendition.

</details>

---

### A transcode worker retries after lease loss

⏱ 15 min

**Scenario:** YouTube-style video platform: A long-running encoder finishes after its lease expired and a replacement worker started the same rendition.

**Constraints:** 20M daily viewers watch 30 minutes each at an assumed 3 Mb/s average: 13.5 PB/day of delivered video; 100K uploads/day averaging 600MB = 60TB/day of originals before renditions. Preserve: Resumable multipart uploads; HLS and DASH adaptive playback.

<details><summary>Hints</summary>

1. A lease alone does not fence late writes.
2. The output identity must include source and recipe versions.

</details>

<details><summary>Reference answer</summary>

**Answer:** Key each job by videoId, sourceVersion, recipeVersion, and rendition. Write attempt outputs to staging, verify checksums, and use a fenced compare-and-set to commit one successful asset set. Losing attempts are harmless and garbage-collected. A durable workflow records stage completion; at-least-once queue delivery is expected.

**Trade-offs:** More codecs and renditions improve device coverage and bandwidth efficiency but increase encoding time and storage; eager encoding helps popular uploads while demand-driven renditions reduce waste.

**Staff-level view:** Separate upload, processing, playback-start, rebuffering, and deletion SLOs. Compare regional encoding cost, CDN egress, and origin shielding; plan takedown propagation through authorization, manifests, caches, search, and derived clips.

**Follow-ups:**

- Now combine this scenario with: A newly published video draws 2M concurrent viewers while CDN segment caches are empty. How must the solution change?
- Which part of your solution protects the invariant in this related case: A newly published video draws 2M concurrent viewers while CDN segment caches are empty.

</details>

---

### Final-minute sports traffic surge

⏱ 12 min

**Scenario:** Live streaming platform: A match audience grows from 200K to 5M viewers in ninety seconds.

**Constraints:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds. Preserve: Authenticated ingest; Live manifest updates.

<details><summary>Hints</summary>

1. The first segment and manifest are hot objects.
2. Live manifests cannot have VOD-length TTLs.

</details>

<details><summary>Reference answer</summary>

**Answer:** Provision CDN delivery headroom, shield segment origin, collapse requests, and keep a carefully bounded manifest refresh cadence. Use player bitrate adaptation and cohort-based failover rather than redirecting every viewer simultaneously.

**Trade-offs:** Smaller segments reduce live delay but increase request overhead and sensitivity to jitter; longer buffers improve resilience while moving viewers further behind live.

**Staff-level view:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

**Follow-ups:**

- Now combine this scenario with: An encoder reconnects with a new timeline and existing players stall. How must the solution change?
- Which part of your solution protects the invariant in this related case: A broadcaster reconnects to another region while the old ingest keeps publishing.

</details>

---

### Encoder restart resets timestamps

⏱ 10 min

**Scenario:** Live streaming platform: An encoder reconnects with a new timeline and existing players stall.

**Constraints:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds. Preserve: Authenticated ingest; Live manifest updates.

<details><summary>Hints</summary>

1. Segment sequence and media time are distinct.
2. A discontinuity must be explicit.

</details>

<details><summary>Reference answer</summary>

**Answer:** Start a new ingest epoch, emit protocol-appropriate discontinuity or period boundaries, and preserve coherent sequence and timing metadata. Validate with players across codecs; keep DVR references tied to immutable segment identities.

**Trade-offs:** Smaller segments reduce live delay but increase request overhead and sensitivity to jitter; longer buffers improve resilience while moving viewers further behind live.

**Staff-level view:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

**Follow-ups:**

- Now combine this scenario with: A match audience grows from 200K to 5M viewers in ninety seconds. How must the solution change?
- Which part of your solution protects the invariant in this related case: A broadcaster reconnects to another region while the old ingest keeps publishing.

</details>

---

### Two ingests believe they own a stream

⏱ 15 min

**Scenario:** Live streaming platform: A broadcaster reconnects to another region while the old ingest keeps publishing.

**Constraints:** 10K concurrent broadcasters; 5M peak viewers; target glass-to-glass latency 5 seconds. Preserve: Authenticated ingest; Live manifest updates.

<details><summary>Hints</summary>

1. Single publisher ownership needs fencing.
2. A new epoch should invalidate old publication.

</details>

<details><summary>Reference answer</summary>

**Answer:** Allocate a monotonic stream epoch through an authoritative coordinator and permit publication only for the current epoch. Old workers may finish objects but cannot advance the visible manifest.

**Trade-offs:** Smaller segments reduce live delay but increase request overhead and sensitivity to jitter; longer buffers improve resilience while moving viewers further behind live.

**Staff-level view:** Treat live latency, playback availability, recording completeness, and rights enforcement as independent objectives; test failover mid-segment and mid-ad-break.

**Follow-ups:**

- Now combine this scenario with: A match audience grows from 200K to 5M viewers in ninety seconds. How must the solution change?
- Which part of your solution protects the invariant in this related case: A match audience grows from 200K to 5M viewers in ninety seconds.

</details>

---

### A new album creates a cache cliff

⏱ 12 min

**Scenario:** Music streaming: Ten million listeners start the same album immediately at release.

**Constraints:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track. Preserve: Low-latency track start; Playlist edits.

<details><summary>Hints</summary>

1. Authorization traffic does not cache like audio.
2. Preposition immutable assets without leaking release access.

</details>

<details><summary>Reference answer</summary>

**Answer:** Prewarm encrypted or access-controlled audio assets and cache catalog metadata separately from entitlement decisions. Stagger telemetry uploads and scale authorization by active sessions, not audio segment count.

**Trade-offs:** Frequent entitlement checks improve revocation latency but add startup latency and dependency load; token lifetime sets a measurable compromise.

**Staff-level view:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

**Follow-ups:**

- Now combine this scenario with: A track loses rights in a territory while long-lived playback tokens remain valid. How must the solution change?
- Which part of your solution protects the invariant in this related case: A phone adds tracks while a laptop reorders an old playlist snapshot.

</details>

---

### Rights change but cached playback remains authorized

⏱ 10 min

**Scenario:** Music streaming: A track loses rights in a territory while long-lived playback tokens remain valid.

**Constraints:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track. Preserve: Low-latency track start; Playlist edits.

<details><summary>Hints</summary>

1. Authorization has a maximum lifetime.
2. CDN object TTL need not equal entitlement TTL.

</details>

<details><summary>Reference answer</summary>

**Answer:** Issue short-lived playback authorization tied to rights version and region, recheck on renewal, and invalidate urgent removals. Keep immutable audio caching efficient while applying access controls at the serving boundary.

**Trade-offs:** Frequent entitlement checks improve revocation latency but add startup latency and dependency load; token lifetime sets a measurable compromise.

**Staff-level view:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

**Follow-ups:**

- Now combine this scenario with: Ten million listeners start the same album immediately at release. How must the solution change?
- Which part of your solution protects the invariant in this related case: A phone adds tracks while a laptop reorders an old playlist snapshot.

</details>

---

### Two devices overwrite playlist edits

⏱ 15 min

**Scenario:** Music streaming: A phone adds tracks while a laptop reorders an old playlist snapshot.

**Constraints:** 20M daily listeners; 40 tracks/day; 4MB average delivered audio per track. Preserve: Low-latency track start; Playlist edits.

<details><summary>Hints</summary>

1. Whole-list replacement loses intent.
2. Operations need stable entry identifiers.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use versioned edit operations on stable playlist entry IDs and return conflicts for ambiguous moves. Rebase independent additions and deletions; reserve a CRDT only if offline concurrent editing is a real requirement.

**Trade-offs:** Frequent entitlement checks improve revocation latency but add startup latency and dependency load; token lifetime sets a measurable compromise.

**Staff-level view:** Maintain an auditable rights-decision version and reconcile royalty events by stable playback IDs without counting network retries as extra listens.

**Follow-ups:**

- Now combine this scenario with: Ten million listeners start the same album immediately at release. How must the solution change?
- Which part of your solution protects the invariant in this related case: Ten million listeners start the same album immediately at release.

</details>

---

### Large town hall exhausts SFU egress

⏱ 12 min

**Scenario:** Video conferencing: A 2,000-person meeting sends every camera to every participant.

**Constraints:** 500K concurrent users; average room size 6; target interactive latency below 300 ms. Preserve: Join and leave rooms; Screen share.

<details><summary>Hints</summary>

1. Selective forwarding still costs bandwidth.
2. Subscriptions should follow visible tiles.

</details>

<details><summary>Reference answer</summary>

**Answer:** Limit active senders, forward selected simulcast or scalable-video layers per subscriber, and move passive viewers to broadcast delivery when acceptable. Model SFU egress by subscribed tracks rather than participants squared by default.

**Trade-offs:** An SFU saves endpoint uplink compared with full mesh but costs server egress; mixing simplifies clients at a substantial compute and latency cost.

**Staff-level view:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

**Follow-ups:**

- Now combine this scenario with: Signaling succeeds but participants see no media. How must the solution change?
- Which part of your solution protects the invariant in this related case: A previously valid join credential is reused after host removal.

</details>

---

### UDP is blocked on a corporate network

⏱ 10 min

**Scenario:** Video conferencing: Signaling succeeds but participants see no media.

**Constraints:** 500K concurrent users; average room size 6; target interactive latency below 300 ms. Preserve: Join and leave rooms; Screen share.

<details><summary>Hints</summary>

1. Signaling success is not media connectivity.
2. Relay paths need capacity and telemetry.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use ICE connectivity checks with provisioned TURN fallback, track relay selection and connection setup stages, and reduce bitrate on constrained paths. Show connection recovery status rather than retrying room creation.

**Trade-offs:** An SFU saves endpoint uplink compared with full mesh but costs server egress; mixing simplifies clients at a substantial compute and latency cost.

**Staff-level view:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

**Follow-ups:**

- Now combine this scenario with: A 2,000-person meeting sends every camera to every participant. How must the solution change?
- Which part of your solution protects the invariant in this related case: A previously valid join credential is reused after host removal.

</details>

---

### A removed participant reconnects with an old token

⏱ 15 min

**Scenario:** Video conferencing: A previously valid join credential is reused after host removal.

**Constraints:** 500K concurrent users; average room size 6; target interactive latency below 300 ms. Preserve: Join and leave rooms; Screen share.

<details><summary>Hints</summary>

1. Room membership and credential validity differ.
2. A token may need a session epoch.

</details>

<details><summary>Reference answer</summary>

**Answer:** Bind credentials to room and participant session epochs, check revocation at join, and remove existing SFU subscriptions on kick. Set a short token lifetime and reauthorize reconnects.

**Trade-offs:** An SFU saves endpoint uplink compared with full mesh but costs server egress; mixing simplifies clients at a substantial compute and latency cost.

**Staff-level view:** Partition failure domains by room, quantify regional evacuation capacity, and make recording consent and membership policy explicit without putting media through the signaling service.

**Follow-ups:**

- Now combine this scenario with: A 2,000-person meeting sends every camera to every participant. How must the solution change?
- Which part of your solution protects the invariant in this related case: A 2,000-person meeting sends every camera to every participant.

</details>

---

### Polling every feed wastes most requests

⏱ 12 min

**Scenario:** Podcast platform: Two million feeds are polled every minute although most publish weekly.

**Constraints:** 2M feeds; 200K updated episodes/day; 10M listeners. Preserve: Feed ingestion; Episode deduplication.

<details><summary>Hints</summary>

1. Freshness requirements vary by feed activity.
2. HTTP validators avoid transferring unchanged content.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use adaptive polling based on recent publication intervals, conditional requests with validators, and a minimum/maximum poll cadence. Allocate host-level concurrency budgets and prioritize recently active followed shows.

**Trade-offs:** Aggressive polling improves freshness for active shows but wastes network and publisher capacity; adaptive schedules require starvation bounds for quiet feeds.

**Staff-level view:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

**Follow-ups:**

- Now combine this scenario with: A feed replaces audio under an existing identifier and users download mixed versions. How must the solution change?
- Which part of your solution protects the invariant in this related case: An old offline device overwrites a newer position with a smaller value.

</details>

---

### A publisher reuses an episode GUID

⏱ 10 min

**Scenario:** Podcast platform: A feed replaces audio under an existing identifier and users download mixed versions.

**Constraints:** 2M feeds; 200K updated episodes/day; 10M listeners. Preserve: Feed ingestion; Episode deduplication.

<details><summary>Hints</summary>

1. Publisher identifiers may not be reliable.
2. Immutable asset versions protect partial downloads.

</details>

<details><summary>Reference answer</summary>

**Answer:** Track GUID plus observed enclosure metadata and content version, validate changed media, and publish a new asset version deliberately. Keep existing range downloads pinned to their original immutable object.

**Trade-offs:** Aggressive polling improves freshness for active shows but wastes network and publisher capacity; adaptive schedules require starvation bounds for quiet feeds.

**Staff-level view:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

**Follow-ups:**

- Now combine this scenario with: Two million feeds are polled every minute although most publish weekly. How must the solution change?
- Which part of your solution protects the invariant in this related case: An old offline device overwrites a newer position with a smaller value.

</details>

---

### Progress regresses after offline sync

⏱ 15 min

**Scenario:** Podcast platform: An old offline device overwrites a newer position with a smaller value.

**Constraints:** 2M feeds; 200K updated episodes/day; 10M listeners. Preserve: Feed ingestion; Episode deduplication.

<details><summary>Hints</summary>

1. Maximum position fails when the user intentionally rewinds.
2. Session intent needs an ordering model.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use session versions and explicit seek events, reject stale session writes, and let users choose which active session controls progress. Do not blindly take the maximum or last client wall-clock timestamp.

**Trade-offs:** Aggressive polling improves freshness for active shows but wastes network and publisher capacity; adaptive schedules require starvation bounds for quiet feeds.

**Staff-level view:** Isolate hostile or malformed feeds with byte, recursion, redirect, and fetch-time limits, and build explainable ingestion status for creators.

**Follow-ups:**

- Now combine this scenario with: Two million feeds are polled every minute although most publish weekly. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two million feeds are polled every minute although most publish weekly.

</details>

---

## Storage

### A shared folder has a million small files

⏱ 12 min

**Scenario:** Cloud file synchronization: Initial synchronization performs one request per file and takes days.

**Constraints:** 10M active users; 100M file mutations/day; typical file 2MB. Preserve: Resumable uploads; Offline edits.

<details><summary>Hints</summary>

1. Metadata round trips dominate small objects.
2. A snapshot needs a matching change cursor.

</details>

<details><summary>Reference answer</summary>

**Answer:** Provide paginated batched manifests, compressed metadata snapshots, and chunked directory traversal. Capture a consistent snapshot cursor then apply subsequent changes; limit parallel downloads per client and preserve resumability.

**Trade-offs:** Content-defined chunking reduces upload bytes for edits but adds client CPU, indexing complexity, and encryption/deduplication tradeoffs.

**Staff-level view:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

**Follow-ups:**

- Now combine this scenario with: The metadata pointer changes while some chunks are still missing. How must the solution change?
- Which part of your solution protects the invariant in this related case: Both devices upload valid content based on version 12.

</details>

---

### A partial upload becomes the current version

⏱ 10 min

**Scenario:** Cloud file synchronization: The metadata pointer changes while some chunks are still missing.

**Constraints:** 10M active users; 100M file mutations/day; typical file 2MB. Preserve: Resumable uploads; Offline edits.

<details><summary>Hints</summary>

1. A manifest is a publication boundary.
2. Chunk presence must be verified before promotion.

</details>

<details><summary>Reference answer</summary>

**Answer:** Stage chunks and verify their identities, then atomically commit a version manifest and current pointer. Readers see only complete versions; a garbage collector removes unreferenced staged chunks after a grace period.

**Trade-offs:** Content-defined chunking reduces upload bytes for edits but adds client CPU, indexing complexity, and encryption/deduplication tradeoffs.

**Staff-level view:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

**Follow-ups:**

- Now combine this scenario with: Initial synchronization performs one request per file and takes days. How must the solution change?
- Which part of your solution protects the invariant in this related case: Both devices upload valid content based on version 12.

</details>

---

### Two offline devices modify the same file

⏱ 15 min

**Scenario:** Cloud file synchronization: Both devices upload valid content based on version 12.

**Constraints:** 10M active users; 100M file mutations/day; typical file 2MB. Preserve: Resumable uploads; Offline edits.

<details><summary>Hints</summary>

1. Binary files generally cannot merge semantically.
2. Preserve both user changes.

</details>

<details><summary>Reference answer</summary>

**Answer:** Require baseVersion on commit; accept one successor and store the other as an explicit conflict version or conflict file. Use format-aware merge only for supported document types and keep recoverable history.

**Trade-offs:** Content-defined chunking reduces upload bytes for edits but adds client CPU, indexing complexity, and encryption/deduplication tradeoffs.

**Staff-level view:** Define rename atomicity across folders, deletion recovery, shared-folder permission revocation, and safe garbage collection in the presence of offline devices.

**Follow-ups:**

- Now combine this scenario with: Initial synchronization performs one request per file and takes days. How must the solution change?
- Which part of your solution protects the invariant in this related case: Initial synchronization performs one request per file and takes days.

</details>

---

### Repair competes with serving traffic

⏱ 12 min

**Scenario:** Distributed object storage: Losing a rack requires reconstructing petabytes while clients continue reading.

**Constraints:** 10PB logical data; 50M objects; 20GB/s peak read traffic. Preserve: PUT/GET/range reads; Checksums and versions.

<details><summary>Hints</summary>

1. Durability recovery consumes the same disks and network.
2. Repair priority should follow remaining redundancy.

</details>

<details><summary>Reference answer</summary>

**Answer:** Reserve repair bandwidth, prioritize under-redundant stripes, and throttle background scrubbing. Model correlated failure domains and rebuild duration; placement should spread fragments across racks before optimizing balance.

**Trade-offs:** Erasure coding lowers storage overhead but increases repair and small-write complexity; replication is simpler and often better for hot small objects.

**Staff-level view:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

**Follow-ups:**

- Now combine this scenario with: One disk returns wrong bytes without a read error. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two uploads target the same key and a reader assembles pieces from both.

</details>

---

### Silent corruption passes through replication

⏱ 10 min

**Scenario:** Distributed object storage: One disk returns wrong bytes without a read error.

**Constraints:** 10PB logical data; 50M objects; 20GB/s peak read traffic. Preserve: PUT/GET/range reads; Checksums and versions.

<details><summary>Hints</summary>

1. Replication alone can copy bad data.
2. Checksums must be verified at trust boundaries.

</details>

<details><summary>Reference answer</summary>

**Answer:** Verify end-to-end object or fragment checksums on write and read, retrieve a healthy replica or reconstruct from parity, and quarantine the bad copy. Schedule scrubbing so cold corruption is found before another failure.

**Trade-offs:** Erasure coding lowers storage overhead but increases repair and small-write complexity; replication is simpler and often better for hot small objects.

**Staff-level view:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

**Follow-ups:**

- Now combine this scenario with: Losing a rack requires reconstructing petabytes while clients continue reading. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two uploads target the same key and a reader assembles pieces from both.

</details>

---

### Concurrent overwrites return mixed fragments

⏱ 15 min

**Scenario:** Distributed object storage: Two uploads target the same key and a reader assembles pieces from both.

**Constraints:** 10PB logical data; 50M objects; 20GB/s peak read traffic. Preserve: PUT/GET/range reads; Checksums and versions.

<details><summary>Hints</summary>

1. The visible object needs one immutable manifest.
2. Object identity and key identity differ.

</details>

<details><summary>Reference answer</summary>

**Answer:** Write versioned immutable fragments and atomically replace the key-to-manifest pointer under a defined write ordering. Range reads pin a version; conditional writes prevent unintended overwrite races.

**Trade-offs:** Erasure coding lowers storage overhead but increases repair and small-write complexity; replication is simpler and often better for hot small objects.

**Staff-level view:** State consistency separately for object writes, listings, multipart completion, and cross-region replication. Do not assume all object stores or regional copies share identical guarantees.

**Follow-ups:**

- Now combine this scenario with: Losing a rack requires reconstructing petabytes while clients continue reading. How must the solution change?
- Which part of your solution protects the invariant in this related case: Losing a rack requires reconstructing petabytes while clients continue reading.

</details>

---

### Restore bandwidth misses the promised RTO

⏱ 12 min

**Scenario:** Backup and point-in-time restore: A 500TB snapshot must be restored over a 10Gb/s link.

**Constraints:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours. Preserve: Point-in-time restore; Encryption and retention.

<details><summary>Hints</summary>

1. Convert bytes into a hard minimum time.
2. Recovery objectives need measured throughput.

</details>

<details><summary>Reference answer</summary>

**Answer:** Calculate transfer and replay lower bounds, parallelize across storage partitions, and keep warm or incremental recovery capacity if needed. Reduce the restored working set only with an explicit degraded-service contract.

**Trade-offs:** Longer retention and immutable backups improve recovery options but increase cost; low RTO usually needs preprovisioned capacity rather than a cheaper archive alone.

**Staff-level view:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

**Follow-ups:**

- Now combine this scenario with: Snapshots completed successfully, yet a gap prevents replay to the requested time. How must the solution change?
- Which part of your solution protects the invariant in this related case: A cleanup task deletes an incremental base while a restore is reading it.

</details>

---

### Backup is green but missing a log segment

⏱ 10 min

**Scenario:** Backup and point-in-time restore: Snapshots completed successfully, yet a gap prevents replay to the requested time.

**Constraints:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours. Preserve: Point-in-time restore; Encryption and retention.

<details><summary>Hints</summary>

1. A file count is not a recoverable chain.
2. Continuity and checksums need validation.

</details>

<details><summary>Reference answer</summary>

**Answer:** Validate checkpoint-to-log continuity continuously, mark the recoverable time range precisely, and restore to the last verified point if a gap cannot be repaired. Run automated restore drills with application-level checks.

**Trade-offs:** Longer retention and immutable backups improve recovery options but increase cost; low RTO usually needs preprovisioned capacity rather than a cheaper archive alone.

**Staff-level view:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

**Follow-ups:**

- Now combine this scenario with: A 500TB snapshot must be restored over a 10Gb/s link. How must the solution change?
- Which part of your solution protects the invariant in this related case: A cleanup task deletes an incremental base while a restore is reading it.

</details>

---

### Retention cleanup removes an active restore dependency

⏱ 15 min

**Scenario:** Backup and point-in-time restore: A cleanup task deletes an incremental base while a restore is reading it.

**Constraints:** 500TB source data; 2TB changes/day; target RPO 5 minutes and RTO 4 hours. Preserve: Point-in-time restore; Encryption and retention.

<details><summary>Hints</summary>

1. Retention is a dependency graph.
2. Restore pins must survive retries.

</details>

<details><summary>Reference answer</summary>

**Answer:** Track snapshot and log dependencies, acquire durable restore pins, and garbage-collect only unreferenced expired chains after a safety window. Use generation-based deletion with a final reference recheck.

**Trade-offs:** Longer retention and immutable backups improve recovery options but increase cost; low RTO usually needs preprovisioned capacity rather than a cheaper archive alone.

**Staff-level view:** Test compromised credentials, key loss, and region loss as recovery scenarios. A backup success dashboard must show verified recovery points and measured restore time.

**Follow-ups:**

- Now combine this scenario with: A 500TB snapshot must be restored over a 10Gb/s link. How must the solution change?
- Which part of your solution protects the invariant in this related case: A 500TB snapshot must be restored over a 10Gb/s link.

</details>

---

### One shared blob has a hot reference counter

⏱ 12 min

**Scenario:** Content-addressed blob store: A common dependency receives millions of reference additions per minute.

**Constraints:** 3PB logical content; expected 40% duplicate bytes; 100M references/day. Preserve: Hash-verified uploads; Reference tracking.

<details><summary>Hints</summary>

1. Counters serialize a naturally many-to-one workload.
2. References are the durable truth.

</details>

<details><summary>Reference answer</summary>

**Answer:** Shard reference records and compute liveness through indexed existence or periodic marking instead of a single synchronous global counter. Batch accounting and partition work by digest prefix while keeping reference identity unique.

**Trade-offs:** Global deduplication saves more storage but complicates isolation, encryption, accounting, and deletion guarantees compared with tenant-scoped deduplication.

**Staff-level view:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

**Follow-ups:**

- Now combine this scenario with: An unreferenced blob is selected for deletion just before a new tenant links it. How must the solution change?
- Which part of your solution protects the invariant in this related case: A tenant can probe whether another tenant uploaded a known file.

</details>

---

### Garbage collector races with a new reference

⏱ 10 min

**Scenario:** Content-addressed blob store: An unreferenced blob is selected for deletion just before a new tenant links it.

**Constraints:** 3PB logical content; expected 40% duplicate bytes; 100M references/day. Preserve: Hash-verified uploads; Reference tracking.

<details><summary>Hints</summary>

1. Selection and deletion are separate moments.
2. Use a grace period plus a generation check.

</details>

<details><summary>Reference answer</summary>

**Answer:** Mark candidates, delay physical removal, and atomically recheck the reference generation before final deletion. New references either cancel deletion or require a verified re-upload; never trust a stale zero count.

**Trade-offs:** Global deduplication saves more storage but complicates isolation, encryption, accounting, and deletion guarantees compared with tenant-scoped deduplication.

**Staff-level view:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

**Follow-ups:**

- Now combine this scenario with: A common dependency receives millions of reference additions per minute. How must the solution change?
- Which part of your solution protects the invariant in this related case: A tenant can probe whether another tenant uploaded a known file.

</details>

---

### A guessed digest reveals private content

⏱ 15 min

**Scenario:** Content-addressed blob store: A tenant can probe whether another tenant uploaded a known file.

**Constraints:** 3PB logical content; expected 40% duplicate bytes; 100M references/day. Preserve: Hash-verified uploads; Reference tracking.

<details><summary>Hints</summary>

1. Deduplication may expose equality.
2. Authorization must not follow knowledge of a hash.

</details>

<details><summary>Reference answer</summary>

**Answer:** Require proof of content possession or scope deduplication by tenant, and authorize references independently of the digest. Avoid existence endpoints that reveal cross-tenant content and assess encryption requirements before global dedupe.

**Trade-offs:** Global deduplication saves more storage but complicates isolation, encryption, accounting, and deletion guarantees compared with tenant-scoped deduplication.

**Staff-level view:** Prove collection safety under concurrent references, delayed replicas, interrupted sweeps, and hash-verification failures; measure storage saved after accounting for metadata and replication.

**Follow-ups:**

- Now combine this scenario with: A common dependency receives millions of reference additions per minute. How must the solution change?
- Which part of your solution protects the invariant in this related case: A common dependency receives millions of reference additions per minute.

</details>

---

### Small files overwhelm namespace memory

⏱ 12 min

**Scenario:** Distributed filesystem: A billion tiny files fit on disks but exhaust metadata servers.

**Constraints:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads. Preserve: Atomic namespace operations; Chunk replication.

<details><summary>Hints</summary>

1. Metadata bytes per inode can dominate.
2. Namespace partitioning has transaction costs.

</details>

<details><summary>Reference answer</summary>

**Answer:** Measure inode and directory-index overhead, partition namespace metadata by stable subtree or inode ranges, and batch client metadata operations. Define the behavior of cross-partition rename before distributing the namespace.

**Trade-offs:** Fine-grained metadata sharding increases capacity but makes atomic rename and directory listings harder; central metadata is simpler until measured limits require partitioning.

**Staff-level view:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

**Follow-ups:**

- Now combine this scenario with: A paused writer resumes after a new client became the file owner. How must the solution change?
- Which part of your solution protects the invariant in this related case: Moving a directory between shards must not expose it twice or lose it.

</details>

---

### A stale client writes after lease transfer

⏱ 10 min

**Scenario:** Distributed filesystem: A paused writer resumes after a new client became the file owner.

**Constraints:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads. Preserve: Atomic namespace operations; Chunk replication.

<details><summary>Hints</summary>

1. Expired leases do not physically stop old clients.
2. Storage nodes must reject stale epochs.

</details>

<details><summary>Reference answer</summary>

**Answer:** Attach a monotonic fencing epoch to writes and have data nodes reject epochs older than the current file lease. Recover or truncate uncommitted tails according to the documented append contract.

**Trade-offs:** Fine-grained metadata sharding increases capacity but makes atomic rename and directory listings harder; central metadata is simpler until measured limits require partitioning.

**Staff-level view:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

**Follow-ups:**

- Now combine this scenario with: A billion tiny files fit on disks but exhaust metadata servers. How must the solution change?
- Which part of your solution protects the invariant in this related case: Moving a directory between shards must not expose it twice or lose it.

</details>

---

### Rename crosses metadata shards

⏱ 15 min

**Scenario:** Distributed filesystem: Moving a directory between shards must not expose it twice or lose it.

**Constraints:** 1B files; 5PB data; 100K concurrent clients; most operations are metadata reads. Preserve: Atomic namespace operations; Chunk replication.

<details><summary>Hints</summary>

1. Two name entries form one invariant.
2. You may need a transaction or constrained semantics.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a recoverable cross-shard transaction or durable rename intent with an authoritative namespace state machine. Alternatively restrict atomic rename to one namespace partition and expose the restriction clearly.

**Trade-offs:** Fine-grained metadata sharding increases capacity but makes atomic rename and directory listings harder; central metadata is simpler until measured limits require partitioning.

**Staff-level view:** Write down the exact POSIX subset, append semantics, client-cache invalidation rules, and behavior during metadata quorum loss before drawing data-node boxes.

**Follow-ups:**

- Now combine this scenario with: A billion tiny files fit on disks but exhaust metadata servers. How must the solution change?
- Which part of your solution protects the invariant in this related case: A billion tiny files fit on disks but exhaust metadata servers.

</details>

---

## Commerce

### A sale overwhelms inventory reservation

⏱ 12 min

**Scenario:** Commerce checkout: Most checkouts target one scarce SKU while the rest of the store is healthy.

**Constraints:** 5M orders/day; flash-sale peak 20K checkout attempts/s. Preserve: Stable checkout retries; Inventory reservation.

<details><summary>Hints</summary>

1. Scarce stock is a serialization boundary.
2. Admission should happen before expensive payment work.

</details>

<details><summary>Reference answer</summary>

**Answer:** Admit attempts through a bounded SKU queue, reserve stock atomically, and cap per-user attempts. Keep unrelated products on independent partitions and return sold-out promptly once allocatable stock is exhausted.

**Trade-offs:** Long inventory holds improve checkout completion but reduce availability to other buyers; short holds require well-defined late-payment compensation.

**Staff-level view:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

**Follow-ups:**

- Now combine this scenario with: The provider captured funds while the checkout process crashed before recording success. How must the solution change?
- Which part of your solution protects the invariant in this related case: The inventory hold expires while a delayed payment success arrives.

</details>

---

### Payment succeeded but the order timed out

⏱ 10 min

**Scenario:** Commerce checkout: The provider captured funds while the checkout process crashed before recording success.

**Constraints:** 5M orders/day; flash-sale peak 20K checkout attempts/s. Preserve: Stable checkout retries; Inventory reservation.

<details><summary>Hints</summary>

1. A timeout is an unknown payment state.
2. Provider identity must survive retry.

</details>

<details><summary>Reference answer</summary>

**Answer:** Persist a payment attempt with a stable provider idempotency key before calling the provider. Reconcile callbacks and provider lookup into a monotonic order state machine; never create a fresh charge merely because the client retried.

**Trade-offs:** Long inventory holds improve checkout completion but reduce availability to other buyers; short holds require well-defined late-payment compensation.

**Staff-level view:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

**Follow-ups:**

- Now combine this scenario with: Most checkouts target one scarce SKU while the rest of the store is healthy. How must the solution change?
- Which part of your solution protects the invariant in this related case: The inventory hold expires while a delayed payment success arrives.

</details>

---

### A reservation expires as payment completes

⏱ 15 min

**Scenario:** Commerce checkout: The inventory hold expires while a delayed payment success arrives.

**Constraints:** 5M orders/day; flash-sale peak 20K checkout attempts/s. Preserve: Stable checkout retries; Inventory reservation.

<details><summary>Hints</summary>

1. No local transaction spans an external provider.
2. The saga needs a compensation rule.

</details>

<details><summary>Reference answer</summary>

**Answer:** Define a deadline and durable state machine: either extend a valid hold before capture, or compensate a late capture through refund when stock cannot be reacquired. Serialize state transitions and make refund requests idempotent.

**Trade-offs:** Long inventory holds improve checkout completion but reduce availability to other buyers; short holds require well-defined late-payment compensation.

**Staff-level view:** Measure unknown payments, reservation age, refund backlog, and reconciliation lag alongside checkout success. Write operational playbooks that preserve evidence for ambiguous money movement.

**Follow-ups:**

- Now combine this scenario with: Most checkouts target one scarce SKU while the rest of the store is healthy. How must the solution change?
- Which part of your solution protects the invariant in this related case: Most checkouts target one scarce SKU while the rest of the store is healthy.

</details>

---

### Hot SKU saturates one row lock

⏱ 12 min

**Scenario:** Inventory reservation: Thousands of buyers contend for the last thousand units.

**Constraints:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s. Preserve: Prevent overselling; Expiring reservations.

<details><summary>Hints</summary>

1. One exact stock balance serializes mutations.
2. Allocation can be split without inventing stock.

</details>

<details><summary>Reference answer</summary>

**Answer:** Allocate bounded stock quotas to independent reservation buckets or regions and replenish through a single authority. Preserve the invariant that allocated quotas plus central stock never exceed real stock; reject when a local quota is exhausted.

**Trade-offs:** Regional quota allocation reduces contention and latency but can strand stock in idle regions; rebalancing must preserve ownership and prevent double allocation.

**Staff-level view:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

**Follow-ups:**

- Now combine this scenario with: A hold is committed while an expiry worker also releases it. How must the solution change?
- Which part of your solution protects the invariant in this related case: A duplicated shipment event deducts stock twice.

</details>

---

### Expiry and checkout both decrement reservations

⏱ 10 min

**Scenario:** Inventory reservation: A hold is committed while an expiry worker also releases it.

**Constraints:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s. Preserve: Prevent overselling; Expiring reservations.

<details><summary>Hints</summary>

1. Both paths act on the same state transition.
2. Counters should change only with a successful transition.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a transactional compare-and-set from held to committed or expired, with stock counters updated in that same transaction. Exactly one transition wins; retries return the recorded result.

**Trade-offs:** Regional quota allocation reduces contention and latency but can strand stock in idle regions; rebalancing must preserve ownership and prevent double allocation.

**Staff-level view:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

**Follow-ups:**

- Now combine this scenario with: Thousands of buyers contend for the last thousand units. How must the solution change?
- Which part of your solution protects the invariant in this related case: A duplicated shipment event deducts stock twice.

</details>

---

### Warehouse event is delivered twice

⏱ 15 min

**Scenario:** Inventory reservation: A duplicated shipment event deducts stock twice.

**Constraints:** 20M SKUs; 50M stock mutations/day; a hot SKU may receive 50K attempts/s. Preserve: Prevent overselling; Expiring reservations.

<details><summary>Hints</summary>

1. A stock delta needs identity.
2. Deduplication belongs in the ledger transaction.

</details>

<details><summary>Reference answer</summary>

**Answer:** Record a unique warehouse event ID with the stock ledger mutation and reject repeated application. Reconcile physical counts through explicit adjustment entries rather than overwriting balances silently.

**Trade-offs:** Regional quota allocation reduces contention and latency but can strand stock in idle regions; rebalancing must preserve ownership and prevent double allocation.

**Staff-level view:** Define invariants for damaged goods, returns, transfers in flight, backorders, and manual adjustments instead of treating stock as one mutable integer.

**Follow-ups:**

- Now combine this scenario with: Thousands of buyers contend for the last thousand units. How must the solution change?
- Which part of your solution protects the invariant in this related case: Thousands of buyers contend for the last thousand units.

</details>

---

### A settlement account becomes a hot row

⏱ 12 min

**Scenario:** Payment ledger: Every merchant payment updates the same platform settlement balance.

**Constraints:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units. Preserve: Balanced double-entry transactions; Idempotent posting.

<details><summary>Hints</summary>

1. Append throughput and balance-row contention differ.
2. Balance reads may be materialized.

</details>

<details><summary>Reference answer</summary>

**Answer:** Append balanced entries transactionally without forcing all traffic through one synchronously updated aggregate row. Partition settlement subaccounts and reconcile to a parent total; retain strict checks where available-funds authorization requires them.

**Trade-offs:** Synchronous balance projections simplify immediate reads but add contention; asynchronous projections need freshness markers and cannot authorize overspend alone.

**Staff-level view:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

**Follow-ups:**

- Now combine this scenario with: A client retries after a lost response and creates duplicate debit-credit pairs. How must the solution change?
- Which part of your solution protects the invariant in this related case: A multi-currency operation sums decimal approximations across entries.

</details>

---

### Retry posts the same transfer twice

⏱ 10 min

**Scenario:** Payment ledger: A client retries after a lost response and creates duplicate debit-credit pairs.

**Constraints:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units. Preserve: Balanced double-entry transactions; Idempotent posting.

<details><summary>Hints</summary>

1. Balanced duplicates still lose money.
2. Uniqueness must include request semantics.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a durable unique idempotency key and compare a canonical payload hash on reuse. Return the original transaction for identical retries and reject mismatched payloads; keep the dedupe window aligned with actual retry and replay behavior.

**Trade-offs:** Synchronous balance projections simplify immediate reads but add contention; asynchronous projections need freshness markers and cannot authorize overspend alone.

**Staff-level view:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

**Follow-ups:**

- Now combine this scenario with: Every merchant payment updates the same platform settlement balance. How must the solution change?
- Which part of your solution protects the invariant in this related case: A multi-currency operation sums decimal approximations across entries.

</details>

---

### Currency rounding breaks ledger balance

⏱ 15 min

**Scenario:** Payment ledger: A multi-currency operation sums decimal approximations across entries.

**Constraints:** 10M ledger transactions/day; 7-year business retention assumption; exact integer minor units. Preserve: Balanced double-entry transactions; Idempotent posting.

<details><summary>Hints</summary>

1. Balances are per currency.
2. FX needs explicit conversion entries.

</details>

<details><summary>Reference answer</summary>

**Answer:** Represent amounts as integer minor units with currency metadata, require zero-sum entries within each currency, and record FX legs, rates, and rounding accounts explicitly. Reverse through new linked entries instead of editing history.

**Trade-offs:** Synchronous balance projections simplify immediate reads but add contention; asynchronous projections need freshness markers and cannot authorize overspend alone.

**Staff-level view:** Separate ledger truth, available balance, provider settlement, and bank reconciliation. Treat recovery, access control, and immutable audit evidence as design requirements rather than afterthoughts.

**Follow-ups:**

- Now combine this scenario with: Every merchant payment updates the same platform settlement balance. How must the solution change?
- Which part of your solution protects the invariant in this related case: Every merchant payment updates the same platform settlement balance.

</details>

---

### Seat-map refresh melts the backend

⏱ 12 min

**Scenario:** Ticket booking: Three million waiting users poll the full seat map every second.

**Constraints:** 100K seats for one event; 3M buyers arrive in a minute. Preserve: One owner per seat; Hold expiry.

<details><summary>Hints</summary>

1. Most map data is static.
2. Exact ownership matters at hold time.

</details>

<details><summary>Reference answer</summary>

**Answer:** Serve static geometry via CDN and bounded availability deltas via polling or push cohorts. Rate-limit refresh, cache approximate availability briefly, and validate exact seat ownership only inside the hold transaction.

**Trade-offs:** Strict queue fairness can reduce throughput; opportunistic admission improves utilization but must not undermine the advertised purchase policy.

**Staff-level view:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

**Follow-ups:**

- Now combine this scenario with: An old hold-expiry task runs after payment confirmation. How must the solution change?
- Which part of your solution protects the invariant in this related case: Each group obtains some seats and neither can complete the requested block.

</details>

---

### A timer releases a newly confirmed seat

⏱ 10 min

**Scenario:** Ticket booking: An old hold-expiry task runs after payment confirmation.

**Constraints:** 100K seats for one event; 3M buyers arrive in a minute. Preserve: One owner per seat; Hold expiry.

<details><summary>Hints</summary>

1. Scheduled work can arrive late.
2. A state and version check must guard release.

</details>

<details><summary>Reference answer</summary>

**Answer:** Condition expiry on matching hold ID, held state, and expected version. Confirm and expire compete atomically; late expiry becomes a no-op and cannot free a sold seat.

**Trade-offs:** Strict queue fairness can reduce throughput; opportunistic admission improves utilization but must not undermine the advertised purchase policy.

**Staff-level view:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

**Follow-ups:**

- Now combine this scenario with: Three million waiting users poll the full seat map every second. How must the solution change?
- Which part of your solution protects the invariant in this related case: Each group obtains some seats and neither can complete the requested block.

</details>

---

### Two groups partially acquire adjacent seats

⏱ 15 min

**Scenario:** Ticket booking: Each group obtains some seats and neither can complete the requested block.

**Constraints:** 100K seats for one event; 3M buyers arrive in a minute. Preserve: One owner per seat; Hold expiry.

<details><summary>Hints</summary>

1. The requested seat set is the atomic unit.
2. Lock ordering affects deadlocks.

</details>

<details><summary>Reference answer</summary>

**Answer:** Reserve the full seat set in one transaction using a deterministic lock order, or return failure without partial holds. If multi-shard seat groups are unavoidable, define an explicit short-lived coordination protocol and cleanup.

**Trade-offs:** Strict queue fairness can reduce throughput; opportunistic admission improves utilization but must not undermine the advertised purchase policy.

**Staff-level view:** Model bot resistance, waiting-room token replay, queue position continuity during failover, and the user-visible policy for ambiguous payment completion.

**Follow-ups:**

- Now combine this scenario with: Three million waiting users poll the full seat map every second. How must the solution change?
- Which part of your solution protects the invariant in this related case: Three million waiting users poll the full seat map every second.

</details>

---

### A slow seller blocks every order item

⏱ 12 min

**Scenario:** Marketplace orders: One seller integration is unavailable while other items are ready to ship.

**Constraints:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks. Preserve: Split fulfillment; Partial cancellation.

<details><summary>Hints</summary>

1. An order is a collection of independently progressing items.
2. Shared totals still require reconciliation.

</details>

<details><summary>Reference answer</summary>

**Answer:** Run per-seller or per-item fulfillment state machines and aggregate order status. Isolate adapter queues and show partial progress; keep discounts, shipping allocation, and refunds tied to stable item-level amounts.

**Trade-offs:** Independent item workflows improve resilience but make totals, customer communication, and compensation more complex than a single order status.

**Staff-level view:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

**Follow-ups:**

- Now combine this scenario with: A delayed in-transit callback arrives after delivered and triggers another payout hold. How must the solution change?
- Which part of your solution protects the invariant in this related case: A returned item becomes refundable just as its seller payout is released.

</details>

---

### Shipment callback moves an item backwards

⏱ 10 min

**Scenario:** Marketplace orders: A delayed in-transit callback arrives after delivered and triggers another payout hold.

**Constraints:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks. Preserve: Split fulfillment; Partial cancellation.

<details><summary>Hints</summary>

1. Arrival order is not lifecycle order.
2. Carrier events need versions or monotonic rules.

</details>

<details><summary>Reference answer</summary>

**Answer:** Deduplicate carrier events, validate allowed state transitions, and preserve raw evidence. Ignore superseded updates while escalating genuine corrections through a separate correction workflow.

**Trade-offs:** Independent item workflows improve resilience but make totals, customer communication, and compensation more complex than a single order status.

**Staff-level view:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

**Follow-ups:**

- Now combine this scenario with: One seller integration is unavailable while other items are ready to ship. How must the solution change?
- Which part of your solution protects the invariant in this related case: A returned item becomes refundable just as its seller payout is released.

</details>

---

### Refund and payout race

⏱ 15 min

**Scenario:** Marketplace orders: A returned item becomes refundable just as its seller payout is released.

**Constraints:** 2M orders/day; average 3 sellers/order; fulfillment can take weeks. Preserve: Split fulfillment; Partial cancellation.

<details><summary>Hints</summary>

1. Funds ownership spans two workflows.
2. A shared eligibility decision needs serialization.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a transactional payout-eligibility record linked to item and return state. Freeze eligible amounts before payout submission and compensate late returns with explicit seller receivables rather than editing settled ledger entries.

**Trade-offs:** Independent item workflows improve resilience but make totals, customer communication, and compensation more complex than a single order status.

**Staff-level view:** Define operational ownership for stuck states and disputed deliveries, and make seller-specific failures visible without exposing unrelated customer data.

**Follow-ups:**

- Now combine this scenario with: One seller integration is unavailable while other items are ready to ship. How must the solution change?
- Which part of your solution protects the invariant in this related case: One seller integration is unavailable while other items are ready to ship.

</details>

---

## Real Time

### Airport arrivals overload one map cell

⏱ 12 min

**Scenario:** Ride dispatch: Thousands of riders and drivers concentrate in one geospatial bucket.

**Constraints:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds. Preserve: Fresh driver locations; Exclusive driver assignment.

<details><summary>Hints</summary>

1. Geographic partitioning creates natural hot spots.
2. Candidate search and assignment have different ownership.

</details>

<details><summary>Reference answer</summary>

**Answer:** Subdivide hot cells, cap candidate sets, and apply queueing or ranked batches for airport dispatch. Keep driver assignment conditional by driver ID even when candidate search fans across multiple spatial partitions.

**Trade-offs:** Frequent location updates improve matching but consume battery and ingest capacity; adaptive frequency can depend on trip state and movement.

**Staff-level view:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

**Follow-ups:**

- Now combine this scenario with: A driver went offline but remains in the nearby-driver index. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two matchers choose the same available driver from a stale index.

</details>

---

### Stale locations produce impossible pickups

⏱ 10 min

**Scenario:** Ride dispatch: A driver went offline but remains in the nearby-driver index.

**Constraints:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds. Preserve: Fresh driver locations; Exclusive driver assignment.

<details><summary>Hints</summary>

1. Location age is part of eligibility.
2. Out-of-order updates must not freshen old data.

</details>

<details><summary>Reference answer</summary>

**Answer:** Require monotonic per-device sequences and server-observed freshness bounds; filter stale candidates and expire presence. Request confirmation before assignment and monitor stale-candidate rejection and pickup ETA error.

**Trade-offs:** Frequent location updates improve matching but consume battery and ingest capacity; adaptive frequency can depend on trip state and movement.

**Staff-level view:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

**Follow-ups:**

- Now combine this scenario with: Thousands of riders and drivers concentrate in one geospatial bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two matchers choose the same available driver from a stale index.

</details>

---

### Two riders get the same driver

⏱ 15 min

**Scenario:** Ride dispatch: Two matchers choose the same available driver from a stale index.

**Constraints:** 2M active drivers; location updates every 5 seconds; dispatch target under 2 seconds. Preserve: Fresh driver locations; Exclusive driver assignment.

<details><summary>Hints</summary>

1. Search results are advisory.
2. The claim must be atomic.

</details>

<details><summary>Reference answer</summary>

**Answer:** Conditionally transition the driver from available to offered with a unique offer ID and expiry. Acceptance references that offer; late or duplicate acceptances cannot claim a different trip.

**Trade-offs:** Frequent location updates improve matching but consume battery and ingest capacity; adaptive frequency can depend on trip state and movement.

**Staff-level view:** Prove assignment safety across retries and regional ownership changes, and distinguish dispatch latency from location freshness and driver acceptance rate.

**Follow-ups:**

- Now combine this scenario with: Thousands of riders and drivers concentrate in one geospatial bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: Thousands of riders and drivers concentrate in one geospatial bucket.

</details>

---

### A giant document has years of operations

⏱ 12 min

**Scenario:** Collaborative document editor: Opening a document replays five million edits and freezes the client.

**Constraints:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB. Preserve: Convergent edits; Reconnect and replay.

<details><summary>Hints</summary>

1. Log history and current state have different needs.
2. Compaction must preserve offline merge metadata.

</details>

<details><summary>Reference answer</summary>

**Answer:** Create versioned snapshots with a replay checkpoint, bound startup replay, and compact only metadata no longer needed by supported offline clients. Define a maximum offline horizon and resync protocol for older clients.

**Trade-offs:** CRDTs help offline convergence but carry metadata and semantic complexity; OT can be compact but relies on carefully managed transformation context.

**Staff-level view:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

**Follow-ups:**

- Now combine this scenario with: Two edits at the same position produce different text depending on arrival order. How must the solution change?
- Which part of your solution protects the invariant in this related case: A former collaborator reconnects with edits made while disconnected.

</details>

---

### Concurrent insertions diverge across clients

⏱ 10 min

**Scenario:** Collaborative document editor: Two edits at the same position produce different text depending on arrival order.

**Constraints:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB. Preserve: Convergent edits; Reconnect and replay.

<details><summary>Hints</summary>

1. Index positions change as edits apply.
2. Convergence requires a chosen algorithm.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a specified OT or CRDT algorithm with stable operation IDs and causal/version metadata, rather than last-write-wins on whole text. Reproduce the operation schedule and test that all valid delivery orders converge.

**Trade-offs:** CRDTs help offline convergence but carry metadata and semantic complexity; OT can be compact but relies on carefully managed transformation context.

**Staff-level view:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

**Follow-ups:**

- Now combine this scenario with: Opening a document replays five million edits and freezes the client. How must the solution change?
- Which part of your solution protects the invariant in this related case: A former collaborator reconnects with edits made while disconnected.

</details>

---

### Revoked editor uploads offline operations

⏱ 15 min

**Scenario:** Collaborative document editor: A former collaborator reconnects with edits made while disconnected.

**Constraints:** 1M concurrent editors; 5 operations/s per active editor; documents up to 10MB. Preserve: Convergent edits; Reconnect and replay.

<details><summary>Hints</summary>

1. Locally valid operations may no longer be authorized.
2. Access is checked at acceptance time.

</details>

<details><summary>Reference answer</summary>

**Answer:** Authenticate every resumed session and validate current write permission before durable acceptance. Reject unauthorized operations with a recoverable local export path; never merge first and remove later.

**Trade-offs:** CRDTs help offline convergence but carry metadata and semantic complexity; OT can be compact but relies on carefully managed transformation context.

**Staff-level view:** Specify supported edit semantics, undo behavior across collaborators, offline horizon, and snapshot compatibility before choosing a merge algorithm.

**Follow-ups:**

- Now combine this scenario with: Opening a document replays five million edits and freezes the client. How must the solution change?
- Which part of your solution protects the invariant in this related case: Opening a document replays five million edits and freezes the client.

</details>

---

### Reconnect storm floods heartbeats

⏱ 12 min

**Scenario:** Presence and typing indicators: A regional network outage ends and 10M devices reconnect at once.

**Constraints:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds. Preserve: Per-device heartbeats; Multi-device aggregation.

<details><summary>Hints</summary>

1. Synchronized retries create secondary outages.
2. Presence can degrade without blocking chat.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use exponential backoff with jitter, admission control, and staggered initial heartbeats. Drop redundant typing signals and batch presence transitions; protect durable chat traffic with separate resource pools.

**Trade-offs:** Short heartbeat intervals improve freshness but increase battery and traffic; presence accuracy should be bounded and approximate by product contract.

**Staff-level view:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

**Follow-ups:**

- Now combine this scenario with: A mobile device loses network without closing its socket. How must the solution change?
- Which part of your solution protects the invariant in this related case: A delayed close event for session A marks session B offline.

</details>

---

### User remains online after a silent disconnect

⏱ 10 min

**Scenario:** Presence and typing indicators: A mobile device loses network without closing its socket.

**Constraints:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds. Preserve: Per-device heartbeats; Multi-device aggregation.

<details><summary>Hints</summary>

1. Disconnect events are best effort.
2. Online is a freshness estimate.

</details>

<details><summary>Reference answer</summary>

**Answer:** Derive online status from expiring heartbeats and aggregate across active device sessions. Use server time for expiry, expire silent devices, and display approximate last-seen state rather than claiming exact liveness.

**Trade-offs:** Short heartbeat intervals improve freshness but increase battery and traffic; presence accuracy should be bounded and approximate by product contract.

**Staff-level view:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

**Follow-ups:**

- Now combine this scenario with: A regional network outage ends and 10M devices reconnect at once. How must the solution change?
- Which part of your solution protects the invariant in this related case: A delayed close event for session A marks session B offline.

</details>

---

### Old disconnect clears a new session

⏱ 15 min

**Scenario:** Presence and typing indicators: A delayed close event for session A marks session B offline.

**Constraints:** 30M connected devices; heartbeat every 30 seconds; typing expiry after 5 seconds. Preserve: Per-device heartbeats; Multi-device aggregation.

<details><summary>Hints</summary>

1. Connection identity matters.
2. Compare the current session epoch.

</details>

<details><summary>Reference answer</summary>

**Answer:** Condition disconnect cleanup on the matching device session epoch. Maintain independent per-device state and report the user online while any unexpired authorized device session exists.

**Trade-offs:** Short heartbeat intervals improve freshness but increase battery and traffic; presence accuracy should be bounded and approximate by product contract.

**Staff-level view:** Budget fanout by watchers and privacy rules, not just connected users, and isolate presence outages from message delivery.

**Follow-ups:**

- Now combine this scenario with: A regional network outage ends and 10M devices reconnect at once. How must the solution change?
- Which part of your solution protects the invariant in this related case: A regional network outage ends and 10M devices reconnect at once.

</details>

---

### One global sorted set runs out of headroom

⏱ 12 min

**Scenario:** Game leaderboard: Fifty million players and constant writes overload one ranking shard.

**Constraints:** 50M players; 200M score events/day; top-100 reads dominate. Preserve: Top K and user rank; Idempotent score events.

<details><summary>Hints</summary>

1. Exact global rank is harder than top K.
2. Partial top lists can merge cheaply.

</details>

<details><summary>Reference answer</summary>

**Answer:** Partition player scores and merge shard top-K lists for global leaders. Serve exact rank through a separate periodically built order-statistics view or accept bounded approximation; declare the freshness and accuracy separately.

**Trade-offs:** Exact live global rank requires expensive coordination or indexing; top-K and approximate personal rank can scale independently.

**Staff-level view:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

**Follow-ups:**

- Now combine this scenario with: A consumer replay applies the same match result twice. How must the solution change?
- Which part of your solution protects the invariant in this related case: A valid match ends before midnight but arrives after the season is finalized.

</details>

---

### Replayed score event doubles a win

⏱ 10 min

**Scenario:** Game leaderboard: A consumer replay applies the same match result twice.

**Constraints:** 50M players; 200M score events/day; top-100 reads dominate. Preserve: Top K and user rank; Idempotent score events.

<details><summary>Hints</summary>

1. Score deltas are not naturally idempotent.
2. A stable match-result ID is available.

</details>

<details><summary>Reference answer</summary>

**Answer:** Deduplicate the event in the same durable update as the score mutation, or rebuild scores from unique validated results. Keep an auditable event log so incorrect projections can be discarded and recomputed.

**Trade-offs:** Exact live global rank requires expensive coordination or indexing; top-K and approximate personal rank can scale independently.

**Staff-level view:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

**Follow-ups:**

- Now combine this scenario with: Fifty million players and constant writes overload one ranking shard. How must the solution change?
- Which part of your solution protects the invariant in this related case: A valid match ends before midnight but arrives after the season is finalized.

</details>

---

### Late scores cross a season boundary

⏱ 15 min

**Scenario:** Game leaderboard: A valid match ends before midnight but arrives after the season is finalized.

**Constraints:** 50M players; 200M score events/day; top-100 reads dominate. Preserve: Top K and user rank; Idempotent score events.

<details><summary>Hints</summary>

1. Event time and arrival time differ.
2. Finality requires a lateness policy.

</details>

<details><summary>Reference answer</summary>

**Answer:** Assign the season from authoritative match metadata, keep a bounded grace window, and finalize at a recorded checkpoint. Route later corrections through an explicit revision process rather than silently changing awarded rankings.

**Trade-offs:** Exact live global rank requires expensive coordination or indexing; top-K and approximate personal rank can scale independently.

**Staff-level view:** Define tie-breaking, anti-cheat review, prize finality, and late correction policy as part of ranking correctness.

**Follow-ups:**

- Now combine this scenario with: Fifty million players and constant writes overload one ranking shard. How must the solution change?
- Which part of your solution protects the invariant in this related case: Fifty million players and constant writes overload one ranking shard.

</details>

---

### A map zoom returns every device

⏱ 12 min

**Scenario:** Fleet tracking: A large customer opens a world map containing one million vehicles.

**Constraints:** 5M devices; update every 10 seconds while moving; 90-day history. Preserve: Out-of-order GPS handling; Current location and history.

<details><summary>Hints</summary>

1. Screen pixels bound useful detail.
2. Map queries should match zoom level.

</details>

<details><summary>Reference answer</summary>

**Answer:** Serve spatial clusters or tiles at low zoom, fetch individual positions only within a bounded viewport, and stream deltas for visible devices. Separate latest-state queries from historical scans and enforce per-tenant query budgets.

**Trade-offs:** Keeping every raw point aids audits and analytics but increases storage; downsampling must preserve the fidelity required for billing or route evidence.

**Staff-level view:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

**Follow-ups:**

- Now combine this scenario with: A parked truck near a boundary generates dozens of geofence transitions. How must the solution change?
- Which part of your solution protects the invariant in this related case: A device reconnects and uploads yesterday’s points after a fresh live update.

</details>

---

### GPS jitter creates repeated enter-exit alerts

⏱ 10 min

**Scenario:** Fleet tracking: A parked truck near a boundary generates dozens of geofence transitions.

**Constraints:** 5M devices; update every 10 seconds while moving; 90-day history. Preserve: Out-of-order GPS handling; Current location and history.

<details><summary>Hints</summary>

1. Measurement noise is not true movement.
2. Hysteresis and dwell time reduce oscillation.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use entry/exit buffers, minimum dwell time, and accuracy-aware filtering. Deduplicate alert transitions by fence state version and preserve raw points for diagnosis instead of treating every crossing as certain.

**Trade-offs:** Keeping every raw point aids audits and analytics but increases storage; downsampling must preserve the fidelity required for billing or route evidence.

**Staff-level view:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

**Follow-ups:**

- Now combine this scenario with: A large customer opens a world map containing one million vehicles. How must the solution change?
- Which part of your solution protects the invariant in this related case: A device reconnects and uploads yesterday’s points after a fresh live update.

</details>

---

### Offline backlog overwrites the latest position

⏱ 15 min

**Scenario:** Fleet tracking: A device reconnects and uploads yesterday’s points after a fresh live update.

**Constraints:** 5M devices; update every 10 seconds while moving; 90-day history. Preserve: Out-of-order GPS handling; Current location and history.

<details><summary>Hints</summary>

1. Ingestion time is not observation order.
2. Device sequence and reboot epoch are useful.

</details>

<details><summary>Reference answer</summary>

**Answer:** Store all accepted historical points but condition latest-position updates on a newer device epoch/sequence or validated event-time policy. Handle clock anomalies explicitly and prevent stale replay from triggering current-location alerts.

**Trade-offs:** Keeping every raw point aids audits and analytics but increases storage; downsampling must preserve the fidelity required for billing or route evidence.

**Staff-level view:** Define clock-drift tolerance, device identity reset, tenant isolation, and the difference between real-time safety alerts and best-effort business notifications.

**Follow-ups:**

- Now combine this scenario with: A large customer opens a world map containing one million vehicles. How must the solution change?
- Which part of your solution protects the invariant in this related case: A large customer opens a world map containing one million vehicles.

</details>

---

## Search

### Common query fans out to every shard

⏱ 12 min

**Scenario:** Web search engine: A short query matches billions of postings and exhausts tail latency.

**Constraints:** 10B pages; 100M queries/day; p95 search target 300 ms. Preserve: Polite crawling; Freshness and deduplication.

<details><summary>Hints</summary>

1. Candidate pruning comes before expensive ranking.
2. Scatter-gather tails dominate.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use tiered indexes, posting-list pruning, bounded candidate retrieval, and cached popular queries. Apply per-shard deadlines and partial-result policy; rank a limited candidate set with expensive features.

**Trade-offs:** Fresh indexing consumes resources that could improve query latency; tier pages by change rate and query value instead of recrawling everything uniformly.

**Staff-level view:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

**Follow-ups:**

- Now combine this scenario with: A site generates unlimited distinct URLs with equivalent low-value pages. How must the solution change?
- Which part of your solution protects the invariant in this related case: The source is gone but an old index generation still serves it.

</details>

---

### Crawler falls into an infinite calendar

⏱ 10 min

**Scenario:** Web search engine: A site generates unlimited distinct URLs with equivalent low-value pages.

**Constraints:** 10B pages; 100M queries/day; p95 search target 300 ms. Preserve: Polite crawling; Freshness and deduplication.

<details><summary>Hints</summary>

1. URL cardinality is adversarially unbounded.
2. Host budgets and canonicalization constrain exploration.

</details>

<details><summary>Reference answer</summary>

**Answer:** Cap host and pattern crawl budgets, canonicalize known query parameters, detect content duplicates, and stop low-yield URL families. Respect crawl policies and avoid letting one host consume the frontier.

**Trade-offs:** Fresh indexing consumes resources that could improve query latency; tier pages by change rate and query value instead of recrawling everything uniformly.

**Staff-level view:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

**Follow-ups:**

- Now combine this scenario with: A short query matches billions of postings and exhausts tail latency. How must the solution change?
- Which part of your solution protects the invariant in this related case: The source is gone but an old index generation still serves it.

</details>

---

### Deleted pages remain in search results

⏱ 15 min

**Scenario:** Web search engine: The source is gone but an old index generation still serves it.

**Constraints:** 10B pages; 100M queries/day; p95 search target 300 ms. Preserve: Polite crawling; Freshness and deduplication.

<details><summary>Hints</summary>

1. Fetch freshness and index publication are separate.
2. Tombstones must survive index rebuilds.

</details>

<details><summary>Reference answer</summary>

**Answer:** Record durable removal tombstones with versions, apply them to serving filters and future index builds, and atomically swap validated index generations. Distinguish temporary fetch errors from confirmed removals.

**Trade-offs:** Fresh indexing consumes resources that could improve query latency; tier pages by change rate and query value instead of recrawling everything uniformly.

**Staff-level view:** Set budgets for crawling, indexing freshness, query tail latency, and removal propagation, and plan index-generation rollback with durable tombstone preservation.

**Follow-ups:**

- Now combine this scenario with: A short query matches billions of postings and exhausts tail latency. How must the solution change?
- Which part of your solution protects the invariant in this related case: A short query matches billions of postings and exhausts tail latency.

</details>

---

### Every keystroke becomes a server request

⏱ 12 min

**Scenario:** Search autocomplete: Fast typists create six requests for one intended search.

**Constraints:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales. Preserve: Prefix lookup; Fresh trending suggestions.

<details><summary>Hints</summary>

1. Client cancellation does not undo server work.
2. Prefix results are highly reusable.

</details>

<details><summary>Reference answer</summary>

**Answer:** Debounce input, enforce a minimum useful prefix, cancel obsolete requests, and cache prefix-locale results at edge. Return bounded top-K candidates and suppress stale responses on the client using request sequence IDs.

**Trade-offs:** Precomputed suggestions are fast but stale between builds; online ranking improves freshness while increasing latency and cache fragmentation.

**Staff-level view:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

**Follow-ups:**

- Now combine this scenario with: A moderation update removes a candidate but old prefix snapshots still include it. How must the solution change?
- Which part of your solution protects the invariant in this related case: Suggestions cached only by prefix include a previous user’s private searches.

</details>

---

### A blocked suggestion persists in cached lists

⏱ 10 min

**Scenario:** Search autocomplete: A moderation update removes a candidate but old prefix snapshots still include it.

**Constraints:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales. Preserve: Prefix lookup; Fresh trending suggestions.

<details><summary>Hints</summary>

1. Fast snapshots need a fast removal path.
2. Candidate IDs allow independent filtering.

</details>

<details><summary>Reference answer</summary>

**Answer:** Maintain a small versioned deny filter in serving, invalidate affected hot prefixes, and rebuild snapshots asynchronously. Check policy after retrieval so old index generations cannot resurrect the candidate.

**Trade-offs:** Precomputed suggestions are fast but stale between builds; online ranking improves freshness while increasing latency and cache fragmentation.

**Staff-level view:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

**Follow-ups:**

- Now combine this scenario with: Fast typists create six requests for one intended search. How must the solution change?
- Which part of your solution protects the invariant in this related case: Suggestions cached only by prefix include a previous user’s private searches.

</details>

---

### Personalized cache leaks another user’s history

⏱ 15 min

**Scenario:** Search autocomplete: Suggestions cached only by prefix include a previous user’s private searches.

**Constraints:** 500M keystroke queries/day; p99 server budget 30 ms; 50 locales. Preserve: Prefix lookup; Fresh trending suggestions.

<details><summary>Hints</summary>

1. Cache keys must represent personalization scope.
2. Public and private candidates can merge late.

</details>

<details><summary>Reference answer</summary>

**Answer:** Cache only public prefix candidates globally, retrieve private history under user-scoped authorization, and merge at the final response boundary. Never place personalized responses in a shared cache without correct private cache controls.

**Trade-offs:** Precomputed suggestions are fast but stale between builds; online ranking improves freshness while increasing latency and cache fragmentation.

**Staff-level view:** Specify minimum cohort thresholds, retention, locale normalization, and emergency removals before using raw search logs as a suggestion source.

**Follow-ups:**

- Now combine this scenario with: Fast typists create six requests for one intended search. How must the solution change?
- Which part of your solution protects the invariant in this related case: Fast typists create six requests for one intended search.

</details>

---

### High-cardinality facets blow query memory

⏱ 12 min

**Scenario:** Product search: A request aggregates millions of distinct seller and attribute combinations.

**Constraints:** 50M products; 80M queries/day; 10M product updates/day. Preserve: Text relevance and filters; Facets and sorting.

<details><summary>Hints</summary>

1. Facets can cost more than retrieval.
2. Bound user-controlled aggregation complexity.

</details>

<details><summary>Reference answer</summary>

**Answer:** Allowlist facets, cap bucket counts and candidate sets, precompute common category facets, and impose query time/memory budgets. Avoid unrestricted scripts and expose approximate counts when exact aggregation is too costly.

**Trade-offs:** Denormalizing catalog fields accelerates queries but adds update fanout and staleness; current checkout validation protects the purchase invariant.

**Staff-level view:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

**Follow-ups:**

- Now combine this scenario with: Index lag leaves an old sale price visible after the catalog changed. How must the solution change?
- Which part of your solution protects the invariant in this related case: A slow full reindex writes version 40 after live ingestion wrote version 44.

</details>

---

### Search advertises yesterday’s price

⏱ 10 min

**Scenario:** Product search: Index lag leaves an old sale price visible after the catalog changed.

**Constraints:** 50M products; 80M queries/day; 10M product updates/day. Preserve: Text relevance and filters; Facets and sorting.

<details><summary>Hints</summary>

1. Search is a projection, not the price authority.
2. Checkout must validate the authoritative version.

</details>

<details><summary>Reference answer</summary>

**Answer:** Display enrichment from current price for top results where feasible, show bounded freshness, and always reprice at checkout with user-visible confirmation. Monitor source-to-index lag and replay missing catalog events.

**Trade-offs:** Denormalizing catalog fields accelerates queries but adds update fanout and staleness; current checkout validation protects the purchase invariant.

**Staff-level view:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

**Follow-ups:**

- Now combine this scenario with: A request aggregates millions of distinct seller and attribute combinations. How must the solution change?
- Which part of your solution protects the invariant in this related case: A slow full reindex writes version 40 after live ingestion wrote version 44.

</details>

---

### Backfill overwrites a newer product update

⏱ 15 min

**Scenario:** Product search: A slow full reindex writes version 40 after live ingestion wrote version 44.

**Constraints:** 50M products; 80M queries/day; 10M product updates/day. Preserve: Text relevance and filters; Facets and sorting.

<details><summary>Hints</summary>

1. Bulk rebuild and live updates need version ordering.
2. Index generation swap is safer than in-place mixing.

</details>

<details><summary>Reference answer</summary>

**Answer:** Build a new generation from a consistent snapshot, replay changes from its checkpoint, and reject older per-product versions. Switch the alias only after lag and validation checks pass, retaining rollback capability.

**Trade-offs:** Denormalizing catalog fields accelerates queries but adds update fanout and staleness; current checkout validation protects the purchase invariant.

**Staff-level view:** Define a freshness budget per field: descriptions can lag longer than price or availability. Measure zero-result rate and business correctness separately from query latency.

**Follow-ups:**

- Now combine this scenario with: A request aggregates millions of distinct seller and attribute combinations. How must the solution change?
- Which part of your solution protects the invariant in this related case: A request aggregates millions of distinct seller and attribute combinations.

</details>

---

### One incident causes a logging storm

⏱ 12 min

**Scenario:** Log ingestion and search: A failing service emits the same stack trace millions of times per second.

**Constraints:** 100TB raw logs/day; 30-day hot retention; 10K tenants. Preserve: Durable ingestion; Tenant isolation.

<details><summary>Hints</summary>

1. Observability traffic can worsen an outage.
2. Admission must preserve useful evidence.

</details>

<details><summary>Reference answer</summary>

**Answer:** Apply tenant/service byte quotas, sample repeated low-priority events, and preserve error fingerprints with counts. Buffer within strict limits and publish dropped-byte metrics so reduced visibility is explicit.

**Trade-offs:** Indexing every field improves arbitrary search but multiplies storage and ingest cost; selective indexing plus compressed scans is often more economical.

**Staff-level view:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

**Follow-ups:**

- Now combine this scenario with: User-supplied JSON creates a new field name for every request ID. How must the solution change?
- Which part of your solution protects the invariant in this related case: An offline agent uploads events older than the hot-retention boundary.

</details>

---

### Mapping explosion takes down indexing

⏱ 10 min

**Scenario:** Log ingestion and search: User-supplied JSON creates a new field name for every request ID.

**Constraints:** 100TB raw logs/day; 30-day hot retention; 10K tenants. Preserve: Durable ingestion; Tenant isolation.

<details><summary>Hints</summary>

1. Unbounded schema cardinality is a resource attack.
2. Not every field needs a dedicated index.

</details>

<details><summary>Reference answer</summary>

**Answer:** Limit indexed field counts, normalize dynamic keys, and store arbitrary attributes in a controlled representation. Quarantine offending streams and reprocess from the durable buffer after fixing the schema.

**Trade-offs:** Indexing every field improves arbitrary search but multiplies storage and ingest cost; selective indexing plus compressed scans is often more economical.

**Staff-level view:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

**Follow-ups:**

- Now combine this scenario with: A failing service emits the same stack trace millions of times per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: An offline agent uploads events older than the hot-retention boundary.

</details>

---

### Late logs fall into an already-expired partition

⏱ 15 min

**Scenario:** Log ingestion and search: An offline agent uploads events older than the hot-retention boundary.

**Constraints:** 100TB raw logs/day; 30-day hot retention; 10K tenants. Preserve: Durable ingestion; Tenant isolation.

<details><summary>Hints</summary>

1. Retention can use event time or ingest time.
2. The choice affects cost and discoverability.

</details>

<details><summary>Reference answer</summary>

**Answer:** Define an explicit lateness policy: archive old events, reject them with feedback, or route to a late-arrival partition. Keep event and ingest timestamps and ensure retention enforcement cannot be bypassed by future-dated events.

**Trade-offs:** Indexing every field improves arbitrary search but multiplies storage and ingest cost; selective indexing plus compressed scans is often more economical.

**Staff-level view:** Budget bytes, cardinality, query CPU, and result size per tenant, and provide operationally useful degradation during the exact incidents when logs surge.

**Follow-ups:**

- Now combine this scenario with: A failing service emits the same stack trace millions of times per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: A failing service emits the same stack trace millions of times per second.

</details>

---

### Restrictive filters return too few neighbors

⏱ 12 min

**Scenario:** Vector retrieval service: Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category.

**Constraints:** 500M chunks; 768-dimensional embeddings; 50M searches/day. Preserve: Embedding versioning; Hybrid retrieval.

<details><summary>Hints</summary>

1. Post-filtering can destroy recall.
2. Measure filtered recall, not only raw ANN recall.

</details>

<details><summary>Reference answer</summary>

**Answer:** Partition by tenant where practical, use supported prefiltering or filter-aware search, and adapt candidate overfetch within a budget. Benchmark filtered recall against exact search samples and fall back for small candidate sets.

**Trade-offs:** Approximate indexes improve speed with a recall tradeoff; aggressive post-filtering and compression can further reduce useful recall.

**Staff-level view:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

**Follow-ups:**

- Now combine this scenario with: New documents use a different model while queries still use the old model. How must the solution change?
- Which part of your solution protects the invariant in this related case: The vector index contains an old ACL and passes private text to a downstream model.

</details>

---

### Embedding upgrade mixes incompatible spaces

⏱ 10 min

**Scenario:** Vector retrieval service: New documents use a different model while queries still use the old model.

**Constraints:** 500M chunks; 768-dimensional embeddings; 50M searches/day. Preserve: Embedding versioning; Hybrid retrieval.

<details><summary>Hints</summary>

1. Equal vector dimensions do not mean compatible meaning.
2. Model version belongs in the index identity.

</details>

<details><summary>Reference answer</summary>

**Answer:** Build a separate model-versioned index, generate matching query embeddings, backfill and validate retrieval quality, then switch routing atomically. Dual-serve during evaluation and retain a rollback generation.

**Trade-offs:** Approximate indexes improve speed with a recall tradeoff; aggressive post-filtering and compression can further reduce useful recall.

**Staff-level view:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

**Follow-ups:**

- Now combine this scenario with: Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category. How must the solution change?
- Which part of your solution protects the invariant in this related case: The vector index contains an old ACL and passes private text to a downstream model.

</details>

---

### Private document appears in retrieved context

⏱ 15 min

**Scenario:** Vector retrieval service: The vector index contains an old ACL and passes private text to a downstream model.

**Constraints:** 500M chunks; 768-dimensional embeddings; 50M searches/day. Preserve: Embedding versioning; Hybrid retrieval.

<details><summary>Hints</summary>

1. Retrieval candidates are not authorization.
2. Filter before exposing text to another component.

</details>

<details><summary>Reference answer</summary>

**Answer:** Recheck authoritative permissions before fetching or returning chunk text, include ACL versioning, and propagate deletions to all index generations. Avoid logging unauthorized candidate contents and test cross-tenant adversarial queries.

**Trade-offs:** Approximate indexes improve speed with a recall tradeoff; aggressive post-filtering and compression can further reduce useful recall.

**Staff-level view:** Evaluate semantic quality, filtered recall, freshness, deletion propagation, and per-query cost independently; never treat similarity score as proof of factual correctness.

**Follow-ups:**

- Now combine this scenario with: Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category. How must the solution change?
- Which part of your solution protects the invariant in this related case: Approximate search retrieves nearby vectors that are mostly outside the requested tenant or category.

</details>

---

## Infrastructure

### Failover overwhelms the healthy region

⏱ 12 min

**Scenario:** Global load balancer: A failed region’s entire traffic is redirected to a region with only 30% spare capacity.

**Constraints:** 10M requests/s global peak; three active regions; long-lived connections included. Preserve: Health-aware routing; Connection draining.

<details><summary>Hints</summary>

1. Health is not the same as capacity.
2. Evacuation must respect admission limits.

</details>

<details><summary>Reference answer</summary>

**Answer:** Route only admitted load according to available capacity, shed lower-priority traffic, and ramp failover by cohorts. Reserve disaster capacity or define a degraded-mode contract; retries must stay within the same global budget.

**Trade-offs:** Fast health reaction shortens outages but risks flapping; slower convergence is steadier but routes more requests to failing capacity.

**Staff-level view:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

**Follow-ups:**

- Now combine this scenario with: A shared dependency makes a deep health endpoint fail on all otherwise useful servers. How must the solution change?
- Which part of your solution protects the invariant in this related case: A deployment removes a backend while WebSocket sessions still rely on it.

</details>

---

### Health checks remove every backend

⏱ 10 min

**Scenario:** Global load balancer: A shared dependency makes a deep health endpoint fail on all otherwise useful servers.

**Constraints:** 10M requests/s global peak; three active regions; long-lived connections included. Preserve: Health-aware routing; Connection draining.

<details><summary>Hints</summary>

1. A health signal can create correlated ejection.
2. Liveness and readiness are different.

</details>

<details><summary>Reference answer</summary>

**Answer:** Separate process health from dependency readiness, require evidence across samples, and limit simultaneous ejection. Maintain a carefully defined last-resort pool for safe operations and expose dependency-specific degradation.

**Trade-offs:** Fast health reaction shortens outages but risks flapping; slower convergence is steadier but routes more requests to failing capacity.

**Staff-level view:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

**Follow-ups:**

- Now combine this scenario with: A failed region’s entire traffic is redirected to a region with only 30% spare capacity. How must the solution change?
- Which part of your solution protects the invariant in this related case: A deployment removes a backend while WebSocket sessions still rely on it.

</details>

---

### Draining drops long-lived connections

⏱ 15 min

**Scenario:** Global load balancer: A deployment removes a backend while WebSocket sessions still rely on it.

**Constraints:** 10M requests/s global peak; three active regions; long-lived connections included. Preserve: Health-aware routing; Connection draining.

<details><summary>Hints</summary>

1. Stopping new traffic does not migrate existing state.
2. Clients need resumable sessions.

</details>

<details><summary>Reference answer</summary>

**Answer:** Stop new assignments, allow a bounded drain window, notify clients to reconnect with jitter, and recover sessions through durable cursors. Force-close only after the documented deadline and measure lost in-flight requests.

**Trade-offs:** Fast health reaction shortens outages but risks flapping; slower convergence is steadier but routes more requests to failing capacity.

**Staff-level view:** Define overload policy before failover, test correlated health-check failures, and include long-lived connection migration in regional evacuation drills.

**Follow-ups:**

- Now combine this scenario with: A failed region’s entire traffic is redirected to a region with only 30% spare capacity. How must the solution change?
- Which part of your solution protects the invariant in this related case: A failed region’s entire traffic is redirected to a region with only 30% spare capacity.

</details>

---

### A tiny zone receives a massive query flood

⏱ 12 min

**Scenario:** Authoritative DNS: One popular or attacked name attracts millions of queries per second.

**Constraints:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs. Preserve: Fast lookup; Zone versioning.

<details><summary>Hints</summary>

1. UDP packet rate and amplification matter.
2. Authoritative serving should avoid a database hit.

</details>

<details><summary>Reference answer</summary>

**Answer:** Serve validated zone snapshots from memory at distributed edges, apply response-rate limiting where appropriate, and provision packet-processing capacity. Separate zone control-plane quotas from public query traffic.

**Trade-offs:** Low TTL speeds planned changes but increases query load and dependency on authoritative availability; high TTL improves resilience while slowing propagation.

**Staff-level view:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

**Follow-ups:**

- Now combine this scenario with: A configuration update accidentally removes an essential record. How must the solution change?
- Which part of your solution protects the invariant in this related case: A record is corrected but recursive resolvers still return the previous answer.

</details>

---

### Bad zone version reaches every region

⏱ 10 min

**Scenario:** Authoritative DNS: A configuration update accidentally removes an essential record.

**Constraints:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs. Preserve: Fast lookup; Zone versioning.

<details><summary>Hints</summary>

1. A syntactically valid change can be operationally dangerous.
2. Versioned rollout makes rollback concrete.

</details>

<details><summary>Reference answer</summary>

**Answer:** Validate zone invariants, canary the publication, compare critical answers, and roll back the active version quickly. Preserve the prior signed generation and audit who changed which records.

**Trade-offs:** Low TTL speeds planned changes but increases query load and dependency on authoritative availability; high TTL improves resilience while slowing propagation.

**Staff-level view:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

**Follow-ups:**

- Now combine this scenario with: One popular or attacked name attracts millions of queries per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: A record is corrected but recursive resolvers still return the previous answer.

</details>

---

### DNS rollback appears ineffective

⏱ 15 min

**Scenario:** Authoritative DNS: A record is corrected but recursive resolvers still return the previous answer.

**Constraints:** 5M queries/s; 10M zones; records range from 30-second to 1-day TTLs. Preserve: Fast lookup; Zone versioning.

<details><summary>Hints</summary>

1. Authoritative state does not erase resolver caches.
2. TTL applies after responses leave your servers.

</details>

<details><summary>Reference answer</summary>

**Answer:** Explain and measure the remaining cache lifetime; plan migrations by lowering TTL in advance. Avoid promising instant global revocation and account for negative caching as well as positive records.

**Trade-offs:** Low TTL speeds planned changes but increases query load and dependency on authoritative availability; high TTL improves resilience while slowing propagation.

**Staff-level view:** Test DNSSEC signing rollover where used, negative-cache behavior, delegation errors, and recovery from a control-plane compromise without disrupting stable serving.

**Follow-ups:**

- Now combine this scenario with: One popular or attacked name attracts millions of queries per second. How must the solution change?
- Which part of your solution protects the invariant in this related case: One popular or attacked name attracts millions of queries per second.

</details>

---

### Request IDs create millions of new series

⏱ 12 min

**Scenario:** Metrics and alerting: An instrumentation change adds request_id as a metric label.

**Constraints:** 50M active series; scrape every 15 seconds; 30-day raw retention. Preserve: High-volume ingestion; Range queries.

<details><summary>Hints</summary>

1. Series cardinality drives index and memory cost.
2. Labels should represent bounded dimensions.

</details>

<details><summary>Reference answer</summary>

**Answer:** Reject or drop high-cardinality labels at ingestion, enforce per-tenant series budgets, and use logs or traces for request IDs. Roll back the instrumentation and track created-series rate as well as sample rate.

**Trade-offs:** Fine scrape intervals improve resolution but multiply samples; downsampling reduces cost while losing short-lived spikes and some aggregation fidelity.

**Staff-level view:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

**Follow-ups:**

- Now combine this scenario with: Ingestion works but no rules are evaluated for twenty minutes. How must the solution change?
- Which part of your solution protects the invariant in this related case: A network partition makes an error-rate alert falsely report healthy traffic.

</details>

---

### Alert evaluator fails silently

⏱ 10 min

**Scenario:** Metrics and alerting: Ingestion works but no rules are evaluated for twenty minutes.

**Constraints:** 50M active series; scrape every 15 seconds; 30-day raw retention. Preserve: High-volume ingestion; Range queries.

<details><summary>Hints</summary>

1. No alerts is not proof of health.
2. The monitoring pipeline needs independent monitoring.

</details>

<details><summary>Reference answer</summary>

**Answer:** Alert on evaluation heartbeat and staleness from an independent path, fail over rule ownership with deduplication, and expose gaps. Evaluate missed windows according to a defined catch-up policy rather than fabricating continuous health.

**Trade-offs:** Fine scrape intervals improve resolution but multiply samples; downsampling reduces cost while losing short-lived spikes and some aggregation fidelity.

**Staff-level view:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

**Follow-ups:**

- Now combine this scenario with: An instrumentation change adds request_id as a metric label. How must the solution change?
- Which part of your solution protects the invariant in this related case: A network partition makes an error-rate alert falsely report healthy traffic.

</details>

---

### Missing samples are treated as zero

⏱ 15 min

**Scenario:** Metrics and alerting: A network partition makes an error-rate alert falsely report healthy traffic.

**Constraints:** 50M active series; scrape every 15 seconds; 30-day raw retention. Preserve: High-volume ingestion; Range queries.

<details><summary>Hints</summary>

1. Absent data and numeric zero differ.
2. Alert semantics must define missingness.

</details>

<details><summary>Reference answer</summary>

**Answer:** Preserve no-data as a distinct state, require sufficient samples for ratios, and configure explicit absent-series alerts for critical services. Report ingestion lag and evaluation data age alongside the computed value.

**Trade-offs:** Fine scrape intervals improve resolution but multiply samples; downsampling reduces cost while losing short-lived spikes and some aggregation fidelity.

**Staff-level view:** Design alert ownership, deduplication, inhibition, and evaluation continuity during regional failover; monitor the monitor through an independent failure domain.

**Follow-ups:**

- Now combine this scenario with: An instrumentation change adds request_id as a metric label. How must the solution change?
- Which part of your solution protects the invariant in this related case: An instrumentation change adds request_id as a metric label.

</details>

---

### Remote evaluation adds a dependency to every request

⏱ 12 min

**Scenario:** Feature flag platform: Ten billion evaluations per day call the flag server synchronously.

**Constraints:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare. Preserve: Stable targeting; Fast local evaluation.

<details><summary>Hints</summary>

1. Rules are small and changes are infrequent.
2. The data plane can evaluate locally.

</details>

<details><summary>Reference answer</summary>

**Answer:** Distribute signed or integrity-checked versioned snapshots to SDKs and evaluate locally. Keep the last-known-good version and bounded refresh; send sampled exposure events asynchronously with privacy-aware context.

**Trade-offs:** Local evaluation improves availability and latency but makes immediate revocation depend on SDK refresh and connectivity; critical safety controls may need a separate path.

**Staff-level view:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

**Follow-ups:**

- Now combine this scenario with: A malformed rule crashes an old SDK version. How must the solution change?
- Which part of your solution protects the invariant in this related case: A 10% rollout uses a random number on every evaluation.

</details>

---

### Bad rollout config reaches all clients

⏱ 10 min

**Scenario:** Feature flag platform: A malformed rule crashes an old SDK version.

**Constraints:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare. Preserve: Stable targeting; Fast local evaluation.

<details><summary>Hints</summary>

1. The producer and consumer evolve independently.
2. Schema compatibility and safe defaults matter.

</details>

<details><summary>Reference answer</summary>

**Answer:** Validate configurations against supported SDK schemas, canary by client version, and retain a fallback snapshot. Reject unsupported operators or safely use the flag default; provide a minimal emergency-disable path.

**Trade-offs:** Local evaluation improves availability and latency but makes immediate revocation depend on SDK refresh and connectivity; critical safety controls may need a separate path.

**Staff-level view:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

**Follow-ups:**

- Now combine this scenario with: Ten billion evaluations per day call the flag server synchronously. How must the solution change?
- Which part of your solution protects the invariant in this related case: A 10% rollout uses a random number on every evaluation.

</details>

---

### Users jump between rollout cohorts

⏱ 15 min

**Scenario:** Feature flag platform: A 10% rollout uses a random number on every evaluation.

**Constraints:** 100K applications; 10B evaluations/day; configuration updates are comparatively rare. Preserve: Stable targeting; Fast local evaluation.

<details><summary>Hints</summary>

1. Assignment should be deterministic.
2. A stable salt allows independent experiments.

</details>

<details><summary>Reference answer</summary>

**Answer:** Hash a stable subject key with flag or experiment salt into a fixed bucket space. Increasing rollout percentage expands a stable cohort; version deliberate rebucketing and keep subject identity consistent across devices when required.

**Trade-offs:** Local evaluation improves availability and latency but makes immediate revocation depend on SDK refresh and connectivity; critical safety controls may need a separate path.

**Staff-level view:** Define stale-config limits, defaults during startup, audit and approval ownership, and the maximum supported offline period for each class of flag.

**Follow-ups:**

- Now combine this scenario with: Ten billion evaluations per day call the flag server synchronously. How must the solution change?
- Which part of your solution protects the invariant in this related case: Ten billion evaluations per day call the flag server synchronously.

</details>

---

### Deployment generates a watch storm

⏱ 12 min

**Scenario:** Service discovery and configuration: Restarting 100K instances sends endpoint updates to every client.

**Constraints:** 1M service instances; 50K watch clients; endpoint churn during deployments. Preserve: Lease-based registration; Watch and resync.

<details><summary>Hints</summary>

1. Fanout magnifies small control-plane changes.
2. Clients only need relevant service subsets.

</details>

<details><summary>Reference answer</summary>

**Answer:** Subscribe by service or namespace, batch updates, and distribute through regional watch relays. Rate-limit registrations and reconnects; clients retain a valid cached view while catching up from a revision.

**Trade-offs:** Strict configuration consistency can reduce control-plane availability during partitions; cached data-plane operation limits the user impact.

**Staff-level view:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

**Follow-ups:**

- Now combine this scenario with: A disconnected client requests events older than retained history. How must the solution change?
- Which part of your solution protects the invariant in this related case: A partitioned former leader continues sending updates after a new leader is elected.

</details>

---

### Watcher resumes from a compacted revision

⏱ 10 min

**Scenario:** Service discovery and configuration: A disconnected client requests events older than retained history.

**Constraints:** 1M service instances; 50K watch clients; endpoint churn during deployments. Preserve: Lease-based registration; Watch and resync.

<details><summary>Hints</summary>

1. A delta stream has a bounded history.
2. Recovery needs a snapshot plus revision.

</details>

<details><summary>Reference answer</summary>

**Answer:** Return an explicit resync-required result, fetch a consistent snapshot with its revision, then watch subsequent changes. Do not silently continue from current time and leave missing endpoint updates undetected.

**Trade-offs:** Strict configuration consistency can reduce control-plane availability during partitions; cached data-plane operation limits the user impact.

**Staff-level view:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

**Follow-ups:**

- Now combine this scenario with: Restarting 100K instances sends endpoint updates to every client. How must the solution change?
- Which part of your solution protects the invariant in this related case: A partitioned former leader continues sending updates after a new leader is elected.

</details>

---

### Two leaders push conflicting configuration

⏱ 15 min

**Scenario:** Service discovery and configuration: A partitioned former leader continues sending updates after a new leader is elected.

**Constraints:** 1M service instances; 50K watch clients; endpoint churn during deployments. Preserve: Lease-based registration; Watch and resync.

<details><summary>Hints</summary>

1. A leadership belief is not write authority.
2. Consumers can verify monotonic committed revisions.

</details>

<details><summary>Reference answer</summary>

**Answer:** Accept configuration from committed consensus revisions and reject stale epochs. Keep writes unavailable without quorum if required for safety, while data-plane clients continue with last-known-good configuration within a documented staleness bound.

**Trade-offs:** Strict configuration consistency can reduce control-plane availability during partitions; cached data-plane operation limits the user impact.

**Staff-level view:** Test quorum loss, watch compaction, mass reconnects, and certificate rotation together; control-plane recovery must not trigger a larger data-plane outage.

**Follow-ups:**

- Now combine this scenario with: Restarting 100K instances sends endpoint updates to every client. How must the solution change?
- Which part of your solution protects the invariant in this related case: Restarting 100K instances sends endpoint updates to every client.

</details>

---

## Advanced

### A hot tenant dominates one partition

⏱ 12 min

**Scenario:** Multi-region key-value store: One tenant key prefix generates 40% of global write traffic.

**Constraints:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT. Preserve: Regional reads; Defined write ownership.

<details><summary>Hints</summary>

1. Range partitioning follows key distribution.
2. A single logical key cannot be split without semantics.

</details>

<details><summary>Reference answer</summary>

**Answer:** Split hot ranges and distribute independent tenant entities with hashed suffixes. For one hot value, redesign as sharded components or a single-owner log; adding replicas does not automatically parallelize conflicting writes.

**Trade-offs:** Low-latency independent regional writes require conflict handling or weaker invariants; synchronous global coordination trades latency and partition availability for stronger ordering.

**Staff-level view:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

**Follow-ups:**

- Now combine this scenario with: The primary region fails before its replication backlog reaches the standby. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two regions update the same shopping preference while disconnected.

</details>

---

### Async failover loses acknowledged recent writes

⏱ 10 min

**Scenario:** Multi-region key-value store: The primary region fails before its replication backlog reaches the standby.

**Constraints:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT. Preserve: Regional reads; Defined write ownership.

<details><summary>Hints</summary>

1. RPO follows the acknowledgement boundary.
2. Promoting a lagging replica changes durability.

</details>

<details><summary>Reference answer</summary>

**Answer:** Quantify the missing log range, fence the old primary, and promote only under the accepted RPO policy. If zero acknowledged-write loss is required, use synchronous cross-region quorum before acknowledgement and accept its latency.

**Trade-offs:** Low-latency independent regional writes require conflict handling or weaker invariants; synchronous global coordination trades latency and partition availability for stronger ordering.

**Staff-level view:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

**Follow-ups:**

- Now combine this scenario with: One tenant key prefix generates 40% of global write traffic. How must the solution change?
- Which part of your solution protects the invariant in this related case: Two regions update the same shopping preference while disconnected.

</details>

---

### Concurrent regions overwrite each other

⏱ 15 min

**Scenario:** Multi-region key-value store: Two regions update the same shopping preference while disconnected.

**Constraints:** 20TB hot data; 200K writes/s; three regions with 80-180 ms inter-region RTT. Preserve: Regional reads; Defined write ownership.

<details><summary>Hints</summary>

1. Last-write-wins depends on clocks and semantics.
2. Some values merge; others need coordination.

</details>

<details><summary>Reference answer</summary>

**Answer:** Choose per-field merge semantics or explicit conflict records for mergeable data; use single ownership or quorum transactions for nonmergeable invariants. Preserve causal/version metadata and tombstones long enough for repair.

**Trade-offs:** Low-latency independent regional writes require conflict handling or weaker invariants; synchronous global coordination trades latency and partition availability for stronger ordering.

**Staff-level view:** State the permitted anomaly for each operation, prove fencing during failover, and test restoration of an old region without resurrecting deleted data.

**Follow-ups:**

- Now combine this scenario with: One tenant key prefix generates 40% of global write traffic. How must the solution change?
- Which part of your solution protects the invariant in this related case: One tenant key prefix generates 40% of global write traffic.

</details>

---

### Long histories make every decision expensive

⏱ 12 min

**Scenario:** Durable workflow engine: A workflow has accumulated a million events and replay takes minutes.

**Constraints:** 100M active workflows; 20M activity completions/day; workflows may last a year. Preserve: Crash-resilient progress; Versioned workflow logic.

<details><summary>Hints</summary>

1. Durable history is not free to replay.
2. Continuation can bound one execution history.

</details>

<details><summary>Reference answer</summary>

**Answer:** Checkpoint supported state or continue as a new linked execution with a bounded history. Move high-frequency telemetry outside workflow history and keep only business decisions needed for deterministic recovery.

**Trade-offs:** Durable orchestration centralizes recoverability but introduces history storage and versioning complexity; choreography distributes ownership while making global progress harder to inspect.

**Staff-level view:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

**Follow-ups:**

- Now combine this scenario with: New workflow code takes a different branch while replaying an old history. How must the solution change?
- Which part of your solution protects the invariant in this related case: A booking workflow needs to refund payment after hotel reservation fails, but refunds are unavailable.

</details>

---

### Deployment makes replay nondeterministic

⏱ 10 min

**Scenario:** Durable workflow engine: New workflow code takes a different branch while replaying an old history.

**Constraints:** 100M active workflows; 20M activity completions/day; workflows may last a year. Preserve: Crash-resilient progress; Versioned workflow logic.

<details><summary>Hints</summary>

1. Past decisions must reproduce under replay.
2. External reads belong in recorded activities.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use workflow version markers or compatible code paths for existing histories; record external activity results and avoid uncontrolled wall-clock or random calls in deterministic execution. Canary replay against real sanitized histories.

**Trade-offs:** Durable orchestration centralizes recoverability but introduces history storage and versioning complexity; choreography distributes ownership while making global progress harder to inspect.

**Staff-level view:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

**Follow-ups:**

- Now combine this scenario with: A workflow has accumulated a million events and replay takes minutes. How must the solution change?
- Which part of your solution protects the invariant in this related case: A booking workflow needs to refund payment after hotel reservation fails, but refunds are unavailable.

</details>

---

### Compensation itself fails

⏱ 15 min

**Scenario:** Durable workflow engine: A booking workflow needs to refund payment after hotel reservation fails, but refunds are unavailable.

**Constraints:** 100M active workflows; 20M activity completions/day; workflows may last a year. Preserve: Crash-resilient progress; Versioned workflow logic.

<details><summary>Hints</summary>

1. Compensation is another fallible workflow step.
2. Rollback is not instantaneous across services.

</details>

<details><summary>Reference answer</summary>

**Answer:** Persist compensation state, retry idempotently with bounded backoff, and escalate to an operator queue after a policy threshold. Expose compensating and unresolved states rather than marking the original workflow simply rolled back.

**Trade-offs:** Durable orchestration centralizes recoverability but introduces history storage and versioning complexity; choreography distributes ownership while making global progress harder to inspect.

**Staff-level view:** Design version retirement, operator intervention, workflow cancellation, and data retention for processes that outlive several application releases.

**Follow-ups:**

- Now combine this scenario with: A workflow has accumulated a million events and replay takes minutes. How must the solution change?
- Which part of your solution protects the invariant in this related case: A workflow has accumulated a million events and replay takes minutes.

</details>

---

### A global campaign hot-spots budget checks

⏱ 12 min

**Scenario:** Ad auction and budget pacing: One campaign participates in millions of auctions across regions.

**Constraints:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns. Preserve: Candidate eligibility; Auction scoring.

<details><summary>Hints</summary>

1. An exact global counter adds network coordination.
2. Pacing can allocate bounded spend rights.

</details>

<details><summary>Reference answer</summary>

**Answer:** Lease spend quotas to regions or serving shards and debit locally within those limits. Refill conservatively, account for in-flight impressions, and compute the worst-case budget excess before accepting a pacing design.

**Trade-offs:** Tight pacing reduces overspend but can underspend during partitions; larger local allocations improve serving availability while increasing in-flight exposure.

**Staff-level view:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

**Follow-ups:**

- Now combine this scenario with: The system allocates new ads before earlier impressions are reported as billable. How must the solution change?
- Which part of your solution protects the invariant in this related case: A client retries impression tracking after a lost response.

</details>

---

### Delayed impressions overshoot the budget

⏱ 10 min

**Scenario:** Ad auction and budget pacing: The system allocates new ads before earlier impressions are reported as billable.

**Constraints:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns. Preserve: Candidate eligibility; Auction scoring.

<details><summary>Hints</summary>

1. Decision time and billing time differ.
2. Outstanding exposure is a liability.

</details>

<details><summary>Reference answer</summary>

**Answer:** Reserve estimated spend at auction or delivery according to billing rules, reconcile actual billable events, and release unused reservations with a bounded timeout. Include outstanding reservations in pacing and monitor reporting lag.

**Trade-offs:** Tight pacing reduces overspend but can underspend during partitions; larger local allocations improve serving availability while increasing in-flight exposure.

**Staff-level view:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

**Follow-ups:**

- Now combine this scenario with: One campaign participates in millions of auctions across regions. How must the solution change?
- Which part of your solution protects the invariant in this related case: A client retries impression tracking after a lost response.

</details>

---

### Duplicate impression events bill twice

⏱ 15 min

**Scenario:** Ad auction and budget pacing: A client retries impression tracking after a lost response.

**Constraints:** 5B auctions/day; p99 decision budget 100 ms; 1M campaigns. Preserve: Candidate eligibility; Auction scoring.

<details><summary>Hints</summary>

1. Billing requires durable event uniqueness.
2. An auction may have several event types.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use a canonical billable-event identity tied to auction, placement, and event type, validate eligibility, and deduplicate within the ledger transaction. Keep raw events for fraud review without counting every raw delivery as billable.

**Trade-offs:** Tight pacing reduces overspend but can underspend during partitions; larger local allocations improve serving availability while increasing in-flight exposure.

**Staff-level view:** Separate auction latency, relevance, billable-event accuracy, advertiser budget safety, and fraud detection. Explain which accounting decisions are provisional.

**Follow-ups:**

- Now combine this scenario with: One campaign participates in millions of auctions across regions. How must the solution change?
- Which part of your solution protects the invariant in this related case: One campaign participates in millions of auctions across regions.

</details>

---

### One slow feature source consumes the latency budget

⏱ 12 min

**Scenario:** Recommendation serving: The ranker waits 120ms for a rarely useful feature on every request.

**Constraints:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget. Preserve: Low-latency recommendations; Feature freshness.

<details><summary>Hints</summary>

1. Tail latency sums across dependencies.
2. Feature value should justify its cost.

</details>

<details><summary>Reference answer</summary>

**Answer:** Set per-feature deadlines, precompute high-value slow features, and use explicit missing-value defaults trained into the model. Compare quality lift against latency and cost; fall back to a simpler ranker when dependencies degrade.

**Trade-offs:** Fresh online features improve responsiveness but add dependency and consistency costs; precomputed candidates are resilient but may miss immediate intent.

**Staff-level view:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

**Follow-ups:**

- Now combine this scenario with: Offline evaluation improves while production recommendations regress. How must the solution change?
- Which part of your solution protects the invariant in this related case: A request times out but its assigned treatment is logged as if the user saw the recommendations.

</details>

---

### Training and serving use different feature definitions

⏱ 10 min

**Scenario:** Recommendation serving: Offline evaluation improves while production recommendations regress.

**Constraints:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget. Preserve: Low-latency recommendations; Feature freshness.

<details><summary>Hints</summary>

1. Feature names can match while semantics differ.
2. Version transformations and event-time cutoffs.

</details>

<details><summary>Reference answer</summary>

**Answer:** Share or validate transformation definitions, version feature schemas with the model, and compare sampled online vectors to point-in-time-correct offline recomputation. Roll back the model-feature bundle rather than only model weights.

**Trade-offs:** Fresh online features improve responsiveness but add dependency and consistency costs; precomputed candidates are resilient but may miss immediate intent.

**Staff-level view:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

**Follow-ups:**

- Now combine this scenario with: The ranker waits 120ms for a rarely useful feature on every request. How must the solution change?
- Which part of your solution protects the invariant in this related case: A request times out but its assigned treatment is logged as if the user saw the recommendations.

</details>

---

### Experiment exposure is counted before delivery

⏱ 15 min

**Scenario:** Recommendation serving: A request times out but its assigned treatment is logged as if the user saw the recommendations.

**Constraints:** 100M daily users; 30 recommendations requests/day; 100ms p95 serving budget. Preserve: Low-latency recommendations; Feature freshness.

<details><summary>Hints</summary>

1. Assignment and exposure are different events.
2. Attribution needs stable request identity.

</details>

<details><summary>Reference answer</summary>

**Answer:** Record assignment separately from actual rendered exposure, deduplicate both by stable request/item identity, and define the experiment analysis unit. Preserve model and feature versions for reproducibility.

**Trade-offs:** Fresh online features improve responsiveness but add dependency and consistency costs; precomputed candidates are resilient but may miss immediate intent.

**Staff-level view:** Evaluate quality, latency, feature freshness, policy compliance, experiment validity, and cost together; maintain a useful fallback that can run when the feature platform is unavailable.

**Follow-ups:**

- Now combine this scenario with: The ranker waits 120ms for a rarely useful feature on every request. How must the solution change?
- Which part of your solution protects the invariant in this related case: The ranker waits 120ms for a rarely useful feature on every request.

</details>

---

### One shared IP becomes a hot aggregation key

⏱ 12 min

**Scenario:** Streaming fraud detection: A mobile carrier NAT causes millions of users to share one IP feature bucket.

**Constraints:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late. Preserve: Low-latency decisioning; Late-event handling.

<details><summary>Hints</summary>

1. A high-volume key is not necessarily suspicious.
2. Feature definitions affect both scale and bias.

</details>

<details><summary>Reference answer</summary>

**Answer:** Use hierarchical or salted partial aggregates and merge bounded windows, cap per-key work, and include network context. Avoid treating shared-IP volume as a standalone fraud verdict; compare feature utility on legitimate cohorts.

**Trade-offs:** Waiting for more events can improve signal completeness but violates real-time latency; the design needs an explicit policy for uncertain or stale features.

**Staff-level view:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

**Follow-ups:**

- Now combine this scenario with: A deployment resets rolling velocity counts and risky transactions are approved. How must the solution change?
- Which part of your solution protects the invariant in this related case: An old device event arrives after the original transaction was approved.

</details>

---

### Processor restarts and forgets window state

⏱ 10 min

**Scenario:** Streaming fraud detection: A deployment resets rolling velocity counts and risky transactions are approved.

**Constraints:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late. Preserve: Low-latency decisioning; Late-event handling.

<details><summary>Hints</summary>

1. Offsets and state must recover together.
2. A fresh process does not imply fresh features.

</details>

<details><summary>Reference answer</summary>

**Answer:** Restore a consistent state checkpoint with its source offsets, replay the gap, and mark features stale until caught up. Use a defined degraded decision policy and monitor event-time lag rather than CPU alone.

**Trade-offs:** Waiting for more events can improve signal completeness but violates real-time latency; the design needs an explicit policy for uncertain or stale features.

**Staff-level view:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

**Follow-ups:**

- Now combine this scenario with: A mobile carrier NAT causes millions of users to share one IP feature bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: An old device event arrives after the original transaction was approved.

</details>

---

### Late events change a decision after funds moved

⏱ 15 min

**Scenario:** Streaming fraud detection: An old device event arrives after the original transaction was approved.

**Constraints:** 100M payment attempts/day; 50ms feature budget; events arrive up to 10 minutes late. Preserve: Low-latency decisioning; Late-event handling.

<details><summary>Hints</summary>

1. Streaming corrections cannot undo external history.
2. Decision records should remain immutable.

</details>

<details><summary>Reference answer</summary>

**Answer:** Preserve the original decision with its feature snapshot, compute a linked updated risk assessment, and trigger an explicit review or downstream action when policy allows. Use watermarks and a lateness policy for aggregates without rewriting past approvals.

**Trade-offs:** Waiting for more events can improve signal completeness but violates real-time latency; the design needs an explicit policy for uncertain or stale features.

**Staff-level view:** Separate detection quality from financial loss and customer friction; document model versions, fallback decisions, appeals, and the operational response to a stale feature pipeline.

**Follow-ups:**

- Now combine this scenario with: A mobile carrier NAT causes millions of users to share one IP feature bucket. How must the solution change?
- Which part of your solution protects the invariant in this related case: A mobile carrier NAT causes millions of users to share one IP feature bucket.

</details>

---
